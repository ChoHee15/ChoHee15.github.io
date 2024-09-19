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












