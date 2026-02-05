---
title: 毕设日志（二）Luckfox RKMPI示例的资源管理分析
published: 2026-01-31
description: '分析 luckfox_pico_rtsp_retinaface_osd 例程中对媒体资源的处理，并阐述毕设思路'
image: ''
tags: [rockit, webrtc, rknn, rtsp, C++]
category: 'linux'
draft: false 
lang: ''
---
分析 [`luckfox_pico_rtsp_retinaface_osd`](https://github.com/LuckfoxTECH/luckfox_pico_rkmpi_example/blob/kernel-5.10.160/example/luckfox_pico_rtsp_retinaface_osd/src/main.cc)例程

::github{repo="LuckfoxTECH/luckfox_pico_rkmpi_example"}

------
## 一、VENC编码和RTSP推流线程

### 1. 编码和推流线程分析

在 RKMPI 架构中，VI（视频输入）到 VENC（视频编码）的数据流转通常是**硬件自动完成**的，实现零拷贝。我们的任务是从 VENC 获取编码后的 H.264/H.265 数据包并交付给推流协议（如 RTSP）。

```cpp
static void *GetMediaBuffer(void *arg) {
	(void)arg;
	printf("========%s========\n", __func__);
	void *pData = RK_NULL;
	int s32Ret;

	VENC_STREAM_S stFrame;
	stFrame.pstPack = (VENC_PACK_S *)malloc(sizeof(VENC_PACK_S));

	while (1) {
    
    // 获取编码后的H.264包
		s32Ret = RK_MPI_VENC_GetStream(0, &stFrame, -1);
		if (s32Ret == RK_SUCCESS) {
			if (g_rtsplive && g_rtsp_session) {
				pData = RK_MPI_MB_Handle2VirAddr(stFrame.pstPack->pMbBlk);
				rtsp_tx_video(g_rtsp_session, (uint8_t *)pData, stFrame.pstPack->u32Len,
				              stFrame.pstPack->u64PTS);
				rtsp_do_event(g_rtsplive);
			}

      // 资源释放，释放编码后的包
			s32Ret = RK_MPI_VENC_ReleaseStream(0, &stFrame);
			if (s32Ret != RK_SUCCESS) {
				RK_LOGE("RK_MPI_VENC_ReleaseStream fail %x", s32Ret);
			}
		}
    
    // 暂停10ms
		usleep(10 * 1000);
	}
	printf("\n======exit %s=======\n", __func__);
  
  // 线程结束时释放stFrame
	free(stFrame.pstPack);
	return NULL;
}
```

### 2. `VENC_STREAM_S`编码数据帧分析

其中，主要的帧格式是`VENC_STREAM_S`，可以看到它封装了 VENC 支持的编码格式

```cpp
/* Defines the features of an stream */
typedef struct rkVENC_STREAM_S {
    VENC_PACK_S ATTRIBUTE* pstPack;            /* R; stream pack attribute*/
    RK_U32      ATTRIBUTE u32PackCount;        /* R; the pack number of one frame stream*/
    RK_U32      u32Seq;                        /* R; the list number of stream*/

    union {
        VENC_STREAM_INFO_H264_S   stH264Info;                        /* R; the stream info of h264*/
        VENC_STREAM_INFO_JPEG_S   stJpegInfo;                        /* R; the stream info of jpeg*/
        VENC_STREAM_INFO_H265_S   stH265Info;                        /* R; the stream info of h265*/
        VENC_STREAM_INFO_PRORES_S stProresInfo;                      /* R; the stream info of prores*/
    };

    union {
        VENC_STREAM_ADVANCE_INFO_H264_S   stAdvanceH264Info;         /* R; the stream info of h264*/
        VENC_STREAM_ADVANCE_INFO_JPEG_S   stAdvanceJpegInfo;         /* R; the stream info of jpeg*/
        VENC_STREAM_ADVANCE_INFO_H265_S   stAdvanceH265Info;         /* R; the stream info of h265*/
        VENC_STREAM_ADVANCE_INFO_PRORES_S stAdvanceProresInfo;       /* R; the stream info of prores*/
    };
} VENC_STREAM_S;
```

## 二、NPU推理线程

### 1. NPU推理线程分析

利用`RK_MPI_VI_GetChnFrame`从 VI 通道获取原始数据帧，再利用 **opencv-mobile** 进行格式转换和缩放，最后用 RGN，在 VENC 编码前将 OSD 合入视频帧

- **格式转换**：将 **NV12** 转换为 **BGR**

- **缩放**：将图像适配模型输入的 **640×640** 尺寸

```cpp
static void *RetinaProcessBuffer(void *arg) {
  
  // ... 变量初始化 ...
  
  // 这是rkmpi中最核心最原始的数据帧，用它来承接从VI通道中获取的原始图像
	VIDEO_FRAME_INFO_S stViFrame;

	while(1)
	{
    // 手动抓取，主动向VI的通道1（专门用于推理的）请求一帧数据
		s32Ret = RK_MPI_VI_GetChnFrame(0, 1, &stViFrame, -1);
    
    // 之后利用opencv-mobile，对数据帧进行格式转换和缩放
		if(s32Ret == RK_SUCCESS)
		{
      // 获取虚拟地址
			void *vi_data = RK_MPI_MB_Handle2VirAddr(stViFrame.stVFrame.pMbBlk);
			if(vi_data != RK_NULL)
			{
        // 使用cv::Mat对原始数据帧进行封装，并进行格式转换和缩放处理
				cv::Mat yuv420sp(disp_height + disp_height / 2, disp_width, CV_8UC1, vi_data);
				cv::Mat bgr(disp_height, disp_width, CV_8UC3);			
				cv::Mat model_bgr(model_height, model_width, CV_8UC3);			

				cv::cvtColor(yuv420sp, bgr, cv::COLOR_YUV420sp2BGR);

				cv::resize(bgr, model_bgr, cv::Size(model_width ,model_height), 0, 0, cv::INTER_LINEAR);	
        
        // memcpy到RKNN输入
				memcpy(rknn_app_ctx.input_mems[0]->virt_addr, model_bgr.data, model_width * model_height * 3);
        
        // 阻塞，该函数计算结束才会返回
        // od_results为推理结果
				inference_retinaface_model(&rknn_app_ctx, &od_results);

        // 利用rgn，将osd覆盖到venc的输入数据上，实现标注
				// ... RGN 处理逻辑 ...
				}		
			}
      // 释放帧资源
			s32Ret = RK_MPI_VI_ReleaseChnFrame(0, 1, &stViFrame);
		}
    // 暂停500ms
		usleep(500000);		
		// ... 清理 RGN 资源 ...
	}			
	return NULL;
}
```
### 2. `VIDEO_FRAME_INFO_S`原始数据帧分析

该线程操作的数据帧：`VIDEO_FRAME_INFO_S`

```cpp
typedef struct rkVIDEO_FRAME_S {
    MB_BLK              pMbBlk; // 底层内存块句柄，该结构体本质上是对它的一层封装
    RK_U32              u32Width;
    RK_U32              u32Height;
    RK_U32              u32VirWidth;
    RK_U32              u32VirHeight;
    VIDEO_FIELD_E       enField;
    PIXEL_FORMAT_E      enPixelFormat;
    VIDEO_FORMAT_E      enVideoFormat;
    COMPRESS_MODE_E     enCompressMode;
    DYNAMIC_RANGE_E     enDynamicRange;
    COLOR_GAMUT_E       enColorGamut;

    RK_VOID            *pVirAddr[RK_MAX_COLOR_COMPONENT]; //虚拟地址

    RK_U32              u32TimeRef;
    RK_U64              u64PTS;

    RK_U64              u64PrivateData;
    RK_U32              u32FrameFlag;     /* FRAME_FLAG_E, can be OR operation. */
} VIDEO_FRAME_S;

typedef struct rkVIDEO_FRAME_INFO_S {
    VIDEO_FRAME_S stVFrame;
} VIDEO_FRAME_INFO_S;
```

## 三、毕设软件设计思路

也就是说，对于我毕业设计这个需求，我需要处理两种格式，一是 VENC 编码后的 H.264 视频流，二是从 VI 通道直接获取的原始数据帧。rkmpi 实际上已经做了一定程度的封装了，我应该在它封装的基础上进行管理，而不是拆开它们再自己封装。

### 1.  使用`std::share_ptr`和自定义删除器来封装数据帧

原始帧和编码流虽然接口不同，但是它们的使用都符合这么一个逻辑：每一次循环开始时，先获取，进行处理，再释放。对于线性场景，比如 VENC 编码后，交由 RTSP 进行推流，那么可以在循环中执行上述逻辑。

但是对于我毕设的需求来说，编码流会可能会同时用于 RTSP、WebRTC 推流和本地保存，那么上述逻辑就会变得复杂且难以处理。此时适合使用`std::shared_ptr`结合**自定义删除器**来进行处理，不同场景共享引用计数，只有当所有引用都结束后，再结束它的生命周期，进行析构（释放资源）。

以下是对编码流的处理，原始帧类似

```cpp
/**
 * @brief 编码后的视频流智能指针 (VENC 输出)
 * 
 * 用于推流路径，H.264/H.265 编码后的数据包
 * 释放时自动调用 RK_MPI_VENC_ReleaseStream
 */
using EncodedStreamPtr = std::shared_ptr<VENC_STREAM_S>;

/**
 * @brief 从 VENC 通道获取编码流并包装为智能指针
 * 
 * @param chn_id VENC 通道 ID
 * @param timeout_ms 超时时间（毫秒），-1 表示阻塞等待
 * @return EncodedStreamPtr 成功返回流指针，失败返回 nullptr
 * 
 * @note 返回的智能指针在所有引用释放后会自动调用 RK_MPI_VENC_ReleaseStream
 * @note 内部会分配 VENC_PACK_S，也会在释放时一并清理
 */
inline EncodedStreamPtr acquire_encoded_stream(RK_S32 chn_id, RK_S32 timeout_ms = -1) {
    auto stream = new VENC_STREAM_S();
    stream->pstPack = new VENC_PACK_S();
    
    RK_S32 ret = RK_MPI_VENC_GetStream(chn_id, stream, timeout_ms);
    if (ret != RK_SUCCESS) {
        delete stream->pstPack;
        delete stream;
        return nullptr;
    }
    
    // 创建带自定义删除器的 shared_ptr
    return EncodedStreamPtr(stream, [chn_id](VENC_STREAM_S* p) {
        if (p) {
            RK_MPI_VENC_ReleaseStream(chn_id, p);
            delete p->pstPack;
            delete p;
        }
    });
}
```

**零拷贝传递**：不同场景实际上拿到的是同一个内存块地址，不会有内存拷贝。

**多路共享**：即使 RTSP 线程因为网络抖动卡住了，WebRTC 处理完后引用计数减 1，等到 RTSP 也处理完，引用计数归零，MPI 资源自动释放。

缺点就是：VENC的缓冲池是有限的，如果 RTSP 和 WebRTC 任何一方长时间持有指针不释放，会导致 VENC 模块因为拿不到空闲 Buffer 而阻塞或丢帧。后续应该引入**丢帧机制**或**超时强制释放**。如果某个消费者（如本地保存）处理太慢，应该主动`reset()`掉该路指针，宁可丢帧也不要阻塞采集前端。

### 2. 针对 AI 推理的原始帧处理 (VI → RKNN)

对于推理线程使用的原始数据帧，它的路径和所有权比较明确，毕竟同时只会有一个 AI 推理的线程存在。

