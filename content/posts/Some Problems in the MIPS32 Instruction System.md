---
title: "MIPS32指令系统的一些常见问题" 
date: 2023-07-25T15:43:50+08:00
draft: false
---

根据MIPS32 I型指令格式:

OP码(6位)+ Rs(5位)+ Rt(5位)+ 立即数(16位)

BEQ指令的OP码是0x04(6位二进制为000100)

Rs寄存器编号18的二进制为10010(5位二进制)

Rt寄存器编号17的二进制为10001(5位二进制)

立即数为100时的二进制结果0000000001100100

正确的二进制表示应该是:

000100 10010 10001 0000000001100100


分组: 0001 0010 0101 0001 0000 0000 0110 0100

转换为十六进制: 0x1 0x2 0x5 0x1 0x0 0x0 0x6 0x4

合并结果: 0x12510064

指令加载到主存的单元地址是 0xA04040A0若该指令执行时转移条件成立，那么程序转移的目的地址是多少?

解:

指令加载地址为0xA04040A0

立即数为100

在MIPS32中,立即数表示的是要转移的字节数

每个字节为8bit,所以要转移的字节数是100 * 4 = 400 (十六进制0x190)

所以转移目的地址 = 指令地址 + 要转移的字节数

指令地址:0xA04040A0 = 2688565408 (十进制)

要转移的指令数:100

每条指令4个字节,所以要转移的字节数:100 * 4 = 400 (十六进制0x190)

2688565408 (十进制) + 400 (十进制) = 2688565808 (十进制) = 0xA0404230 (十六进制)

计算j 1000的机器指令:

j指令的OP码为0x2

地址为1000(十进制) = 0x3E8(十六进制)

正确的计算过程应该是: OP码(0x2) + 地址(0x3E8) = 0x2 + 0x3E8 = 0x3EA

已知:

指令0x3EA加载到地址0x40C04000

该指令为j指令,当执行时会无条件跳转

要求:算出指令执行后跳转的目的地址

解:

j指令的地址字段表示跳转的目的地址

指令0x3EA中的地址字段是0x3E8

所以跳转的目的地址就是地址字段的值0x3E8

因为MIPS中,地址字段代表的是字节偏移量

需要将字节偏移量转换为绝对地址

0x3E8表示的字节偏移量是1000个字节(因为每个字节为8bit,0x3E8=1000)

每条指令4个字节,所以1000字节表示1000/4=250条指令

以0x40C04000为基地址,加上偏移量250条指令,每条指令4字节,所以偏移量是250 * 4 = 0x3E8

绝对地址 = 基地址 + 偏移量 = 0x40C04000 + 0x3E8 = 0x40C03E88