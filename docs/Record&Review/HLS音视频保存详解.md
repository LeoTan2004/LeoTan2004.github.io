# HLS音视频保存详解

> 参考资料：
>
> 【[音视频开发学习：HLS 协议详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/460011665)】
>
> 【[m3u8 文件格式详解 - 简书 (jianshu.com)](https://www.jianshu.com/p/e97f6555a070)】
>
> 【[了解 HLS 流媒体 - 探索 - Apple Developer](https://developer.apple.com/cn/news/?id=lve6alo6)】
>
> 【[RFC 8216 - HTTP Live Streaming (ietf.org)](https://datatracker.ietf.org/doc/html/rfc8216)】
>
> 【[RFC 2038 - RTP Payload Format for MPEG1/MPEG2 Video (ietf.org)](https://datatracker.ietf.org/doc/html/rfc2038)】
>
> 【[Opencv HSL video stream to web (funvisiontutorials.com)](https://www.funvisiontutorials.com/2020/11/opencv-hsl-video-stream-to-web.html)】
>
> 【[MPEG2-TS - 维基百科，自由的百科全书 (wikipedia.org)](https://zh.wikipedia.org/wiki/MPEG2-TS)】

## 引入

### HLS是什么

HLS 全称是 HTTP Live Streaming， 是一个由 Apple 公司实现的基于 HTTP 的媒体流传输协议。 他跟 DASH 协议的原理非常类似。通过将整条流切割成一个小的可以通过 HTTP 下载的媒体文件，然后提供一个配套的媒体列表文件，提供给客户端，让客户端顺序地拉取这些媒体文件播放, 来实现看上去是在播放一条流的效果。

### 为何选择HLS

由于传输层协议只需要标准的 HTTP 协议，HLS 可以方便的透过防火墙或者代理服务器，而且可以很方便的利用 CDN 进行分发加速，并且客户端实现起来也很方便。

HLS目前在流媒体领域有着广泛的运用，在互联网上也能够找到大量的资料好前人留下的经验和建议，这对于需要迅捷开发和维护的项目无疑是一个最佳的选择，同时使用HLS这项技术我也是考虑到兼容新的问题，目前对于多平台兼容性最好的当属HTTP协议，因此选择基于HTTP协议的流式媒体传输能够给项目带来更大的灵活性和兼容性。

## HLS协议

![img](https://pic2.zhimg.com/80/v2-561df18ad78a909fa18570bc4a6ec2ad_720w.webp)

上面是HLS协议的整体架构

首先是视屏来源（摄像机等）将获取到的实时画面写入媒体编码器，然后再由服务器通过**MPEG-2**重新编码、切片。同时生成一个索引文件，也可以称之为播放列表文件，发送给分发器，接着客户端只需要通过浏览器（HTTP协议）获取到索引文件，便可以自动对于索引文件中的播放片段进行HTTP请求，解码以及控制播放顺序。

针对于本项目，我们对于网络上给出的项目架构做出了调整。大致架构如下：

![image-20240501135802608](C:\Users\35098\AppData\Roaming\Typora\typora-user-images\image-20240501135802608.png)

我们将视屏编码工作调整到下位采样机（摄像机）端完成，而服务端只需要负责编码工作，从下位机发送的ts文件通过低延迟的KCP网络协议传输。这样不仅分担了中央服务器的压力，而且还能够在下位机保存原始数据，同时也方便管理人员直接在下位机查看视频（下位机上面也由一个自动编排索引文件的功能）

### 索引文件M3U8

m3u8 文件实质是一个播放列表（playlist），其可能是一个媒体播放列表（Media Playlist），或者是一个主列表（Master Playlist）。但无论是哪种播放列表，其内部文字使用的都是 **UTF-8** 编码。

当 m3u8 文件作为媒体播放列表（Media Playlist）时，其内部信息记录的是一系列媒体片段资源，顺序播放该片段资源，即可完整展示多媒体资源。下面是一个简单的M3U8的文件示例：

```shell
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:8
#EXT-X-MEDIA-SEQUENCE:2680

#EXTINF:7.975,
https://priv.example.com/fileSequence2680.ts
#EXTINF:7.941,
https://priv.example.com/fileSequence2681.ts
#EXTINF:7.975,
https://priv.example.com/fileSequence2682.ts
#EXT-X-ENDLIST
```

首先我们讲解一下索引文件的基本格式要求：

1. m3u8 文件必须以 **UTF-8 进行编码**，不能使用 Byte Order Mark（BOM）字节序， 不能包含 UTF-8 控制字符。
2. m3u8 文件的每一行要么是一个 URI，要么是空行，要么就是以 `#` 开头的字符串。**不能出现空白字符**，除了显示声明的元素。
3. m3u8 文件中以 `#` 开头的字符串要么是注释，要么就是标签。标签以 `#EXT` 开头，**大小写敏感**。

在这个文件中我们可以看到M3U8的基础标签如下：

1. **[\#EXTM3U](https://datatracker.ietf.org/doc/html/rfc8216#section-4.3.1.1)**: 表明该文件是一个 M3U8文件。每个 M3U8文件必须将该标签放置在第一行
2. **[\#EXT-X-VERSION](https://datatracker.ietf.org/doc/html/rfc8216#section-4.3.1.2)**：表明HLS版本号
3. **[\#EXT-X-TARGETDURATION](https://datatracker.ietf.org/doc/html/rfc8216#section-4.3.3.1)**：表明每个片段的最大时间跨度，也就是每个片段最长限制
4. **[\#EXT-X-MEDIA-SEQUENCE](https://datatracker.ietf.org/doc/html/rfc8216#section-4.3.3.2)**：表明播放的起始片段索引，默认从0开始。
5. **[\#EXTINF](https://datatracker.ietf.org/doc/html/rfc8216#section-4.3.2.1)**：表明媒体片段的时长，下一行必须跟上媒体片段的地址，否则不生效。数值可以使整数，也可以是小数。

6. **[\#EXT-X-ENDLIST](https://datatracker.ietf.org/doc/html/rfc8216#section-4.3.3.4)**：表明播放片段结束标识，存在表示点播，不存在表示直播。


### 视频片段

M3U8中定义的播放片段有特定的格式要求，在[RFC 8216 - HTTP Live Streaming (ietf.org)](https://datatracker.ietf.org/doc/html/rfc8216#section-3.1)中我们可以看到目前支持的编码格式如下：

- **MPEG-2 Transport Streams**
- **Fragmented MPEG-4**
- **Packed Audio**
- **WebVTT**

这里我们选择使用**MPEG-2 Transport Streams**，也就是我们架构图上所标记的TS文件，这种文件也是实际场景中最常用的编码格式之一。

#### MPEG-2 Transport Streams

> **MPEG2-TS 传输流**（MPEG-2 Transport Stream；又称MPEG-TS、MTS、TS）是一种标准数字封装格式,用来传输和存储视频、音频与频道、节目信息，应用于数字电视广播系统。
>
> MPEG2-TS面向的传输介质是地面和卫星等可靠性较低的传输介质，这一点与面向较可靠介质如DVD等的[MPEG PS](https://zh.wikipedia.org/w/index.php?title=MPEG_PS&action=edit&redlink=1)不同。

如果我们在Python-OpenCV中直接使用`cv2`

todo 这里我们由于时间关系，调整发展方向为识别结果回传

#### **Fragmented MPEG-4**

> cv2.VideoWriter_fourcc(**‘X’,‘V’,‘I’,‘D’**)
>
> **MPEG-4**编码类型，视频大小为平均值，MPEG4所需要的空间是MPEG1或M-JPEG的1/10，它对运动物体可以保证有良好的清晰度，间/时间/画质具有可调性。文件扩展名.avi。

# 基于KCP的传输服务

> 【[skywind3000/kcp: :zap: KCP - A Fast and Reliable ARQ Protocol (github.com)](https://github.com/skywind3000/kcp)】
>
> 【[HatBoy/Python-KCP: Python2/Python3 binding for KCP (github.com)](https://github.com/HatBoy/Python-KCP?tab=readme-ov-file)】
>
> 【[详解 KCP 协议的原理和实现 - Luyu Huang's Blog](https://luyuhuang.tech/2020/12/09/kcp.html)】
>
> 【[在网络中狂奔：KCP协议 (zhihu.com)](https://www.zhihu.com/tardis/zm/art/112442341?source_id=1003)】



## KCP介绍

**KCP**是一个快速可靠的 **ARQ** (Automatic Repeat-reQuest, 自动重传请求) 协议, 采用了不同于 TCP 的自动重传策略, 有着比 TCP 更低的网络延迟. 实际通常使用 KCP over UDP 代替 TCP, 应用在网络游戏和音视频传输中. 











