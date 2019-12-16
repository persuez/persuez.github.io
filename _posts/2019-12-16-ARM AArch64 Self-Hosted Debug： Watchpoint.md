---
published: true
tags:
  - 调试
author: persuez
---

转发自：https://blog.csdn.net/moonllz/article/details/52213212

可参考资料：ARM ®Architecture Reference Manual ARMv8, for ARMv8-A architecture profile 中 D2.10.4，D2.10.1

---

## Introduction

ARMv8 中的类型：
- Byte 8 bits. 
- Halfword 16 bits. 
- Word 32 bits. 
- Doubleword 64 bits. 
- Quadword 128 bits. 

watchpoint，也可被称为 data breakpoint，是由 data address 产生的一个 debug event 
- debugger 配置 watchpoint 在一个 data address 或者一个 data address range 
    - 可配置为 read access only, write access only, or both
    - 可以 lint to a Linked Context breakpoint,即必须在特定的 context 下 match 才会触发 watchpint event
- watchpint 会产生一个 watchpoint debug event
- 指令 fetch 是不会产生 watchpint event
  
## regs
- ID_AA64DFR0_EL1.WRPs 查询实现了多少个 watchpoint
- watchpoint reg pairs 
    - DBGWCR_EL1: Watchpoint Control Register: controls of watchpoint。32 位寄存器，n = 0 - 15，see ARM v8 doc D7.3.11
    - DGBWVR.EL1: Watchpoint Value Register: data address value。64 位寄存器，n = 0 - 15，see ARM v8 doc D7.3.12

## 配置
### 基本配置
一个 watchpoint 可以配置 match 1个或者多个 bytes。当其中任意一个 byte match，就会产生一个 watchpoint event。byte 的数目可以有以下两种配置方式

- 1~8 个 bytes：通过设置 DBGWCR_EL1.BAS Byte Address Selected filed 实现，要求这些 byte 必须是连续的，并且都要在 alliened doubleword 中
- 8 bytes ~ 2GB： 通过设置 DBGWCR_EL1.MASK MASK field 实现。要求必须是 2 的幂次方，并且地址需要 range align
- debugger 只能选择其中一种配置，否则unpredictable

### 配置1~8个bytes的watchpoint
通过设置 DBGWCR_EL1.BAS Byte Address Selected Field，可以配置 1~8 bytes 的 address match。有以下两种配置方式

- Doubleword-Aligned: 使用 BAS[7:0]
- Word-Aligned: 使用 BAS[3:0]，BAS[7:4] res0

BAS 设置必须是连续的 byte。如果将 BAS 配置为非全 1，则 DBGWCR_EL1.MASK 必须配置为全0.

### 配置8bytes~2GB的watchpoint
通过设置 DBGWCR_EL1.MASK，可以将 watchpoint 配置到最多 2GB 的 match range。
MASK 定义了addr LSB bits 被 mask 的个数，MASK 包含 5 bits，最多可以 mask 掉低 31 bits。注意 0b00000 表示未 mask 任何 bits，而 0b00001 和 0b00010 保留为Reserved data。
当使用MASK配置时，debugger必须保证下列全部条件：

- DBGWCR_EL1.BAS 必须为全 1
- DBGWVR_EL1, 被 mask 掉的 address bits 设为 0
  
### Linked watchpoint
watchpoint 通过设置 type field in DBGWCR_EL1.WT，可以配置为以下任意一种：

- Unlinked watchpint：used in isolation
- Linked watchpint：link to Linked Context breakpoint。该配置下，watchpoint event 只有在同时满足 address match 和 context match 才会产生

### Linked watchpoint constraints
- 只有 Linked watchpoint 才能被 link
- linked watchpoint 可以 link 任意类型的 Linked Context Breakpoint。DBGWCR_EL1.LBN Linked Breakpoint Number field，设置了对应的 Linked Context Breakpoint；DBGWCR_EL1.WT.{SSC, HMC, PAC} 定义了产生 watchpoint event 的 execution conditions；DBGBCR_EL1.{SSC, HMC, PMC} 则被忽略
- 一个Linked watchpoint 不能 link到另一个watchpoint。因此DBGWCR_EL1.LBN只能配置Breakpoint
- 如果Linked Context Breakpoint 不是 context-aware的，则该行为unpredictable
- 如果linked watchpoint link到一个Unlinked Context breakpoint，则watchpoint event永远不会发生
- 多个linked watchpoint可以link到一个linked Context Breakpoint；同样的，多个address breakpoint也可以link到一个Linked Context Breakpoint。

## execute conditions
watchpoint 可以配置为在某个 execute condition 下才发生。由 DBGWCR_EL1.WT.{SSC, HMC, PAC} 定义

- SCC：Security State Control。可配置为仅在在 secure state 下、仅在 non-secure state 下发生，或两者都可以。注意是和对应 PE 的 secure state 比，而不是指令fetch 地址的 NS 属性比
- HMC：Higher Mode Control；PAC：privilege Access Control。两者决定了发生 watchpint event 的 exception level 
    - 注意 PAC 决定的是 access privilege。因此 unprivilege load/store 在 EL1 或者更高 level 产生的 watchppoint event 可能是设置在 EL0 级的 addr match（unprivilege load/store 在 EL1 或者更高 level 执行时，会把访问当做还在 EL0 级时进行 check）

