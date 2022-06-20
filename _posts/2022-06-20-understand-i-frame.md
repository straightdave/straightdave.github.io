---
layout: post
title:  "一文带你了解 I-frame"
date:   2022-06-20 10:10:10 +0800
categories: live streaming
comments: true
---

## I-frame 是什么

在 HLS 协议中，有两个跟 I-frame 相关的标签：
* EXT-X-I-FRAME-STREAM-INF
* EXT-X-I-FRAME-ONLY

比如下面的 Master Playlist 就指定了 3 个 I-frame 流：
```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=1280000
low/audio-video.m3u8
#EXT-X-I-FRAME-STREAM-INF:BANDWIDTH=86000,URI="low/iframe.m3u8" #EXT-X-STREAM-INF:BANDWIDTH=2560000
mid/audio-video.m3u8
#EXT-X-I-FRAME-STREAM-INF:BANDWIDTH=150000,URI="mid/iframe.m3u8" #EXT-X-STREAM-INF:BANDWIDTH=7680000
hi/audio-video.m3u8
#EXT-X-I-FRAME-STREAM-INF:BANDWIDTH=550000,URI="hi/iframe.m3u8" #EXT-X-STREAM-INF:BANDWIDTH=65000,CODECS="mp4a.40.5"
audio-only.m3u8
```

它和 `EXT-X-STREAM-INF` 长得很像，都有带宽和具体媒体文件的 URL。
那么问题就来了：
* 它是做什么用的，重要吗？
* 在对该视频媒体做二次处理的时候，需不需要对这样的 I-frame 流进行处理？
* 如何进行处理？

为了回答上述问题，需要我们对 I-frame 有一定的了解。

> 注：DASH 协议也有对 I-frame 的支持，本文仅用 HLS 进行讲解。

## I-frame 的定义

在 HLS 协议中，如同 `EXT-X-STREAM-INF` 一样，`EXT-X-I-FRAME-STREAM-INF` 注明了带宽要求，以及该带宽水平下的某个 Playlist 地址。这个地址返回的 Playlist 文件中，必须带有前面提到的 `EXT-X-I-FRAME-ONLY` 标签，以表明该 Playlist 是一个特殊的媒体列表文件。
这个媒体列表文件中的每一个 Segment，都是独立的帧，而不是具有特定时长的视频。Segment 头上的 `EXTINF` 标签指定的时间，其意义不是视频段的时长，而是本帧到下一帧或者媒体列表结束之前的时间。

## I-frame 的作用

I-frame 的作用就是为了实现视频的 trick play。
Trick play，或称 trick mode，是描述早期录像带那种模拟信号视频实现的快进或者快退的播放效果。比如播放录像带的时候，用户按住遥控器快进键，让播放器的小马达用物理方法使该视频以一定的倍速播放。期间，快速流转的画面仍然可以反馈给用户。最后用户在想看的画面情节处松开按键，视频开始正常播放。
进入数字化时代，视频想要快速播放或者回退，无法通过物理方式使录影带快速转动或倒转，但可以通过选取部分帧的方式，让视频长度缩短，再以正常速度播放的时候，其视觉效果就是加速。
注意，当前的视频压缩系统，比如 MPEG-2 或者 H.264，大多数视频帧会依赖其之前的某帧，无法被独立解码。因此为了打到 trick play 效果而选取的帧，一般都是这样的关键帧（I 表示其为 independent 或者 intra）。

## 实际使用场景

比如，大多数流媒体平台的播放器，用户都可以将鼠标放在播放进度条上，看到该时段的截图。或者用户可以来回拖动播放进度图标，实现画面的快进和倒退显示效果。

<img src="/assets/img/i-frame.png">

## 其它

如果该媒体使用 fragment MP4 作为媒体文件载体的话，I-frame 流可以和视频流共用一个 fragment MP4 媒体文件。即 I-frame 流的每个 segment 利用 EXT-X-BYTERANGE 标签选取该 fMP4 文件的部分字节（关键帧所在）。

另外，AWS Media 服务也实现了一种 Trickplay 的方式。其主要核心思想是：扫描原始视频，生成多个截图，然后使用图片流代替关键帧流。这种方式好处是选取的画面不必局限于关键帧，可以随时截图，甚至利用图片识别等技术进行智能截图。但这需要客户端实现对 AWS 创造的非标准 HLS 标签以及图片加载方式的支持。

## 参考资料

* HLS 协议 [Link](https://datatracker.ietf.org/doc/html/rfc8216)
* Trick play [Wiki](https://en.wikipedia.org/wiki/Trick_mode)
* AWS 对 Trickplay 的介绍 [Youtube](https://www.youtube.com/watch?v=WKwDYVCIa6I)
