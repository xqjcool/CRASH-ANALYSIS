layout: post
title: "困扰一年多的 use-after-free crash 问题，终于找到原因并解决了！"
categories: [Linux Kernel, Debugging]
tags: [use-after-free, crash, debugging, networking]

困扰一年多的 use-after-free crash 问题，终于找到原因并解决了！

目录

TOC
{:toc}

一、问题初现

那是一年多前，客户上报了一个设备 crash 问题，当时拿到 coredump 后，先是发现 _skb_refdst 异常。

coredump 1 号

异常栈分析

异常发生在 CPU0 上，"PANIC: general protection fault: 0000 [#1] SMP PTI" 提示异常地址访问，导致 crash。

 #5 [ffff88827ec03a70] general_protection at ffffffff80c012c5
    [exception RIP: ip_route_input_rcu+507]
    RIP: ffffffff8091c63b  RSP: ffff88827ec03b20  RFLAGS: 00010246
    RAX: ffff888150ba6b0e  RBX: ffff888181b60500  RCX: 000000000000c910
    RDX: 9ac14df3f281abd3  RSI: 0000000000000006  RDI: ffff888150ba6b0e
    RBP: ffff88827ec03bc8   R8: 0000000000000000   R9: 000000000000bb01
    R10: 0000000000000028  R11: ffff88827ec03be0  R12: ffff8882329f5000
    R13: 000000002ea3d9ac  R14: 00000000c8b2150a  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018

查看反汇编

crash> dis -r ip_route_input_rcu+507   
0xffffffff8091c440 <ip_route_input_rcu>:        nopl   0x0(%rax,%rax,1) [FTRACE NOP]
0xffffffff8091c445 <ip_route_input_rcu+5>:      push   %rbp
0xffffffff8091c446 <ip_route_input_rcu+6>:      mov    %r9,%r11
0xffffffff8091c449 <ip_route_input_rcu+9>:      xor    %r9d,%r9d

二、再起波澜

几个月后，另一个客户也遇到了系统 crash，这次的现象更加怪异。

coredump 3 号 异常栈分析

问题现象：PANIC: "BUG: unable to handle kernel paging request at ffffffff00001848"

 #8 [ffff88847ec038b0] page_fault at ffffffff80c01315
    [exception RIP: nf_reinject+55]
    RIP: ffffffff808629e7  RSP: ffff88847ec03968  RFLAGS: 00010282

查看反汇编

crash> dis -r nf_reinject+55
0xffffffff808629b0 <nf_reinject>:       nopl   0x0(%rax,%rax,1) [FTRACE NOP]
0xffffffff808629b5 <nf_reinject+5>:     push   %rbp
0xffffffff808629b6 <nf_reinject+6>:     mov    %rsp,%rbp

三、问题分析与解决

经过进一步分析，发现 priv_stats 释放问题，最终定位到 refcnt 计数错误，导致 use-after-free。

最终，我们修复了 priv_stats 计数错误，并对 refcnt 逻辑进行了完整检查，彻底解决了这个问题。
