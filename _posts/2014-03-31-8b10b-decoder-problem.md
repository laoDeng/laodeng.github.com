---
layout: post
title: "Serdes接口中8B/10B解码器设计的一些问题及思考"
description: ""
category: "Tech"
tags: ["Verilog", "Paper"]
---
{% include JB/setup %}

根据要求，我对SERDES接口协议再一次进行了梳理，着重阅读了其中8B/10B编解码的内容。
在阅读过程中发现，SERDES接口协议的8B/10B编解码内容主要是参照Ethernet中8B/10B的编解码标准。
但是又与以太网有显著不同，其中一些问题无法准确判断，下面提出来供大家探讨。

协议研究
========

IEEE 802.3 8B/10B编解码内容
---------------------------

简要描述下以太网协议中的编解码方式。
在SERDES接口中主要参照的标准是<span>IEEE 802.3 36.2.4</span>中的内容。
其中有一些与SERDES接口无关的内容不再赘述。

### 标准码表

以太网协议提供了完整的编码表（TABLE 36-1、TABLE
36-2），也可看作为解码表，这张表的正确性毋庸置疑，是最值得参考的资料。
他将总共256+12=268的编码情况全部列出，这将作为今后校验编解码正确性的基准。

### RD规则

以太网协议建议，将一个码组的RD分为3部分，第一是上一码组计算后的RD，第二是编码后6B部分的RD，第三是编码后4B部分的RD。
其中上一码组的RD即上一码组4B的RD。
RD运算的基本结构：last_code_group_RD -> 6B_sub-block -> 6B_RD -> 4B_sub-block -> 4B_RD(new_last_code_group_RD)。

每个sub-block的判断可用以下伪代码表示（6B和4B略有不同）：

~~~
if 000111 or 0011 or 1s>0s
6B_RD=+;RD_4B=+;
else if 111000 or 1100 or 1s<0s
6B_RD=-;RD_4B=-;
else
6B_RD=last_code_group_RD;
4B_RD=6B_RD;
endif
~~~

最后提到了RD错误，在协议附录<span>Annex 36B</span>中给出了一些接收当中的RD错误。
可以发现，RD错误是不能精确定位的，它的检测主要是通过接收机本地的RD和所接收到的RD不符所产生的错误。
但由于一系列的中性码并不会改变RD，前一码接收产生的错误可能因为一系列的中性码而直到几个码字后才能检测到。

JEDEC 204B 8B/10B解码内容
-------------------------

阅读接口协议的数据链路层内容，可以发现，在编解码器之前还有一级控制，主要是用来针对SERDES帧结构中的Lane、Frame、Multiframe校准、同步和错误的控制，而控制的依据就是编解码中获得控制字。

值得一提的是在SERDES所用的控制字只有5个，这简化了控制字的解码复杂度，分别如下：

K.29.0
:   即D，表示Multiframe的开始。

K.28.3
:   即A，表示Lane校准，一般在多帧最后出现。

K.28.4
:   即Q，表示Link设置数据的开始，在他之后跟一系列设置数据，配置Link，他也是ILAS[^1]的组成部分。

K.28.7
:   即K，表示Group同步，可以说是链接开头最重要的部分，用来保持同步，是CGS[^2]的重要控制字。

K.28.7
:   即F，表示Frame校准，一般表示一帧结束。

SERDES中8B/10B解码的主体同以太网一样，主要的区别在于错误检测及控制。
根据我的理解可以分为两部分，同步错误和解码错误。

### 同步错误

同步错误是比较严重的错误，可能的结果就是重同步，这一部分是由校准、同步和错误控制模块处理的，这里不再赘述。
但其中的Frame和Lane同步要主要参考的也是解码过程中的/A/、/F/控制字。

### 解码错误

这些是一些比较小的错误，并不需要进行重同步，但是需要检测出来做深度的控制。
这一部分错误就与解码器息息相关，解码器可能要做的不单是检测出这些错误，还要针对不同错误作出协议规定的反应。
这在<span>JEDEC 204B 7.6</span>中有详细说明。

#### Not-in-table error

这种错误意味着接收到的码字在任何RD情况下都不存在于码表中，就是一些非法的码字。
对于这些码字，协议规定解码器要重复之前收到最新的没有错误的帧。

> In response to a not-in-table error, the decoder shall repeat the
> previously received non-error frame. A not-in-table error with
> scrambling enabled can corrupt the following two octets, so in cases
> where the corrupted octets span a frame boundary both the frame
> containing the not-in-table error and the following frame shall be
> replaced by the frame preceding the not-in-table error.

#### Disparity error

这种错误就是上文提到的RD错误，协议规定解码器要根据收到的数据和RD直接解码。
由于在检测到RD错误时，可能产生错误的不是这个码字，这样规定也无可厚非。

> In case of a disparity error, the output shall be the decoded symbol
> according to the running disparity indicated by the received code
> group.

#### Unexpected control character

这种错误就是未出现在指定位置的控制字，分成两种。
一种是/A/和/F/字，即校准位没有出现在正确的位置可能导致需要重同步；
一种是其他控制字，这种情况视作Not-in-table error。
协议的思想很明确，就是在传输时所产生的非正常位置的控制字其实都是误码。

> If an unexpected control character is received, the action depends on
> its value. In case of an /A/ or /F/, octet replacement will take place
> according to the rules for frame and lane alignment monitoring. Other
> unexpected control characters are treated like not-in-table
> characters.

问题汇总
========

基于以上协议规定，我有一些问题。

1.  解码器设计的范围是否要包括同步、校准和关于同步和校准的检错？

2.  解码器包含检错功能，但在检出错误的同时解码器自身是停止工作等待命令还是继续解码？（根据协议，解码器要完成一部分的重新分配数据任务）

3.  对于Not-in-table
    error，要重复正确帧就意味着解码器自身要存储一帧以上的数据（可能还不够），这样的模块是否需要？

4.  对于Unexpected control character，非/A/、/F/字就需要视作Not-in-table
    error来操作，这样的模块是否需要？

这些问题困扰了我一些时间。
帧重复的问题可以通过后面模块来解决，但是我在解码过程中应该解出怎样的数据呢？
还是干脆停止解码交给下一级一串没有意义的码字？
但解码器自身的RD转换又该如何处理呢。

我想这些问题就是和下一级控制所需要协商的，在协议中没有发现明确规定，但在ADI的设计中采用了一个控制模块，集中处理这些问题。

[^1]: Initial Lane Alignment Sequence

[^2]: Code Group Synchronization
