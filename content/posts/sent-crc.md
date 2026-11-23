---
title: "SENT (SAE J2716) CRC 校验完整说明"
date: 2026-11-20T17:37:00+08:00
draft: false
tags: ["SENT", "CRC", "SAE J2716"]
categories: ["协议笔记"]
---

# SENT (SAE J2716) CRC 校验完整说明

| 快通道每帧CRC4 /简短型串行消息CRC4(16帧) /增强型串行消息CRC6(18帧)<br>依据：SAE J2716 APR2016 标准原文 5.2.4 / 5.4 节。本文档所有表格和算例均已用官方MATLAB参考<br>现独立核算过。 | 实 |  |
| --- | --- | --- |
| 0. 三种CRC的关系一览<br>SENT里一共有三处要做CRC/校验和，互相独立、不要混用：<br>位<br>#保护对象帧数多项式种子<br>数 |  |  |
| 4        1帧<br>1     快通道每一帧的数据nibble        x⁴+x³+x²+1        0101 (5)<br>bit        内 |  |  |
| 简短型串行消息(Short Serial        4        x⁴+x³+x²+1（与①同一套算<br>2        16帧        0101 (5)<br>Message)        bit        法） |  |  |
| 增强型串行消息(Enhanced Serial        6        010101<br>3        18帧      x⁶+x⁴+x³+1<br>Message)        bit        (0x15=21)<br>①和②用的是完全相同的CRC4算法和表，只是喂进去的数据不同（①是快通道的数据nibble，②是慢通道<br>来的ID+Data）。③是完全独立的一套6-bit算法。 | 拼 | 出 |

## 1. 快通道（Fast Channel）每帧 CRC4 校验

### 1.1 帧结构回顾

一帧SENT快通道消息 = 校准脉冲(56 tick) + 1个状态/通信nibble + 1~6个数据nibble + 1个CRC nibble + (可选)暂停脉冲。

重要：状态/通信nibble（就是承载慢通道bit2/bit3的那个nibble）不参与这个CRC计算，CRC只保护数据nibble本身。

### 1.2 CRC参数

多项式：x⁴+x³+x²+1（4-bit CRC，标准里直接写多项式，不是查表法里那种"去掉最高位"的简化写法）

种子：0101（十进制5）

推荐实现：在数据nibble序列末尾补一个值为0的nibble再算一次（这一步是2010年后"推荐实现"新加的，2008年及更早的"legacy实现"没有补零这一步，兼容性差，标准里明确说不推荐）

### 1.3 查表法（16元素表，推荐实现）

官方给出的16元素异或表：

<!-- footer
1 / 7
-->

<!-- figure: 2 shape(s), page 1 -->

<!-- figure: 1 shape(s), page 1 -->

<!-- figure: 1 shape(s), page 1 -->

<!-- page 2 -->

2026-07-24

SENT_CRC校验完整说明.md

crc4_table = [0, 13, 7, 10, 14, 3, 9, 4, 1, 12, 6, 11, 15, 2, 8, 5]索引： 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15

算法步骤：

CheckSum ← 0 XOR crc4_table[CheckSum] // 第③步：末尾补一个"0值nibble"再算一轮// 最终 CheckSum 就是CRC nibble要发送的值

标准原文Appendix B给的例子：6个数据nibble = [7, 4, 8, 7, 4, 8]，官方给出的期望结果是 Check Sum = 3。逐步走一遍：

步骤 输入nibble     查表：crc4_table[当前Check Sum] 新Check Sum = 输入 XOR 表值

| 初始化 | — | — | 5 |
| --- | --- | --- | --- |
| 第1个 | 7 | table[5]=3 | 7 XOR 3 = 4 |
| 第2个 | 4 | table[4]=14 | 4 XOR 14 = 10 |
| 第3个 | 8 | table[10]=6 | 8 XOR 6 = 14 |
| 第4个 | 7 | table[14]=8 | 7 XOR 8 = 15 |
| 第5个 | 4 | table[15]=5 | 4 XOR 5 = 1 |
| 第6个 | 8 | table[1]=13 | 8 XOR 13 = 5 |

