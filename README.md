


---


　　大家好，我是痞子衡，是正经搞技术的痞子。今天痞子衡给大家分享的是**i.MXRT1170上PXP对CM7 TCM进行随机地址短小数据写入操作限制**。


　　在 MCU 里能够对片内外映射的存储器进行读写操作的主设备(Master)除了常见的 Core 以及 DMA 外，其实还有一些面向高速数据传输（比如 USB、uSDHC、ENET 接口等）或其他特定功能（比如 GPU、LCD、Crypto 等）的外设，但就用户数据搬移处理而言，一般我们只借助 Core 和 DMA。


　　在 i.MXRT 四位数上，还有一个叫 PXP 的外设，这本是一个面向像素数据处理的模块，但是它也能够完成一般数据搬移处理任务。当我们借助这个 PXP 来做数据搬移时，发现它在对 CM7 TCM 写入时有一些使用限制。今天我们就来聊聊这个话题：


### 一、PXP功能简介


　　先来看一下 PXP 模块功能框图，既然是面向图像数据处理，那常见的图像缩放、色彩空间转换、图像旋转功能支持必不可少（即下图蓝框里的三个独立引擎被整合在 PXP 里），这些操作实际上都涉及到 FrameBuffer 像素数据处理(读改写) 。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT1170_sparse_write_PXP_arch.PNG)


　　再进一步细读 PXP 特性，我们发现除了像素处理之外，它还是个标准的 2D DMA（这里 2D 的意思是为搬移二维图像数据而设计的），这就是我们所要的数据搬移特性。在用 PXP 做数据搬移操作时，当源 FrameBuffer 和目的 FrameBuffer 大小相同，且搬移目标尺寸就是 FrameBuffer 长度时，其就蜕变成了大家所熟悉的普通 DMA。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT1170_sparse_write_PXP_feature.PNG)


### 二、一个RT1170的Errata


　　在我们实测 PXP 数据搬移功能时，我们先来看一个 RT1160/1170 上独有的 Errata，也正是因为这个 Errata 让痞子衡关注到了 PXP 的 2D DMA 功能。


　　这个 Errata 提及到 RT1160/1170 里若干个具有存储器读写能力的主设备在对 CM7 TCM 进行 Sparse write（随机地址短小数据写入操作）时可能会导致数据出错，PXP 就是其一，解决方案就是 CM7 TCM 不要作为目的 FrameBuffer。



> * Note：列出来的有限制的主设备大多是 RT1170 里新增的外设（CAAM, ENET\_1G, ENET\_QOS, GC355, LCDIFv2），除了 PXP 是 RT10xx 上也存在的，但是 RT10xx PXP 并没有这个限制。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT1170_sparse_write_errata.PNG)


### 三、在PXP下实测数据搬移


　　要实测 PXP 数据搬移功能可以直接借助 \\SDK\_2\_16\_000\_MIMXRT1170\-EVKB\\boards\\evkbmimxrt1170\\driver\_examples\\pxp\\copy\_pic\\cm7 例程，其主要函数 APP\_CopyPicture() 摘录如下，代码清晰明了。我们要做不同的测试，只需要将 s\_inputBuf、s\_outputBuf 分别链接在不同存储器空间里，并且设置不同的拷贝块大小与坐标位置即可。



> * Note：仅需调整 COPY\_WIDTH、DEST\_OFFSET\_X 值来测试对一维数据搬移影响（数据长度、起始地址对齐因素）



```
#include "fsl_pxp.h"
// 源/目标 Buffer 长宽设置（为测试方便，可设置成一样）
#define BUF_WIDTH   64
#define BUF_HEIGHT  64
// 拷贝块长宽及在目标 Buffer 坐标设置（从源 Buffer 坐标固定为 [0,0]）
#define COPY_WIDTH        8
#define COPY_HEIGHT       8
#define DEST_OFFSET_X     1
#define DEST_OFFSET_Y     1

uint16_t s_inputBuf[BUF_HEIGHT][BUF_WIDTH];
uint16_t s_outputBuf[BUF_HEIGHT][BUF_WIDTH];

static void APP_CopyPicture(void)
{
    pxp_pic_copy_config_t pxpCopyConfig;
    // 设置拷贝参数（将s_inputBuf里坐标[0,0]开始的大小为8x8的数据拷贝到s_outputBuf里[1,1]位置处）
    // 源 Buffer 地址与拷贝块坐标设置
    pxpCopyConfig.srcPicBaseAddr  = (uint32_t)s_inputBuf;
    pxpCopyConfig.srcPitchBytes   = sizeof(uint16_t) * BUF_WIDTH;
    pxpCopyConfig.srcOffsetX      = 0;
    pxpCopyConfig.srcOffsetY      = 0;
    // 目的 Buffer 地址与拷贝块坐标设置
    pxpCopyConfig.destPicBaseAddr = (uint32_t)s_outputBuf;
    pxpCopyConfig.destPitchBytes  = sizeof(uint16_t) * BUF_WIDTH;
    pxpCopyConfig.destOffsetX     = DEST_OFFSET_X;
    pxpCopyConfig.destOffsetY     = DEST_OFFSET_Y;
    // 拷贝块大小设置（像素点格式为 RGB565 即 2bytes）
    pxpCopyConfig.width           = COPY_WIDTH;
    pxpCopyConfig.height          = COPY_HEIGHT;
    pxpCopyConfig.pixelFormat     = kPXP_AsPixelFormatRGB565;
    // 启动拷贝（将拷贝块数据从源 Buffer 搬移到目的 Buffer）
    PXP_StartPictureCopy(PXP, &pxpCopyConfig);
    while (!(kPXP_CompleteFlag & PXP_GetStatusFlags(PXP)));
    PXP_ClearStatusFlags(PXP, kPXP_CompleteFlag);
}

```

　　经测试当 s\_outputBuf 放在 OCRAM 或者外部 RAM 空间里时，搬移结果完全如预期。而当 s\_outputBuf 放在 CM7 ITCM 或者 DTCM 时，则会出现异常结果，在 ITCM/DTCM 异常表现是一致的。


　　设置不同的 COPY\_WIDTH、DEST\_OFFSET\_X 值组合带来的异常结果不尽相同，这里仅放出一个 COPY\_WIDTH \= 1、DEST\_OFFSET\_X \= 3 的情况供参考，可以看到除了目标地址数据之外，前后还会有一些额外数据被写入，这样的数据搬移操作显然不可靠了。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT1170_sparse_write_PXP_res.PNG)


　　当然并不是 s\_outputBuf 放在 CM7 TCM 就一定能引起异常，只要拷贝的一维数据长度是 16bytes 整数倍，且目的起始地址以 8 对齐时，此时并无出错情况发生。不满足这个条件的写入我们即称之为有风险的 Sparse write（随机地址短小数据写入）。


　　至此，i.MXRT1170上PXP对CM7 TCM进行随机地址短小数据写入操作限制痞子衡便介绍完毕了，掌声在哪里\~\~\~


### 欢迎订阅


文章会同时发布到我的 [博客园主页](https://github.com)、[CSDN主页](https://github.com)、[知乎主页](https://github.com):[西部世界官网](https://www.xbsj9.com)、[微信公众号](https://github.com) 平台上。


微信搜索"**痞子衡嵌入式**"或者扫描下面二维码，就可以在手机上第一时间看了哦。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/wechat/pzhMcu_qrcode_258x258.jpg)


