# 实验1

_吴昊 PB16001800_

# 实验目的

追踪 Linux 操作系统内核的启动

本次实验选择了 Linux 3.18.102 版本的内核

系统环境：Ubuntu 17.10

# 准备工作

1. 从官方网站 https://www.kernel.org/ 下载 3.18.102 的内核源代码

2. `xz -d linux-3.18.102.tar.xz`

3. `tar -xvf linux-3.18.102.tar`

2. `make i386_defconfig` 配置为 x86 的默认编译选项

3. `make menuconfig` 进入 makefile 的字符图形界面（TUI）

   注意这一步可能会失败，出现 `curses.h` 未找到等提示，这是因为 Ubuntu 缺少 `libncurses5-dev` 这个包，使用 `sudo apt install libncurses5-dev` 即可

4. 在界面中找到`kernel hacking` ，勾选（enter来勾选）`kernel debugging`，然后进入

`Compile-time checks and compiler options` 勾选`Compile the kernel with debug info`

5. `make -j16 ` 用 16 个核心并行编译

# 调试过程

1. `qemu-system-i386 -kernel arch/i386/boot/bzimage -s -S` 用虚拟机启动 Linux 内核

   -S：开机时冻结CPU（暂停）

   -s：在 1234 端口创建 gdb server（相当于-gdb tcp:1234）

2. `gdb vmlinux`  用 gdb 打开 linux 内核镜像

3. `(gdb) target remote:1234` 连接 1234 端口

4. `break start_kernel` 在 start_kernel 函数处中断

   start_kernel 是内核代码中 汇编 与 C语言 的分界线

   在此之前的汇编代码负责解压缩内核

5. 输入`continue` ，让内核继续执行，直到遇到`start_kernel`函数停止

6. 用`list `命令查看`start_kernel` 函数附近的代码

7. 几个函数的解释

   - 509 行 `lockdep_init()` 是 Linux 死锁检测模块
   - 510 行`set_task_stack_end_magic(&init_task);` 启动了0号进程（PID = 0，即 idle ）    
   - 517 行 `boot_init_stack_canary()`初始化 canary word
   - 561 行 `trap_init()` 初始化系统异常
   - 562 行 `mm_init()` 初始化内存管理
   - 603 行 有 Hack Alert，在初始化 console 之前

   之后的工作就是各种初始化，不再赘述

8. `break rest_init` ，然后`continue` ，可以追踪到PID 1 的启动和创建

   - 397 行`rcu_scheduler_starting();` 

     启动了 RCU 锁机制


   - 403 行 `kernel_thread(kernel_init, NULL, CLONE_FS);` 

   ​	创建 init 进程（PID = 1），但不能调度它，原因如下：

   > ​	 We need to spawn init first so that it obtains pid 1, however
   >
   > ​	the init task will end up wanting to create kthreads, which, if
   >
   > ​	we schedule it before we create kthreadd, will OOPS.

   - 405 行 `pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);`

   ​	创建 kthreadd 内核线程（PID = 2），它的作用是管理和调度其它内核线程。

   - 407 行 `kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);`

     获取kthreadd的线程信息，获取完成说明kthreadd已经创建成功。

   - 409 行 `complete(&kthreadd_done);` 通知 kernel_init 线程 PID = 2 已经创建完成。

   至此，PID 1 和 PID 2 已经创建完成



# 附录

gdb日志

