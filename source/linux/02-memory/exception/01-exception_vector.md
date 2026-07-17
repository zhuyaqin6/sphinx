## 一. Excepiton 基础

对于ARM64而言，exception是指cpu的某些异常状态或者一些系统的事件（可能来自外部，也可能来自内部），这些状态或者事件可以导致cpu去执行一些预先设定的，具有更高执行权利的软件（也叫exception handler）。执行exception handler可以进行异常的处理，从而让系统平滑的运行。exception handler执行完毕之后，需要返回发生异常的现场。

### Exception level 异常等级

ARM64最大支持EL0～EL3四个exception level，EL0的execution privilege最低，EL3的execution privilege最高。当发生异常的时候，系统的exception会迁移到更高的exception level或者维持不变，但是绝不会降低。此外，不会有任何的异常会去到EL0。

- 如果没有支持两个security state（但是支持虚拟化），那么ARM64有3个exception level，分别是：EL0（对应user mode的application），EL1（guest OS）和EL2（Hypervisor）。

- 如果支持两个security state（但是不支持虚拟化），ARM64还是有3个exception level，分别是：EL0（对应trusted service），EL1（trusted OS kernel）和EL3（Secure monitor）。

- 如果支持了虚拟化，并且同时支持两种security state，那么ARM64的处理器可以处于4种exception level。

### 异常分类

**asynchronous exception 异步异常**

是毫无预警的，不考虑cpu core感受的外部事件，这些事件打断了cpu core对当前软件的执行，因此也称为 interrupt。
  特点：

     1. 异常与cpu执行的指令无关。
     2. 返回地址是硬件保存下来并提供给handler，以便进行异常返回现场的处理。这个返回地址并非产生异常时的指令。
          即IRQ,FIQ,SError interrupt属于asynchronous exception。

**synchronous exception 同步异常**
特点：
     1. 异常的产生是和cpu core执行的指令或者试图执行有关。
          2. 硬件提供给handler的返回地址就是产生异常的那条指令所在的地址。
          3. 可分为：
           synchronous abort：如未定义的指令，data abort, prefetch intruciton abort, SP未对齐异常，debug exception等；
           正常指令触发： 包括 SVC(EL0->EL1跳转)；HVC(geust os -> hypervisor 跳转)；SMC(normal world -> secure world)。

## Exception level handler

ARM64 异常向量表定义





三. linux code分析 - linux 5.0.1
/arch/arm64/kernel/enty.S

/*
 * Exception vectors.
 */
  /*
	对于一些硬件资源的访问，一般都是存在一些寄存器提供访问的，同时为了让 CPU 也能按照预期执行，
	会设置一些基址供给 CPU 在某些情况下调用，比如异常向量表就是这样的一个寄存器，
	他需要你告诉 CPU 在发生异常时如何处理。	
  */
	.pushsection ".entry.text", "ax"

	.align	11
  ENTRY(vectors)
	kernel_ventry	1, sync_invalid			// Synchronous EL1t
	kernel_ventry	1, irq_invalid			// IRQ EL1t
	kernel_ventry	1, fiq_invalid			// FIQ EL1t
	kernel_ventry	1, error_invalid		// Error EL1t

	/*异常发生在kernel space(EL1)*/
	kernel_ventry	1, sync				// Synchronous EL1h
	kernel_ventry	1, irq				// IRQ EL1h (el1_irq)
	kernel_ventry	1, fiq_invalid			// FIQ EL1h
	kernel_ventry	1, error			// Error EL1h

	/*64bit: 异常发生在userspace(EL0)*/
	kernel_ventry	0, sync				// Synchronous 64-bit EL0
	kernel_ventry	0, irq				// IRQ 64-bit EL0 (el0_irq)
	kernel_ventry	0, fiq_invalid			// FIQ 64-bit EL0
	kernel_ventry	0, error			// Error 64-bit EL0

        /*32bit:  异常发生在userspace(EL0)*/
  #ifdef CONFIG_COMPAT
	kernel_ventry	0, sync_compat, 32		// Synchronous 32-bit EL0
	kernel_ventry	0, irq_compat, 32		// IRQ 32-bit EL0
	kernel_ventry	0, fiq_invalid_compat, 32	// FIQ 32-bit EL0
	kernel_ventry	0, error_compat, 32		// Error 32-bit EL0
  #else
	kernel_ventry	0, sync_invalid, 32		// Synchronous 32-bit EL0
	kernel_ventry	0, irq_invalid, 32		// IRQ 32-bit EL0
	kernel_ventry	0, fiq_invalid, 32		// FIQ 32-bit EL0
	kernel_ventry	0, error_invalid, 32		// Error 32-bit EL0
  #endif
  END(vectors)
  发生不同的异常，跳转到不同的exceptin handler中。

## exception_vector

arch/arm64/kernel/entry.S文件中定义了异常向量表

```c
 513 SYM_CODE_START(vectors)
 514     kernel_ventry   1, sync_invalid         // Synchronous EL1t
 515     kernel_ventry   1, irq_invalid          // IRQ EL1t
 516     kernel_ventry   1, fiq_invalid          // FIQ EL1t
 517     kernel_ventry   1, error_invalid        // Error EL1t
 518
 519     kernel_ventry   1, sync             // Synchronous EL1h
 520     kernel_ventry   1, irq              // IRQ EL1h
 521     kernel_ventry   1, fiq_invalid          // FIQ EL1h
 522     kernel_ventry   1, error            // Error EL1h
 523
 524     kernel_ventry   0, sync             // Synchronous 64-bit EL0
 525     kernel_ventry   0, irq              // IRQ 64-bit EL0
 526     kernel_ventry   0, fiq_invalid          // FIQ 64-bit EL0
 527     kernel_ventry   0, error            // Error 64-bit EL0
 528
 529 #ifdef CONFIG_COMPAT
 530     kernel_ventry   0, sync_compat, 32      // Synchronous 32-bit EL0
 531     kernel_ventry   0, irq_compat, 32       // IRQ 32-bit EL0
 532     kernel_ventry   0, fiq_invalid_compat, 32   // FIQ 32-bit EL0
 533     kernel_ventry   0, error_compat, 32     // Error 32-bit EL0
 534 #else
 535     kernel_ventry   0, sync_invalid, 32     // Synchronous 32-bit EL0 529 #ifdef CONFIG_COMPAT
 530     kernel_ventry   0, sync_compat, 32      // Synchronous 32-bit EL0
 531     kernel_ventry   0, irq_compat, 32       // IRQ 32-bit EL0
 532     kernel_ventry   0, fiq_invalid_compat, 32   // FIQ 32-bit EL0
 533     kernel_ventry   0, error_compat, 32     // Error 32-bit EL0
 534 #else
 535     kernel_ventry   0, sync_invalid, 32     // Synchronous 32-bit EL0
 536     kernel_ventry   0, irq_invalid, 32      // IRQ 32-bit EL0
 537     kernel_ventry   0, fiq_invalid, 32      // FIQ 32-bit EL0
 538     kernel_ventry   0, error_invalid, 32        // Error 32-bit EL0
 539 #endif
 540 SYM_CODE_END(vectors)
```

https://blog.csdn.net/weixin_42135087/article/details/120232101

https://blog.csdn.net/vertor11/article/details/104734101