---
layout: post
title: Wireshark 常用排错过滤条件
category: 开发
tags: Wireshark Tcp fliter
keywords: Wireshark, TCP
description:
---

【原文载于 [EMC中文支持论坛](https://community.emc.com/go/chinese) 现在是 [Dell 论坛](https://www.dell.com/community/%E5%85%A5%E9%97%A8%E7%BA%A7%E5%92%8C%E4%B8%AD%E7%AB%AF/%E5%A6%82%E6%9E%9C%E7%9C%8B%E4%BA%86%E8%BF%99%E4%B8%AA%E4%BD%A0%E8%BF%98%E6%98%AF%E4%B8%8D%E4%BC%9A%E7%94%A8Wireshark-%E9%82%A3%E5%B0%B1%E6%9D%A5%E6%89%BE%E6%88%91%E5%90%A7-8%E6%9C%886%E6%97%A5%E5%AE%8C%E7%BB%93/td-p/7007033)】

对于排查网络延时/应用问题有一些过滤条件是非常有用的：

**tcp.analysis.lost_segment:**  表明已经在抓包中看到不连续的序列号。报文丢失会造成重复的 ack, 这会导致重传。

**tcp.analysis.duplicate_ack:**  显示被确认过不止一次的报文。大量的重复 ack 是 tcp 端点之间高延时的迹象。

**tcp.analysis.retransmission:**  显示抓包中的所有重传。如果重传次数不多的话还是正常的，过多重传可能有问题。这通常意味着应用性能缓慢和/或用户报文丢失。

**tcp.analysis.window_update:**  将传输过程中的 tcp window 大小图形化。如果看到窗口大小下降为零，这意味着发送方已经退出了，并等待接收方确认所有已传送数据。这可能表明接收端已经不堪重负了。

**tcp.analysis.bytes_in_flight:**  某一时间点网络上未确认字节数。未确认字节数不能超过 tcp 窗口大小（定义于最初 3 次 tcp 握手），但是为了最大化吞吐量，未确认字节数应该尽可能接近 tcp 窗口大小。如果看到连续低于 tcp 窗口大小，可能意味着报文丢失或者路径上存在其他影响吞吐量的问题。

**tcp.analysis.ack_rtt:**  衡量抓取的 tcp 报文与相应的 ack. 如果这一时间间隔比较长那可能表示某种类型的网络延时（报文丢失，拥塞，等）。

在抓包中应用以上一些过滤条件：

![wireshark_tcp_filter](/assets/image/wireshark_tcp_fliter.png)

注意：Graph 1 是 http 总体流量，显示形式为 packets/tick，时间间隔 1 秒。Graph 2 是 tcp 丢失报文片段。Graph 3 是 tcp 重复 ack, Graph 4 是 tcp 重传。

从这张图可以看到：相比于整体 http 流量，有很多数量的重传及重复 ack. 我们可以比较轻松地判断这些事件发生的时间点，以及在整体流量中所占的比例。