```c
GNU gdb (Ubuntu 8.0.1-0ubuntu1) 8.0.1
Copyright (C) 2017 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from vmlinux...done.
(gdb) remote [K[K[K[K[K[K[Ktarget remote [K:1234
Remote debugging using :1234
0x0000fff0 in ?? ()
(gdb) breakpo[K[K[K[K[K[K [Kreak start_kernel
Breakpoint 1 at 0xc1a4771f: file init/main.c, line 501.
(gdb) c
Continuing.

Breakpoint 1, start_kernel () at init/main.c:501
501	{
(gdb) list
496		pgtable_init();
497		vmalloc_init();
498	}
499	
500	asmlinkage __visible void __init start_kernel(void)
501	{
502		char *command_line;
503		char *after_dashes;
504	
505		/*
(gdb) list
506		 * Need to run as early as possible, to initialize the
507		 * lockdep hash:
508		 */
509		lockdep_init();
510		set_task_stack_end_magic(&init_task);
511		smp_setup_processor_id();
512		debug_objects_early_init();
513	
514		/*
515		 * Set up the the initial canary ASAP:
(gdb) list
516		 */
517		boot_init_stack_canary();
518	
519		cgroup_init_early();
520	
521		local_irq_disable();
522		early_boot_irqs_disabled = true;
523	
524	/*
525	 * Interrupts are still disabled. Do necessary setups, then
(gdb) list
526	 * enable them
527	 */
528		boot_cpu_init();
529		page_address_init();
530		pr_notice("%s", linux_banner);
531		setup_arch(&command_line);
532		mm_init_cpumask(&init_mm);
533		setup_command_line(command_line);
534		setup_nr_cpu_ids();
535		setup_per_cpu_areas();
(gdb) list
536		smp_prepare_boot_cpu();	/* arch-specific boot-cpu hooks */
537	
538		build_all_zonelists(NULL, NULL);
539		page_alloc_init();
540	
541		pr_notice("Kernel command line: %s\n", boot_command_line);
542		parse_early_param();
543		after_dashes = parse_args("Booting kernel",
544					  static_command_line, __start___param,
545					  __stop___param - __start___param,
(gdb) list
546					  -1, -1, &unknown_bootoption);
547		if (!IS_ERR_OR_NULL(after_dashes))
548			parse_args("Setting init args", after_dashes, NULL, 0, -1, -1,
549				   set_init_arg);
550	
551		jump_label_init();
552	
553		/*
554		 * These use large bootmem allocations and must precede
555		 * kmem_cache_init()
(gdb) list
556		 */
557		setup_log_buf(0);
558		pidhash_init();
559		vfs_caches_init_early();
560		sort_main_extable();
561		trap_init();
562		mm_init();
563	
564		/*
565		 * Set up the scheduler prior starting any interrupts (such as the
(gdb) list
566		 * timer interrupt). Full topology setup happens at smp_init()
567		 * time - but meanwhile we still have a functioning scheduler.
568		 */
569		sched_init();
570		/*
571		 * Disable preemption - early bootup scheduling is extremely
572		 * fragile until we cpu_idle() for the first time.
573		 */
574		preempt_disable();
575		if (WARN(!irqs_disabled(),
(gdb) list
576			 "Interrupts were enabled *very* early, fixing it\n"))
577			local_irq_disable();
578		idr_init_cache();
579		rcu_init();
580		context_tracking_init();
581		radix_tree_init();
582		/* init some links before init_ISA_irqs() */
583		early_irq_init();
584		init_IRQ();
585		tick_init();
(gdb) list
586		rcu_init_nohz();
587		init_timers();
588		hrtimers_init();
589		softirq_init();
590		timekeeping_init();
591		time_init();
592		sched_clock_postinit();
593		perf_event_init();
594		profile_init();
595		call_function_init();
(gdb) list
596		WARN(!irqs_disabled(), "Interrupts were enabled early\n");
597		early_boot_irqs_disabled = false;
598		local_irq_enable();
599	
600		kmem_cache_init_late();
601	
602		/*
603		 * HACK ALERT! This is early. We're enabling the console before
604		 * we've done PCI setups etc, and console_init() must be aware of
605		 * this. But we do want output early, in case something goes wrong.
(gdb) list
606		 */
607		console_init();
608		if (panic_later)
609			panic("Too many boot %s vars at `%s'", panic_later,
610			      panic_param);
611	
612		lockdep_info();
613	
614		/*
615		 * Need to run this when irqs are enabled, because it wants
(gdb) list
616		 * to self-test [hard/soft]-irqs on/off lock inversion bugs
617		 * too:
618		 */
619		locking_selftest();
620	
621	#ifdef CONFIG_BLK_DEV_INITRD
622		if (initrd_start && !initrd_below_start_ok &&
623		    page_to_pfn(virt_to_page((void *)initrd_start)) < min_low_pfn) {
624			pr_crit("initrd overwritten (0x%08lx < 0x%08lx) - disabling it.\n",
625			    page_to_pfn(virt_to_page((void *)initrd_start)),
(gdb) list
626			    min_low_pfn);
627			initrd_start = 0;
628		}
629	#endif
630		page_cgroup_init();
631		debug_objects_mem_init();
632		kmemleak_init();
633		setup_per_cpu_pageset();
634		numa_policy_init();
635		if (late_time_init)
(gdb) list
636			late_time_init();
637		sched_clock_init();
638		calibrate_delay();
639		pidmap_init();
640		anon_vma_init();
641		acpi_early_init();
642	#ifdef CONFIG_X86
643		if (efi_enabled(EFI_RUNTIME_SERVICES))
644			efi_enter_virtual_mode();
645	#endif
(gdb) list
646	#ifdef CONFIG_X86_ESPFIX64
647		/* Should be run before the first non-init thread is created */
648		init_espfix_bsp();
649	#endif
650		thread_info_cache_init();
651		cred_init();
652		fork_init(totalram_pages);
653		proc_caches_init();
654		buffer_init();
655		key_init();
(gdb) list
656		security_init();
657		dbg_late_init();
658		vfs_caches_init(totalram_pages);
659		signals_init();
660		/* rootfs populating might need page-writeback */
661		page_writeback_init();
662		proc_root_init();
663		cgroup_init();
664		cpuset_init();
665		taskstats_init_early();
(gdb) list
666		delayacct_init();
667	
668		check_bugs();
669	
670		acpi_subsystem_init();
671		sfi_init_late();
672	
673		if (efi_enabled(EFI_RUNTIME_SERVICES)) {
674			efi_late_init();
675			efi_free_boot_services();
(gdb) b rest_init
Breakpoint 2 at 0xc1758c30: file init/main.c, line 394.
(gdb) b kernel+init[K[K[K[K[K_init
Breakpoint 3 at 0xc1758c90: file init/main.c, line 932.
(gdb) b kthread
Breakpoint 4 at 0xc105b350: file kernel/kthread.c, line 176.
(gdb) b kernel_start
Make breakpoint pending on future shared library load? (y or [n]) n
(gdb) b kernel[K[K[K[K[K[Kstart_kernel
Note: breakpoint 1 also set at pc 0xc1a4771f.
Breakpoint 5 at 0xc1a4771f: file init/main.c, line 501.
(gdb) c
Continuing.

Breakpoint 2, rest_init () at init/main.c:394
394	{
(gdb) list
389	 */
390	
391	static __initdata DECLARE_COMPLETION(kthreadd_done);
392	
393	static noinline void __init_refok rest_init(void)
394	{
395		int pid;
396	
397		rcu_scheduler_starting();
398		/*
(gdb) list
399		 * We need to spawn init first so that it obtains pid 1, however
400		 * the init task will end up wanting to create kthreads, which, if
401		 * we schedule it before we create kthreadd, will OOPS.
402		 */
403		kernel_thread(kernel_init, NULL, CLONE_FS);
404		numa_default_policy();
405		pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
406		rcu_read_lock();
407		kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
408		rcu_read_unlock();
(gdb) b rcu_scheduler_starting();[K[K[K
Breakpoint 6 at 0xc108d080: file include/linux/bitmap.h, line 305.
(gdb) l
409		complete(&kthreadd_done);
410	
411		/*
412		 * The boot idle thread must execute schedule()
413		 * at least once to get things moving:
414		 */
415		init_idle_bootup_task(current);
416		schedule_preempt_disabled();
417		/* Call into cpu_idle with preempt disabled */
418		cpu_startup_entry(CPUHP_ONLINE);
(gdb) c
Continuing.

Breakpoint 6, rcu_scheduler_starting () at kernel/rcu/tree.c:3552
3552		WARN_ON(num_online_cpus() != 1);
(gdb) list[K[K[K
3547	 * idle tasks are prohibited from containing RCU read-side critical
3548	 * sections.  This function also enables RCU lockdep checking.
3549	 */
3550	void rcu_scheduler_starting(void)
3551	{
3552		WARN_ON(num_online_cpus() != 1);
3553		WARN_ON(nr_context_switches() > 0);
3554		rcu_scheduler_active = 1;
3555	}
3556	
(gdb) l
3557	/*
3558	 * Compute the per-level fanout, either using the exact fanout specified
3559	 * or balancing the tree, depending on CONFIG_RCU_FANOUT_EXACT.
3560	 */
3561	#ifdef CONFIG_RCU_FANOUT_EXACT
3562	static void __init rcu_init_levelspread(struct rcu_state *rsp)
3563	{
3564		int i;
3565	
3566		rsp->levelspread[rcu_num_lvls - 1] = rcu_fanout_leaf;
(gdb) l
3567		for (i = rcu_num_lvls - 2; i >= 0; i--)
3568			rsp->levelspread[i] = CONFIG_RCU_FANOUT;
3569	}
3570	#else /* #ifdef CONFIG_RCU_FANOUT_EXACT */
3571	static void __init rcu_init_levelspread(struct rcu_state *rsp)
3572	{
3573		int ccur;
3574		int cprv;
3575		int i;
3576	
(gdb) l
3577		cprv = nr_cpu_ids;
3578		for (i = rcu_num_lvls - 1; i >= 0; i--) {
3579			ccur = rsp->levelcnt[i];
3580			rsp->levelspread[i] = (cprv + ccur - 1) / ccur;
3581			cprv = ccur;
3582		}
3583	}
3584	#endif /* #else #ifdef CONFIG_RCU_FANOUT_EXACT */
3585	
3586	/*
(gdb) l
3587	 * Helper function for rcu_init() that initializes one rcu_state structure.
3588	 */
3589	static void __init rcu_init_one(struct rcu_state *rsp,
3590			struct rcu_data __percpu *rda)
3591	{
3592		static const char * const buf[] = {
3593			"rcu_node_0",
3594			"rcu_node_1",
3595			"rcu_node_2",
3596			"rcu_node_3" };  /* Match MAX_RCU_LVLS */
(gdb) l
3597		static const char * const fqs[] = {
3598			"rcu_node_fqs_0",
3599			"rcu_node_fqs_1",
3600			"rcu_node_fqs_2",
3601			"rcu_node_fqs_3" };  /* Match MAX_RCU_LVLS */
3602		static u8 fl_mask = 0x1;
3603		int cpustride = 1;
3604		int i;
3605		int j;
3606		struct rcu_node *rnp;
(gdb) '[Kl
3607	
3608		BUILD_BUG_ON(MAX_RCU_LVLS > ARRAY_SIZE(buf));  /* Fix buf[] init! */
3609	
3610		/* Silence gcc 4.8 warning about array index out of range. */
3611		if (rcu_num_lvls > RCU_NUM_LVLS)
3612			panic("rcu_init_one: rcu_num_lvls overflow");
3613	
3614		/* Initialize the level-tracking arrays. */
3615	
3616		for (i = 0; i < rcu_num_lvls; i++)
(gdb) l
3617			rsp->levelcnt[i] = num_rcu_lvl[i];
3618		for (i = 1; i < rcu_num_lvls; i++)
3619			rsp->level[i] = rsp->level[i - 1] + rsp->levelcnt[i - 1];
3620		rcu_init_levelspread(rsp);
3621		rsp->flavor_mask = fl_mask;
3622		fl_mask <<= 1;
3623	
3624		/* Initialize the elements themselves, starting from the leaves. */
3625	
3626		for (i = rcu_num_lvls - 1; i >= 0; i--) {
(gdb) l
3627			cpustride *= rsp->levelspread[i];
3628			rnp = rsp->level[i];
3629			for (j = 0; j < rsp->levelcnt[i]; j++, rnp++) {
3630				raw_spin_lock_init(&rnp->lock);
3631				lockdep_set_class_and_name(&rnp->lock,
3632							   &rcu_node_class[i], buf[i]);
3633				raw_spin_lock_init(&rnp->fqslock);
3634				lockdep_set_class_and_name(&rnp->fqslock,
3635							   &rcu_fqs_class[i], fqs[i]);
3636				rnp->gpnum = rsp->gpnum;
(gdb) l
3637				rnp->completed = rsp->completed;
3638				rnp->qsmask = 0;
3639				rnp->qsmaskinit = 0;
3640				rnp->grplo = j * cpustride;
3641				rnp->grphi = (j + 1) * cpustride - 1;
3642				if (rnp->grphi >= nr_cpu_ids)
3643					rnp->grphi = nr_cpu_ids - 1;
3644				if (i == 0) {
3645					rnp->grpnum = 0;
3646					rnp->grpmask = 0;
(gdb) l
3647					rnp->parent = NULL;
3648				} else {
3649					rnp->grpnum = j % rsp->levelspread[i - 1];
3650					rnp->grpmask = 1UL << rnp->grpnum;
3651					rnp->parent = rsp->level[i - 1] +
3652						      j / rsp->levelspread[i - 1];
3653				}
3654				rnp->level = i;
3655				INIT_LIST_HEAD(&rnp->blkd_tasks);
3656				rcu_init_one_nocb(rnp);
(gdb) l
3657			}
3658		}
3659	
3660		rsp->rda = rda;
3661		init_waitqueue_head(&rsp->gp_wq);
3662		rnp = rsp->level[rcu_num_lvls - 1];
3663		for_each_possible_cpu(i) {
3664			while (i > rnp->grphi)
3665				rnp++;
3666			per_cpu_ptr(rsp->rda, i)->mynode = rnp;
(gdb) b complete
Breakpoint 7 at 0xc10751c0: file kernel/sched/completion.c, line 30.
(gdb) c
Continuing.

Breakpoint 7, complete (x=0xc1a93884 <kthreadd_done>)
    at kernel/sched/completion.c:30
30	{
(gdb) list
25	 *
26	 * It may be assumed that this function implies a write memory barrier before
27	 * changing the task state if and only if any tasks are woken up.
28	 */
29	void complete(struct completion *x)
30	{
31		unsigned long flags;
32	
33		spin_lock_irqsave(&x->wait.lock, flags);
34		x->done++;
(gdb) 
```





