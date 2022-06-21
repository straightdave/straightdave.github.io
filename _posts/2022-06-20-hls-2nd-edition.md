---
layout: post
title:  "HLS 第二版"
date:   2022-06-20 12:10:10 +0800
categories: live streaming
comments: true
---

<img src="/assets/img/rfc.jpeg">

不知不觉，HLS 第二版的草案已经进入到第 11 个版本了（详见 [draft-pantos-hls-rfc8216bis-11](https://datatracker.ietf.org/doc/draft-pantos-hls-rfc8216bis/11/)，有效期到 2022 年 11 月 12 日）。
第二版如果通过，将取代原 RFC8216 成为新的 HLS 协议文本，同时 RFC8216 将正式失效。
不过，尽管已经走到了草案的第 11 个版本，但距正式发布应该还会有一段时间。

> 从苹果 2009 年 5 月提交 HLS 初版的首个草案（编号 00），到 2017 年夏天正式通过，成为 RFC8216 为止，历时八年零3个月，其间经历了 24 个草案版本。初版甫一发布，苹果就开始了第二版的编纂工作，到今天已有五年。

## HLS 版本解惑

我们称 RFC8216 为 HLS 协议（初版，也是现行版本）；称正在编纂过程中的为 “第二版”（2nd edition）。
初版和所谓第二版，各自又是经历了数次 **草案版本**。
在不同的 **草案版本** 中，又散布着不同的 **HLS Version**（即由 `EXT-X-VERSION` 标签注明的版本号）。

有下表：

<table>
  <thead>
    <tr>
      <th></th>
      <th>草案版本</th>
      <th>协议版本</th>
      <th>主要变化</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan=7>初版</td>
      <td>00 ~ 02</td>
      <td>1</td>
      <td>基本的媒体列表</td>
    </tr>
    <tr>
      <td>03 ~ 04</td>
      <td>2</td>
      <td>EXT-X-KEY 标签携带 IV 属性</td>
    </tr>
    <tr>
      <td>05 ~ 06</td>
      <td>3</td>
      <td>EXTINF 标签中，时间长度可以用浮点数</td>
    </tr>
    <tr>
      <td>07 ~ 08</td>
      <td>4</td>
      <td>增加了对 fragment MP4，I-frame，可替代音视频流 的支持</td>
    </tr>
    <tr>
      <td>09 ~ 11</td>
      <td>5</td>
      <td>扩充了 EXT-X-KEY 的属性，新增 EXT-X-MAP 标签</td>
    </tr>
    <tr>
      <td>12 ~ 13</td>
      <td>6</td>
      <td>拓展了 EXT-X-MAP 标签的用途，移除了 `*-STREAM-INF` 标签的 PROGRAM-ID 属性；草案 13 修正了一些兼容性</td>
    </tr>
    <tr>
      <td>14 ~ 23</td>
      <td>7</td>
      <td>拓展了 EXT-X-MEDIA 的用途，修正了其一些兼容性问题，等等</td>
    </tr>
    <tr>
      <td rowspan=4>第二版</td>
      <td>00 ~ 06</td>
      <td>8</td>
      <td>增加了变量替换的功能（向 DASH 看齐 :monkey:）</td>
    </tr>
    <tr>
      <td>07 ~ 11</td>
      <td>9, 10</td>
      <td>新增标签 EXT-X-SKIP，调整了 EXT-X-SKIP 在特定场景下的行为</td>
    </tr>
  </tbody>
</table>
