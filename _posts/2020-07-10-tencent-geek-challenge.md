---
layout: post
title: 极客技术挑战赛解题思路
date: 2020-07-10 21:23:00 +0800
tags: [contest, security]
---

[原题](https://mp.weixin.qq.com/s/tZ9BmXfzGYpzrNm2Jl5Mrw) | [官方题解](https://mp.weixin.qq.com/s/IrX0NagbcmHqACcjW62yFQ)

### 0x01
```
1+1
```

### 0x02
```
(x*18-27)/3-(x+7496)=0
```
[WolframAlpha](https://www.wolframalpha.com/input/?i=%28x*18-27%29%2F3-%28x%2B7496%29%3D0)

### 0x03
```
41*x-31*x^2+74252906=0,(x^2表示x的2次方,下同),x的某个根=
```
[WolframAlpha](https://www.wolframalpha.com/input/?i=41*x-31*x%5E2%2B74252906%3D0)

### 0x04
```
(1234567^12345678901234567890)%999999997=
```
* [WolframAlpha](https://www.wolframalpha.com/input/?i=%281234567%5E12345678901234567890%29+mod+999999997)
* `pow(1234567,12345678901234567890,999999997)`

### 0x05
```
1_2_3_4_5_6_7_8_9=-497,每处_填入1个运算符+-*/,且4个运算符必须都用上,使得等式成立(答案保证唯一),表达式为?
```
可穷举(4^8=2^16种可能)

### 0x06
```
x^5-2*x^4+3*x^3-4*x^2-5*x-6=0, x(精确到小数点后14位)=
```
* [WolframAlpha](https://www.wolframalpha.com/input/?i=x%5E5-2*x%5E4%2B3*x%5E3-4*x%5E2-5*x-6%3D0)
  * 点击"more digits"
  * 由图像可知在函数在`[2.0,2.5]`之间单调递增，可二分查找零点并记录误差，小数点后14位应该正好在double精度内
  
### 0x07
```
请输入8位数字PIN码:
```
* 密钥为8位数字做10^7次md5得到，穷举需10^15次md5，查了下在GPU集群上的hash rate，估计十多天也能跑出来吧，勉强赶得上截止日期
* 买了腾讯云的GPU服务器，配完环境已经很晚了，准备第二天起来写cuda
* 大清早睡梦中突然意识到直接用8位数字去尝试解密不就可以了嘛，只需穷举10^8
  * 因为payload是gzip压缩的，所以如果解密错误解压时就会抛异常
* 穷举的密钥之间没有关联，所以能利用多核并行计算来加速
  * python中可使用multiprocessing
  
### 0x08
```
计算第8关密钥中,时间可能非常长...
```
* 程序模拟了一台假想(?)的冯诺依曼机器，带堆栈和寄存器，可执行十多种指令
* 密钥由`reg[0]`和`reg[1]`中的数字拼成，就想着假设是两个32位的寄存器，先用上一题中写的并行穷举代码来暴力一把，一边挂机一边看程序
* 16个寄存器，`reg[15]`是IP寄存器
* 大部分指令1或2字节长
  * 第一个字节高位是指令码，低位是操作的第一个寄存器r0
  * 双字节的指令，第二个字节高位是操作的第二个寄存器r1，低位是第三个寄存器r2
  * 指令`0xe`后跟8个字节的操作数
* 根据程序对指令的实现，对code中的内容进行反汇编（使用自定义的助记符号）
  * 需要一边打log跟踪一边反汇编，按顺序来的话第二条指令开始就乱了
  * 反汇编顺序：0，36-59, 3-28
  * 跟踪发现57中的操作数控制了程序的执行次数，改了一个较小的数字，让程序执行完，继续反汇编30-34和61
* 反汇编结果

```
0:ADD reg[15], 34                   ; GOTO 36

3:SKIPN reg[2]
4:ADD reg[15], 2                    ; GOTO 8
6:POP reg[15]                       ; RETURN
8:PUSH reg[2]; ADD reg[2], -1
10:MUL reg[4], reg[0], reg[3]
12:MUL reg[5], reg[1], reg[9]
14:ADD reg[4], reg[4], reg[5]
16:MOD reg[4], reg[4], reg[6]
18:PUSH reg[4]
19:MUL reg[4], reg[0], reg[7]
21:MUL reg[5], reg[1], reg[8]
23:ADD reg[4], reg[4], reg[5]
25:MOD reg[1], reg[4], reg[6]
27:POP reg[0]
28:PUSH reg[15]; ADD reg[15], -27   ; CALL 3
30:PUSH reg[15]; ADD reg[15], -29   ; CALL 3
32:POP reg[2]
33:SKIPN reg[6]
34:POP reg[15]

36:MOV reg[0], 5
38:MOV reg[1], 6
40:MOV reg[3], 3
42:MOV reg[7], 7
44:MOV reg[8], 8
46:MOV reg[9], 9
48:MOV reg[6], 99999999999999997
57:MOV reg[2], 127
59:PUSH reg[15]; ADD reg[15], -58   ; CALL 3
61:END
```
* 翻译成python

```python
r0 = 5
r1 = 6
r3 = 3
r7 = 7
r8 = 8
r9 = 9
r6 = 99999999999999997
r2 = 127

def f(r2):
  global r0, r1
  if not r2:
    return
  r0, r1 = (r0 * r3 + r1 * r9) % r6, (r0 * r7 + r1 * r8) % r6
  f(r2 - 1)
  f(r2 - 1)
  if r6:
    return

f(r2)
# print(str(r0)+str(r1))
```
* 分析
  * 可以看到r2是控制递归层数的，通过递归对r0和r1做了2^127-1次相同的操作
  * 对r6取模说明r0和r1的取值能达到2^60左右，远超32位，挂机的服务器可以退掉了
* 等价的递推形式

```python
r6 = 99999999999999997
r2 = 127

r0, r1 = 5, 6
for i in range(1, 2**r2):
  r0, r1 = (r0 * 3 + r1 * 9) % r6, (r0 * 7 + r1 * 8) % r6
# print(str(r0)+str(r1))
```
* 写成递推式

```
a[n] = (3*a[n-1] + 9*b[n-1]) % 99999999999999997
b[n] = (7*a[n-1] + 8*b[n-1]) % 99999999999999997
```
* 参照[斐波那契的矩阵表示](https://en.wikipedia.org/wiki/Fibonacci_number#Matrix_form)，写成矩阵形式

```
[[a_n]  = [[3,9]  * [[a_n-1]
 [b_n]]    [7,8]]    [b_n-1]]

[[a_n]  = ([[3,9]     * [[a_0]
 [b_n]]     [7,8]])^n    [b_0]]

[[r0]  = ([[3,9]             * [[5]
 [r1]]     [7,8]])^(2^127-1)    [6]]
```
* 参照[矩阵的模幂](https://en.wikipedia.org/wiki/Modular_exponentiation#Matrices)进行计算

### 小结

* #别人家的奶爸# 系列
* 遇到问题还是要多想不同的解决方案。选择越多越有可能最初想到的方案不是最优的
* 逆向还记得，数学不用已经忘得差不多了
* 希望以后能有个独立的奶爸/奶妈赛道🐶