## Usage Constraints
see ARM v8 doc D2.10.7

## 产生
### watchpoint debug event产生需要满足以下所有条件
- watchpoint enable。通过设置 DBGWCR_EL1.E watchpoint enable control bit 实现
- conditions in DBGWCR_EL1 meets
- DBGWVR_EL1 地址 compare 成功：
- 如果 watchpoint lint to Linked Context Breakpoint，则 context comparison 也要成功
- 产生 watchpoint 的指令 commit
- 产生 watchpoint 的指令 pass condition code check

### taking watchpoint exception
- PE 在 Fault Address Register 中记录 trigger watchpoint 的地址，使用其中一个地址寄存器： 
    - FAR_EL1：若 exception 在 EL1 中响应
    - FAR_EL2：若 exception 在 EL2 中响应
- PE 在 Exception Syndrome Register（ESR）总记录响应 exception 的 Exception level 
    - ESR_EL1：若 exception 在 EL1 中响应
    - ESR_EL2：若 exception 在 EL2 中响应
    - Table D2-19 in D2.10.8 给出了 ESR 个 bit 的值
- watchpoint 不能在 AArch64 EL3 中响应，为什么？
- 如果一条指令 trigger 多个 watchpoint exception，只记录其中一个 address（哪一个？）
- prefered return address 为产生 watchpoint 的指令 address
- 由非 Dcache instruction 引起的 watchpoint 记录的 address，需要同时满足 
    - inclusive range between 
        - memory 访问的最低地址（The lowest address accessed by the memory access that triggered the watchpoint.）
        - 产生 watchpoint 的 memory 访问的最高地址（The highest watchpointed address accessed by the memory access. A watchpointed address is an address that the watchpoint is watching.）
- 在 naturally-aligned block 访问中，同时满足 
    - size 为2的幂次方
    - 不大于 DC ZVA block size
    - 包含产生 watchpoint 的 memory access

这段描述不容易理解，我们采用ARM文档上的一个例子做以详述：

一个 multiple load 从 0x8004 开始向上取数据，在 0x8019 产生了 watchpoint。如果 DC ZVA block 为 32Bytes，则 block 块的地址应为32的整数
倍（0x8000,0x8020...）。最低位应为 memory access 最低地址 0x8004，而最高位则为产生 watchpoint 的地址。因此地址范围 [0x8004:0x8019]；
0x8010为最低地址，因此地址范围为 [0x8010:0x8019]。

- 由 Dcache instruction 引起的 watchpoint event，地址记录 
    - 记录传给指令的地址。因此该地址可能比引起 watchpoint 的地址位置要高。

### entering Debug State
- PE 在 EDWAR 中记录 trigger watchpoint 的地址
- 进入 debug mode 记录的地址和 exception 有相同的限制，见 taking watchpoint exception

### halting
- 如果允许 halting （设置 EDSCR.HDE），则 watchpint event 会进入 Debug State
- 如果禁止 halting，并且 enable watchpint， watchpoint event 会产生 watchpint exceptions；如果没有 enable watchpoint，则 watchpoint event ignores

### addr match
addr[48:2] match DBGWVR_EL1[48:2]，同时满足

-  access size match（见后节详述）。如果 EL0 是 AArch32，EL1 是 AArch64，则 EL0 指令可以使用 AArch64 stage 1 translation regime，此时地址使用 0 扩展与watchpoint 比较
- DBGWVR_EL1.BAS 设置的 Byte selection，或者
- DBGWVR_EL1.MASK 设置的 addr range

### acess size match
注意 watchpoint 是 data address 访问的任意 byte match 都会产生。因此需要注意访问 size 对 match 的影响。如一个 doubleword 的地址访问 0x1003，会覆盖 0x1003~0x100a 共 8 个 bytes。一些特殊指令的地址定义如下：

- DC ZVA instruction：access size 定义为 DC ZVA block size，在 DCZID_EL0.BS 中定义
- DC IVAC instruction: access size 为 implementation defiend，需要同时满足以下条件： 
    - 为 2KB 可 CTR_EL0_DminLine 定义的 size 之间（inclusive range）
    - 2 的幂次方
- 上述两类指令， 
    - The lowest address accessed by the instruction is the address supplied to the instruction, rounded down to the nearest multiple of the access size initiated by that instruction.（？？）；
    - 最高地址为 (size-1) 的位置
 
## watchpoint的一些特殊行为
- 以下指令从不产生 watchpoint exception 
    - ICache maintenance instruction
    - address translation instruction
    - TLB mainttenance instruction
    - prefetch memory instruction
- Store-Exclusive instruction match watchpoint 
    - 如果store-exclusive fails，是否产生watchpoint exception由implementation defiend
    - 如果store-exclusive succeed，产生watchpoint exception
- cache maintenance instruction match watchpoint 
    - 只有DC IVAC和DC ZVA instruction可以产生watchpoint exception。
    - DBGWCR_EL1.LSC必须配置为一下任意一种 
        - 10： match on data stores
        - 11： match on data stores and data loads