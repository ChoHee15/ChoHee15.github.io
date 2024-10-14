---
title: xvisor
date: 2024/09/14 15:42:44
tags:
  - xvisor
  - arceos
categories:
  - xvisor
share: "true"
---

# 1. 运行


较新的toolchain会使用2.38或更高版本的binutil，在编译较低版本xviosr时会有问题，面对csr指令报``Error: unrecognized opcode``错误或者``'zifencei'``相关错误。

[Compilation Issue · Issue #142 · xvisor/xvisor · GitHub](https://github.com/xvisor/xvisor/issues/142)

[Invalid or unknown z ISA extension: 'zifencei' · Issue #150 · xvisor/xvisor · GitHub](https://github.com/xvisor/xvisor/issues/150)

此问题在2022.04.28的某个提交被修复。

![[xvisor.assets/binutil fix.png|xvisor.assets/binutil fix.png]]

在编好更低版本的toolchain后尝试用其编译v0.3.1版本，基本还是跟着xvisor的``riscv64-qemu.txt``文档走。

[Exploring virtualization in RISC-V machines - embeddedinn](https://embeddedinn.com/articles/tutorial/exploring_virtualization_in_riscv_machines/)

```bash
# 构建xvisor
cd <xvisor repo>
CROSS_COMPILE=riscv64-unknown-linux-gnu- make ARCH=riscv generic-64b-defconfig
CROSS_COMPILE=riscv64-unknown-linux-gnu- make -j$(nproc)

# 构建sbi
cd <opensbi_source_directory>
CROSS_COMPILE=riscv64-unknown-linux-gnu- make PLATFORM=generic

# 编译linux
cd <linux_source_directory>

cp arch/riscv/configs/defconfig arch/riscv/configs/tmp-virt64_defconfig
<xvisor_source_directory>/tests/common/scripts/update-linux-defconfig.sh -p arch/riscv/configs/tmp-virt64_defconfig -f <xvisor_source_directory>/tests/riscv/virt64/linux/linux_extra.config
#之前编过linux，所以make ARCH=riscv mrproper清理一下
make ARCH=riscv mrproper
mkdir build_riscv
make O=./build_riscv/ ARCH=riscv tmp-virt64_defconfig

CROSS_COMPILE=riscv64-unknown-linux-gnu- make -j$(nproc) O=./build_riscv/ ARCH=riscv Image dtbs

# 制作镜像
...


# 运行
# mkdir -p ./build/disk/tmp
# mkdir -p ./build/disk/system
# cp -f ./docs/banner/roman.txt ./build/disk/system/banner.txt
# cp -f ./docs/logo/xvisor_logo_name.ppm ./build/disk/system/logo.ppm
# mkdir -p ./build/disk/images/riscv/virt64
# dtc -q -I dts -O dtb -o ./build/disk/images/riscv/virt64-guest.dtb ./tests/riscv/virt64/virt64-guest.dts
# cp -f ./build/tests/riscv/virt64/basic/firmware.bin ./build/disk/images/riscv/virt64/firmware.bin
# cp -f ./tests/riscv/virt64/linux/nor_flash.list ./build/disk/images/riscv/virt64/nor_flash.list
# cp -f ./tests/riscv/virt64/linux/cmdlist ./build/disk/images/riscv/virt64/cmdlist
# cp -f ./tests/riscv/virt64/xscript/one_guest_virt64.xscript ./build/disk/boot.xscript
# cp -f <linux_build_directory>/arch/riscv/boot/Image ./build/disk/images/riscv/virt64/Image
# dtc -q -I dts -O dtb -o ./build/disk/images/riscv/virt64/virt64.dtb ./tests/riscv/virt64/linux/virt64.dts
# cp -f <busybox_rootfs_directory>/rootfs.img ./build/disk/images/riscv/virt64/rootfs.img
# genext2fs -B 1024 -b 32768 -d ./build/disk ./build/disk.img


qemu-system-riscv64 -cpu rv64,x-h=true -M virt -m 512M -nographic -bios /codes/opensbi/build/platform/generic/firmware/fw_jump.bin -kernel ./build/vmm.bin -initrd ./build/disk.img -append 'vmm.bootcmd="vfs mount initrd /;vfs run /boot.xscript;vfs cat /system/banner.txt"'

qemu-system-riscv64 -M virt -m 512M -nographic -bios /codes/opensbi/build/platform/generic/firmware/fw_jump.bin -kernel ./build/vmm.bin -initrd ./build/disk.img -append 'vmm.bootcmd="vfs mount initrd /;vfs run /boot.xscript;vfs cat /system/banner.txt"'

qemu-system-riscv64 -M virt -m 512M -smp 2 -nographic -bios /codes/opensbi/build/platform/generic/firmware/fw_jump.bin -kernel ./build/vmm.bin -initrd ./build/disk.img -append 'vmm.bootcmd="vfs mount initrd /;vfs run /boot.xscript;vfs cat /system/banner.txt"'

XVisor# guest kick guest0

XVisor# vserial bind guest0/uart0

[guest0/uart0] basic# autoexec


qemu-system-riscv64 -M virt -m 512M -smp 1 -nographic -bios /codes/opensbi/build/platform/generic/firmware/fw_jump.bin -kernel ./build/vmm.bin -initrd ./build/disk.img -append 'vmm.bootcmd="vfs mount initrd /;vfs run /boot.xscript;vfs cat /system/banner.txt"' -drive file=/arceos_2024S/arceos/apps/hv/guest/linux/rootfs.img,if=none,id=drive0 -device virtio-blk-device,drive=drive0,id=virtioblk0


qemu-system-riscv64 -M virt -m 512M -smp 1 -nographic -bios /codes/opensbi/build/platform/generic/firmware/fw_jump.bin -kernel ./build/vmm.bin -initrd ./build/disk.img -append 'root=/dev/vda vmm.bootcmd="vfs mount initrd /;vfs run /boot.xscript;vfs cat /system/banner.txt"' -drive file=/arceos_2024S/arceos/apps/hv/guest/linux/rootfs.img,if=none,id=drive0 -device virtio-blk-device,drive=drive0,id=virtioblk0


"root=/dev/vda rw console=ttyS0,115200 earlycon=uart8250,mmio,0x10000000"
# earlycon=uart8250使用8250串口作为启动早期串口，被映射到0x10000000


qemu-system-riscv64 -M virt -m 512M -smp 2 -nographic -bios /codes/opensbi/build/platform/generic/firmware/fw_jump.bin -kernel ./build/vmm.bin -initrd ./build/disk.img -append 'root=/dev/vda vmm.bootcmd="vfs mount initrd /;vfs run /boot.xscript;vfs cat /system/banner.txt"' -drive file=/arceos_2024S/arceos/apps/hv/guest/linux/rootfs.img,if=none,id=drive0 -device virtio-blk-device,drive=drive0,id=virtioblk0 -s -S

/qemu-9.0.0-rc1/build/qemu-system-riscv64 -M virt -m 512M -smp 2 -nographic -bios /codes/opensbi/build/platform/generic/firmware/fw_jump.bin -kernel ./build/vmm.bin -initrd ./build/disk.img -append 'root=/dev/vda vmm.bootcmd="vfs mount initrd /;vfs run /boot.xscript;vfs cat /system/banner.txt"' -drive file=/arceos_2024S/arceos/apps/hv/guest/linux/rootfs.img,if=none,id=drive0 -device virtio-blk-device,drive=drive0,id=virtioblk0



```


# 2. 爆炸

```bash
[guest0/uart0] [    0.000000] riscv-intc: 64 local interrupts mapped
[guest0/uart0] devemu_dowrite: edev=plic offset=0x0000000000000080 src_len=4 failed (error -5)
vmm_devemu_emulate_write: vcpu=guest0/vcpu0 gphys=0x000000000C000080 src_len=4 failed (error -5)
vcpu=guest0/vcpu0 current_state=0x20 to new_state=0x20 failed (error -5)
WARNING: vmm_scheduler_state_change() at /codes/xvisor/core/vmm_scheduler.c:696
0x10011040 vmm_lprintf+0x20/0x2c
0x10025A22 vmm_scheduler_state_change+0x12c/0x496
0x10023498 vmm_manager_vcpu_set_state+0x10/0x2c
0x10002452 do_handle_trap+0xbe/0x1d2
0x10002576 do_handle_exception+0x10/0x2c
0x100003C2 _handle_hyp_exception+0x72/0xd4
do_error: CPU0: VCPU=guest0/vcpu0 page fault failed (error -5)
           zero=0x0000000000000000          ra=0xFFFFFFFF80A1D484
             sp=0xFFFFFFFF81403DB0          gp=0xFFFFFFFF814FD780
             tp=0xFFFFFFFF8140FAC0          s0=0xFFFFFFFF81403EE0
             s1=0xFF600000016043C0          a0=0xFF6000000EDCDDA8
             a1=0x0000000000000000          a2=0x0000000000000080
             a3=0xFF20000000000080          a4=0x0000000000000000
             a5=0x0000000000000001          a6=0xFF60000001A00248
             a7=0xFF60000001A002B8          s2=0x0000000000000001
             s3=0xFF6000000FDF8110          s4=0x0000000000000004
             s5=0x0000000000000000          s6=0xFFFFFFFF80C1FD98
             s7=0xFFFFFFFF8100DF20          s8=0xFF6000000EDCDDA8
             s9=0xFF6000000EDCDD98         s10=0x0000000000000020
            s11=0x0000000000000001          t0=0x0000000000000040
             t1=0x0000000000000000          t2=0x0000000000000400
             t3=0x0000000000000002          t4=0x0000000000000402
             t5=0xFFFFFFFF814890E8          t6=0xFFFFFFFF81489108
           sepc=0xFFFFFFFF80A1D490     sstatus=0x0000000200004120
        hstatus=0x00000002002001C0     sp_exec=0x0000000010A40000
         scause=0x0000000000000017       stval=0xFF20000000000080
          htval=0x0000000003000020      htinst=0x0000000000000000
QEMU: Terminated
```


使用2022.04.28的一个版本出了问题，在issue找到了相关内容：

[riscv: run linux in xvisor, occur fault on wirte plic reg · Issue #146 · xvisor/xvisor · GitHub](https://github.com/xvisor/xvisor/issues/146)

这是2022.09发布的，应该会在之后的版本修复。

里面提到更新到xvisor-next里了，我找了找这个仓库9月份的提交，找到了相应commit：

[EMULATORS: plic: Fix number of irq lines · avpatel/xvisor-next@e475b1a · GitHub](https://github.com/avpatel/xvisor-next/commit/e475b1ab349cc0ed4d02d6bfc824802493d5aae2#diff-b2c9c4a762821f34d6005e0fc3edefb58668a928202f9f0b0b564035beb7a53c)

好像是中断号的问题，我直接在当前版本上修改了，修改后可以运行。

![[xvisor.assets/patch.png|xvisor.assets/patch.png]]



# x. 复活

会因无法读取/dev/ram设备启动失败。

找了半天发现guest的启动参数默认在arch_board.c里写死了

``root=/dev/ram rw console=ttyS0,115200 earlycon=uart8250,mmio,0x10000000``

需要在vserial绑定后，用linux_cmdline命令修改。

配合vdisk attach，可以用另一个磁盘镜像启动：

https://github.com/xvisor/xvisor/issues/168

nice的

# 3. vscode debug

经典

```json
{
    // 使用 IntelliSense 了解相关属性。
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "qemu connect",
            "type": "cppdbg",
            "request": "launch",
            // elf文件，从中获得debug信息
            "program": "${workspaceFolder}/build/vmm.elf",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${fileDirname}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": false
                },
                // {
                //     "description": "将反汇编风格设置为 Intel",
                //     "text": "set disassembly-flavor intel",
                //     "ignoreFailures": false
                // },
                {
                    "description": "告别Cannot access memory",
                    "text": "set riscv use-compressed-breakpoints yes",
                    "ignoreFailures": false
                }
            ],
            "miDebuggerPath": "riscv64-unknown-linux-gnu-gdb",
            // qemu启动gdb server的地址，通常为localhost:1234
            "miDebuggerServerAddress": "localhost:1234",
            // 如果出现蜜汁问题可以打开gdb的log，在调试控制台查看更详细的信息
            // "logging": { "engineLogging": true },
            // "logging": {
            //     "moduleLoad": false,
            //     "trace": true,
            //     "engineLogging": false,
            //     "programOutput": true,
            //     "exceptions": false
            // }
        }   
    ]
}

// qemu-system-riscv64 -M virt -m 512M -nographic -bios /codes/opensbi/build/platform/generic/firmware/fw_jump.bin -kernel ./build/vmm.bin -initrd ./build/disk.img -append 'vmm.bootcmd="vfs mount initrd /;vfs run /boot.xscript;vfs cat /system/banner.txt"' -s -S

```




```c
&per_cpu(chpstate, cpu)
==
&(*RELOC_HIDE(&percpu_chpstate, __percpu_offset[cpu]))
==
&(*({(typeof(&percpu_chpstate)) ((virtual_addr_t)(&percpu_chpstate)) + __percpu_offset[cpu]}))
==

```




# 4. PLIC

vcpu尝试访问设备后会触发``STORE_GUEST_PAGE_FAULT``，根据对应地址等信息执行``cpu_vcpu_emulate_store/load``，以及相应``vmm_devemu_emulate_write/read``。

查找地址所在的``vmm_region``对应哪个模拟设备，取此设备的读写操作。

xvisor似乎对所有设备采用模拟操作？包括串口。在linux启动过程中对串口进行设置时，虚拟串口设备会从自己所属的guest中找到对应chip的处理函数，即``plic_irq_handle``，给guest产生一个虚拟中断。

大致流程为：

1. guest读写plic设置使能/优先级等，因PAGE_FAULT被拦截，并被host设置到虚拟plic中。
2. guest读写设备，因PAGE_FAULT被拦截，host使用虚拟设备进行对应操作，并在某些情况由虚拟设备引起一个“软件层面”的中断。
3. 这个引起中断的操作，实际上导向给虚拟PLIC设置pending，模拟了现实中物理设备和PLIC的交互。
4. 改变虚拟PLIC的状态如pending后，可能会执行``__plic_context_irq_update``，其中会依据虚拟PLIC中使能/优先级等配置，找出当前虚拟PLIC是否有可以发出的中断，若存在，给vcpu assert一个中断，实际上是设置了vcpu结构体中irq相关信息。
5. ``vmm_manager_vcpu_hcpu_func``和``vcpu_irq_wfi_resume``没太看懂，似乎和调度有关。
6. 处理完``plic_irq_handle``后一路返回到``do_handle_trap``中，此函数最后的。``vmm_scheduler_irq_exit``中，会根据当前vcpu的irq信息，设置相应中断注入。例如若之前给vcpu assert了中断，那么此处通过查找``vcpu->irqs.irq[irq_no].assert``就会得知，并设置``hvip``。
7. ``_handle_hyp_exception``返回后回到guest，若``hvip``被设置则在guest中触发中断。



>problem：
>
>1. ``vcpu_irq_wfi_resume``相关。
>
>2. 若在二阶段页表中设置对设备的等值映射，相当于vm能够直接控制设备？




# 5. wfi & ipi

xvisor的设计中，wfi被设置为不可在虚拟机中直接执行。

当hstatus.VTW=1 and mstatus.TW=0时，在vs mode下执行wfi会触发虚拟指令异常。

此时若vcpu执行wfi，会退出虚拟机。vmm便可标记此vcpu处于wfi状态，并将其置于不可运行状态并调度走。

当给vcpu assert irq时，若此vcpu处于wfi状态，便可清除其wfi标记，并将其重新置于可运行状态。



当中断被首次assert到vcpu上后，会执行

```rust
vmm_manager_vcpu_hcpu_func(vcpu,
                       VMM_VCPU_STATE_INTERRUPTIBLE,
                       vcpu_irq_wfi_resume, NULL, FALSE);
```

其中会查看vcpu是否处于可中断状态，并获取此vcpu所在的物理cpu编号，形成cpu_mask

```rust
if (arch_atomic_read(&vcpu->state) & state_mask) {
	cpu_mask = vmm_cpumask_of(vcpu->hcpu);
}
```

最终会调用

```rust
vmm_smp_ipi_sync_call(cpu_mask, 0,
                    manager_vcpu_hcpu_func,
                    func, vcpu, data);
==
vmm_smp_ipi_sync_call(cpu_mask, 0,
                    manager_vcpu_hcpu_func,
                    vcpu_irq_wfi_resume, vcpu, data);
```

其中有逻辑：

```rust
for_each_cpu(c, dest) {
	if (c == cpu) {
		func(arg0, arg1, arg2); //若当前cpu就是目标cpu，则执行func
	} else {
		if (!vmm_cpu_online(c)) { // 否则让当前cpu向目标cpu发送ipigute？
			continue;
		}

		ipic.src_cpu = cpu;
		ipic.dst_cpu = c;
		ipic.func = func;
		ipic.arg0 = arg0;
		ipic.arg1 = arg1;
		ipic.arg2 = arg2;
		smp_ipi_sync_submit(&per_cpu(ictl, c), &ipic);
		vmm_cpumask_set_cpu(c, &trig_mask);
		trig_count++;
	}
}
```

遍历目标cpu_mask中标记的每个cpu，若此cpu就是当前物理cpu，则直接执行func，即由``manager_vcpu_hcpu_func``包装的``vcpu_irq_wfi_resume``。

若标记中的cpu不是当前物理cpu，则执行``smp_ipi_sync_submit``。

其中会将ipi信息入队到目标物理cpu的队列中，并调用``arch_smp_ipi_trigger``，来引发目标物理cpu上的ipi：

```rust
while (!fifo_enqueue(ictlp->sync_fifo, ipic, FALSE) && try) {
	arch_smp_ipi_trigger(vmm_cpumask_of(ipic->dst_cpu));
	vmm_udelay(SMP_IPI_WAIT_UDELAY);
	try--;
}

void arch_smp_ipi_trigger(const struct vmm_cpumask *dest)
{
	/* Raise IPI to other cores */
	if (smp_ipi_available) {
		vmm_host_irq_raise(smp_ipi_irq, dest);
	}
}

int vmm_host_irq_raise(u32 hirq, const struct vmm_cpumask *dest) {
	struct vmm_host_irq *irq;
	
	if (NULL == (irq = vmm_host_irq_get(hirq)))
		return VMM_ENOTAVAIL;
		
	if (irq->chip && irq->chip->irq_raise) {
		irq->chip->irq_raise(irq, dest);
	}
	return VMM_OK;
}
```

其中irq_raise定义于``drivers/irqchip/irq-riscv-aclint-swi.c``中：

```rust
static struct vmm_host_irq_chip aclint_swi_irqchip = {
	.name = "riscv-aclint-swi",
	.irq_mask = aclint_swi_dummy,
	.irq_unmask = aclint_swi_dummy,
	.irq_raise = aclint_swi_raise
};

static void aclint_swi_raise(struct vmm_host_irq *d,
			     const struct vmm_cpumask *mask) {
	u32 cpu;
	void *swi_reg;

	for_each_cpu(cpu, mask) {
		swi_reg = per_cpu(aclint_swi_reg, cpu);
		vmm_writel(1, swi_reg);
	}
}

```

可见其会向aclint_swi_reg写入1。

而aclint_swi_reg在``aclint_swi_init()``被初始化：

```rust
/* Map ACLINT SWI registers */
rc = vmm_devtree_request_regmap(node, &va, 0, "RISC-V ACLINT SWI");
...
for_each_possible_cpu(cpu) {
	vmm_smp_map_hwid(cpu, &thart_id);
	if (thart_id != hart_id) {
		continue;
	}

	per_cpu(aclint_swi_reg, cpu) =
			(void *)(va + sizeof(u32) * i);
	nr_cpus++;
	break;
}
```

其会建立aclint的物理地址与虚拟地址va间的映射，然后每个hart的软件中断寄存器拥有4B偏移，与aclint布局对应，详见aclint手册。


当目标hart收到ipi后，进入软件中断处理，执行到``smp_ipi_handler``：

```rust
static vmm_irq_return_t smp_ipi_handler(int irq_no, void *dev) {
	/* Call core code to handle IPI */
	vmm_smp_ipi_exec();

	return VMM_IRQ_HANDLED;
}

void vmm_smp_ipi_exec(void) {
	struct smp_ipi_call ipic;
	struct smp_ipi_ctrl *ictlp = &this_cpu(ictl);

	/* Process Sync IPIs */
	while (fifo_dequeue(ictlp->sync_fifo, &ipic)) {
		if (ipic.func) {
			ipic.func(ipic.arg0, ipic.arg1, ipic.arg2);
		}
	}

	/* Signal IPI available event */
	if (!fifo_isempty(ictlp->async_fifo)) {
		vmm_completion_complete(&ictlp->async_avail);
	}
}
```


``vmm_smp_ipi_exec()``中会从当前物理cpu的队列中出队之前的ipi信息，并执行其中的函数，即由``manager_vcpu_hcpu_func``包装的``vcpu_irq_wfi_resume``。

``vcpu_irq_wfi_resume(vcpu, ...)``会清除vcpu的wfi状态，并停止wfi超时事件，并将vcpu置于``Ready``状态，可以进行调度运行。

>还有异步ipi方案

``vmm_main.c``中为异步ipi初始化：

```c
#if defined(CONFIG_SMP)
/* Initialize asynchronus inter-processor interrupts */
vmm_init_printf("asynchronus inter-processor interrupts\n");
ret = vmm_smp_async_ipi_init();
if (ret) {
	goto init_bootcpu_fail;
}
#endif


static struct vmm_cpuhp_notify smp_async_ipi_cpuhp = {
	.name = "SMP_ASYNC_IPI",
	.state = VMM_CPUHP_STATE_SMP_ASYNC_IPI,
	.startup = smp_async_ipi_startup,
};

int __init vmm_smp_async_ipi_init(void) {
	/* Setup hotplug notifier */
	return vmm_cpuhp_register(&smp_async_ipi_cpuhp, TRUE);
}

```

其中

```c
int vmm_cpuhp_register(struct vmm_cpuhp_notify *cpuhp, bool invoke_startup) {
	...
	for_each_online_cpu(cpu) {
		if (cpu == curr_cpu)
			continue;
		chps = &per_cpu(chpstate, cpu);
		vmm_read_lock_lite(&chps->lock);
		if (cpuhp->state <= chps->state) {
			// 这里对每个cpu都执行一次cpuhp_register_sync，其中会执行cpuhp->statup，即async的startup
			// 其中为每个cpu结构的ictlp->async_vcpu创建一个orphan_vcpu，用于执行ipi_main
			vmm_smp_ipi_async_call(vmm_cpumask_of(cpu), 
						cpuhp_register_sync, 
						cpuhp, NULL, NULL);
		}
		vmm_read_unlock_lite(&chps->lock);
	}
}

static void cpuhp_register_sync(void *arg1, void *arg2, void *arg3) {
	u32 cpu = vmm_smp_processor_id();
	struct vmm_cpuhp_notify *cpuhp = arg1;
	struct cpuhp_state *chps = &per_cpu(chpstate, cpu);

	vmm_read_lock_lite(&chps->lock);
	// 这里的startup即smp_async_ipi_cpuhp的startup，即smp_async_ipi_startup
	if (cpuhp->startup && (cpuhp->state <= chps->state))
		cpuhp->startup(cpuhp, cpu);
	vmm_read_unlock_lite(&chps->lock);
}
```

``smp_async_ipi_startup``会为当前cpu创建一个orphan_vcpu专门执行``smp_ipi_main``。

```c
static int smp_async_ipi_startup(struct vmm_cpuhp_notify *cpuhp, u32 cpu) {
	int rc = VMM_EFAIL;
	char vcpu_name[VMM_FIELD_NAME_SIZE];
	struct smp_ipi_ctrl *ictlp = &per_cpu(ictl, cpu);

	/* Create IPI bottom-half VCPU. (Per Host CPU) */
	vmm_snprintf(vcpu_name, sizeof(vcpu_name), "ipi/%d", cpu);
	ictlp->async_vcpu = vmm_manager_vcpu_orphan_create(vcpu_name,
						(virtual_addr_t)&smp_ipi_main,
						IPI_VCPU_STACK_SZ,
						IPI_VCPU_PRIORITY,
						IPI_VCPU_TIMESLICE,
						IPI_VCPU_DEADLINE,
						IPI_VCPU_PERIODICITY,
						vmm_cpumask_of(cpu));
	if (!ictlp->async_vcpu) {
		rc = VMM_EFAIL;
		goto fail;
	}

	/* Kick IPI orphan VCPU */
	if ((rc = vmm_manager_vcpu_kick(ictlp->async_vcpu))) {
		goto fail_free_vcpu;
	}

	return VMM_OK;

fail_free_vcpu:
	vmm_manager_vcpu_orphan_destroy(ictlp->async_vcpu);
fail:
	return rc;
}
```

``smp_ipi_main()``是一个类似生产者-消费者的处理结构。它不断尝试取出当前物理cpu队列中的ipi信息并执行

```c
static void smp_ipi_main(void) {
	struct smp_ipi_call ipic;
	struct smp_ipi_ctrl *ictlp = &this_cpu(ictl);

	while (1) {
		/* Wait for some IPI to be available */
		vmm_completion_wait(&ictlp->async_avail);

		/* Process async IPIs */
		while (fifo_dequeue(ictlp->async_fifo, &ipic)) {
			if (ipic.func) {
				ipic.func(ipic.arg0, ipic.arg1, ipic.arg2);
			}
		}
	}
}
```






# 6. 外部中断

以串口为例。

xvisor专门分配了一个vcpu给xvisor控制台mterm。

当键盘触发后，系统会收到物理外部中断，于是去读取物理plic的claim查看是哪个设备发送了中断。得知是串口后，便读取物理串口信息，并将其放入虚拟串口队列，并唤醒等待在这个队列上的vcpu。

这个vcpu会不断读取串口队列，并将内容发送给虚拟串口对应的虚拟串口设备。虚拟设备经过处理后，调用虚拟plic向guest发起虚拟外部中断。

guest收到中断后，查询虚拟plic的claim，查看是哪个设备发起中断。guest得知是虚拟串口设备后，访问虚拟设备来获取信息。

在这种情况下，键盘输入被放入串口队列，并送给与控制台相绑定的模拟设备，进而送给绑定在控制台的guest。虽然多个guest都共享这个结构，但同一时间似乎只有一个guest能绑定控制台，类似于独占这个队列。


>串口的输入约等于被独占，那么更复杂的设备呢？比如网卡等，要如何决定中断及其对应信息要发给哪个guest？多个guest如何复用这个物理网卡？













