补零        0	table[5]=3	0 XOR 3 = 3 ✓

最终CRC nibble = 3，跟标准给出的期望值完全一致，算法验证通过。

### 1.5 备用：256元素表实现（等价，只是查表方式不同）

如果你更习惯"左移4位再查表"的写法（跟256色查色表类似的思路），标准也给了等价的256元素表实现：

CheckSum ← 5 for 每个数据nibble N:

tempCS ← N + CheckSum × 16 // 拼成一个8位索引(高4位CheckSum,低4位N) CheckSum ← CRC4Table_256[tempCS] // 补零同理: tempCS ← CheckSum × 16; CheckSum ← CRC4Table_256[tempCS]

<!-- footer
2 / 7
-->



// 第①步：初始化种子

CheckSum ← 5 for 每个数据nibble N (按顺序):

CheckSum ← N XOR crc4_table[CheckSum] // 第②步：每个nibble做一次"查表+异或"

1.4 官方算例，逐步演示


<!-- page 3 -->

2026-07-24

SENT_CRC校验完整说明.md

这两种实现（16元素异或表 vs 256元素移位表）算出来的结果完全一样，Lab VIEW里随便选一种，16元素表明显更省资源，建议用它。

<!-- figure: 1 shape(s), page 3 -->

## 2. 简短型串行消息（Short Serial Message，16帧）CRC4

### 2.1 帧结构

用状态nibble的 bit#3 做起始标志，bit#2 做数据位，横跨连续16帧：bit#2 (数据位) d0(MSB) d1 d2 d3 ... d15(LSB)

| 接收序号 | 1 | 2 | 3 | 4 | ... | 16 |
| --- | --- | --- | --- | --- | --- | --- |
| bit#3 (起始位) | 1 | 0 | 0 | 0 | ... | 0 |

第1帧的bit#3=1 表示"这是一条新简短消息的开始"，接下来15帧的bit#3必须连续为0（如果中途有一帧bit#3变成1，说明前面的消息被打断了，作废重新开始）。16帧必须全部无错接收（校准脉冲正常、CRC正常等）才算收到一条完整消息。

16个数据位（bit#2，MSB在前）拼起来 = 4位消息ID + 8位数据字节 + 4位CRC，正好16位。

### 2.2 CRC计算方法

标准原文明确写："The CRC checksum is derived for the Message ID and Data Byte and is the same checksum algorithm as used to calculate the SENT CRC nibble"——也就是说，直接复用第1节的CRC4算法

<!-- footer
3 / 7
-->

![page 3](/images/sent-crc/image-004.png)

<!-- figure: 1 shape(s), page 3 -->

<!-- figure: 1 shape(s), page 3 -->

<!-- page 4 -->

2026-07-24

SENT_CRC校验完整说明.md

假设 ID = 0x5，DataByte = 0x3C（二进制 0011 1100）：

步骤	输入nibble     查表：crc4_table[Check Sum] 新Check Sum

| 初始化 | — | — | 5 |
| --- | --- | --- | --- |
| nibble1=5(ID) | 5 | table[5]=3 | 5 XOR 3 = 6 |
| nibble2=3 | 3 | table[6]=9 | 3 XOR 9 = 10 |
| nibble3=12 | 12 | table[10]=6 | 12 XOR 6 = 10 |

补零	0 	table[10]=6	0 XOR 6 = 6

最终CRC nibble = 6（二进制0110）。

第三步：组装成16位并按MSB→LSB顺序发到bit#2

ID(4bit) + DataByte(8bit) + CRC(4bit) = 0101 00111100 0110

按第1帧到第16帧顺序，从最高位到最低位，逐帧填入bit#2；第1帧的bit#3设为1（起始标志），第2~16帧bit#3设为0。

<!-- figure: 1 shape(s), page 4 -->

## 3. 增强型串行消息（Enhanced Serial Message，18帧）CRC6

<!-- footer
4 / 7
-->


和16元素表，喂进去的数据变成：

数据 = [ID nibble, DataByte高nibble, DataByte低nibble] 共3个nibble，12bit

2.3 逐步算例


第一步：拆成3个nibble

nibble1 = ID = 0101 = 5 nibble2 = DataByte高4位 = 0011 = 3 nibble3 = DataByte低4位 = 1100 = 12

第二步：套用第1.3节的算法


<!-- page 5 -->

2026-07-24

SENT_CRC校验完整说明.md

### 3.1 帧结构总览

用状态nibble的 bit#2 和 bit#3 两条并行位流，横跨连续18帧：

帧1~6的 bit#3 = 同步头，固定 111111

帧7~18的 bit#3 上出现"0111 1110"这类特定图案标志一条新消息开始（细节标准里有详细描述，日常调试可以不用太关注，锁定18帧边界即可）帧1~6的 bit#2 = 6-bit CRC本身（CRC5~CRC0）

帧7~18的 bit2+bit3 = 24位消息体（真正参与CRC运算的内容），由配置位C决定具体分配方式：

C=0：8位消息ID + 12位数据

C=1：4位消息ID + 16位数据

18帧必须全部无错接收才算收到完整消息。

### 3.2 C=0（8位ID + 12位数据）的24位消息体位映射

<!-- footer
5 / 7
-->

![page 5](/images/sent-crc/image-008.png)

<!-- figure: 1 shape(s), page 5 -->

<!-- figure: 1 shape(s), page 5 -->

<!-- page 6 -->

2026-07-24

SENT_CRC校验完整说明.md

读入顺序固定为"每帧先读bit#2、再读bit#3"，从帧7读到帧18：

| m下标 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 来源 | D11 | 0 | D10 | C | D9 | ID7 | D8 | ID6 | D7 | ID5 | D6 | ID4 |
| m下标 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 |

来源        D5 0 D4 ID3 D3 ID2 D2 ID1 D1 ID0 D0 0

规律：bit#2流始终是数据D11→D0（MSB在前，12帧对应12位数据）；bit#3流是 0(分隔)→C→ID7~ID4→0(分隔)→ID3~ID0→0(分隔)。固定"0"出现在帧7、13、18的bit#3位置，是标准特意设计的校验冗余（这三位若不为0，直接判定整帧出错，能把漏检的4-bit错误比例从365种降到208种）。

### 3.3 C=1（4位ID + 16位数据）的24位消息体位映射

同样的读入规则，只是ID只占4位，腾出的4个bit#3槽位用来放数据的高4位（D15~D12）：

<!-- footer
6 / 7
-->

<!-- image: page 6 -->

<!-- image: page 6 -->

<!-- page 7 -->

2026-07-24

SENT_CRC校验完整说明.md

m下标     0 1 2 3 4 5 6 7 8 9 10 11

来源        D11 0 D10 C D9 ID3 D8 ID2 D7 ID1 D6 ID0

m下标     12 13 14 15 16 17 18 19 20 21 22 23

来源        D5 0 D4 D15 D3 D14 D2 D13 D1 D12 D0 0

规律：bit#2流仍然是D11→D0（跟C=0完全一样的12位低位数据）；bit#3流变成 0→C→ID3~ID0(4位

ID)→0→D15~D12(数据高4位)→0。

### 3.4 CRC6参数与查表算法（官方校正版64项表）

多项式：x⁶+x⁴+x³+1（记作0x59）

种子：010101（0x15，十进制21）

处理方式：把24位消息体切成4组6位数（十进制0~63），末尾再补6个0作为第5组

<!-- footer
7 / 7
-->


官方64项查表：

索引: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15数值: 0 25 50 43 61 36 15 22 35 58 17 8 30 7 44 53

索引: 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31数值: 31 6 45 52 34 59 16 9 60 37 14 23 1 24 51 42

索引: 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47数值: 62 39 12 21 3 26 49 40 29 4 47 54 32 57 18 11

索引: 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63数值: 33 56 19 10 28 5 46 55 2 27 48 41 63 38 13 20

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

<!-- figure: 1 shape(s), page 7 -->

