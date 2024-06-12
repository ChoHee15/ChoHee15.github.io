---
title: rcore-arceos-hypervisor虚拟化抖动入坑
date: 2024/05/21 18:26:51
tags:
  - rcore
  - rust
  - os
categories:
  - rcore
share: "true"
---
# 0. tmp

```bash
make ARCH=riscv64 A=apps/hv HV=y SMP=2 LOG=debug MODE=debug debug
```

# 1. 基础设施

- [x] vsc debug
- [ ] vsc-ra跳转/识别

为了方便调试，建议在hypervisor``main``中添加断言，当boot_hart不为0号hart时直接退出。跑通后再支持随机hart的启动。

# 2. arceos SMP启动过程

好东西-->https://mermaid.live/


[![](https://mermaid.ink/img/pako:eNqVU8tuwyAQ_BVre3UiG9t5UKmHttee2lPjyiI2CZYMuBiUpFH-veC8bCVKUyTEDjvs7C6whVwWFDAsKrnKGVHa-3hORardbMx8qUjNPLJmpPJS4R3GycFlMVTN2cHCGV1rqoT38G2kfnzZL54yjc6o0Grz1SHHf5GzhuZSFKR3jIrCJncll7qfCpo12tZzjpHltenKJ7NWiZNS_EtoLqXuSwXXC8naBLqS0S3iPUkwOlstyo7f3VOHw4LB4ImFbpeF1lR7Ew0GKTTzEmPWCjm5FBw1av2RM-NDiNiB5AASB6gL33sPyghdcnqlO1U57zXHDXXrWbj-33W92UVcHl674-YiWrdDyrWFh3vAHWDo-ODdBB84VTapwv6KraOloBm1tQK2ZkEXxFS2eanYWSoxWr5vRA5YK0N9UNIsGeAFqRqLTF0QTV9LYkvgp92aiE8p-fGIhYC3sAYcBsFwjFAcjdAoQUmIkA8bwDEaToNknEwn4TRA0Sja-fDTBgiGE7s9DuIJmqARiiIfaFFqqd72n7r927tf3ds1zg?type=png)](https://mermaid.live/edit#pako:eNqVU8tuwyAQ_BVre3UiG9t5UKmHttee2lPjyiI2CZYMuBiUpFH-veC8bCVKUyTEDjvs7C6whVwWFDAsKrnKGVHa-3hORardbMx8qUjNPLJmpPJS4R3GycFlMVTN2cHCGV1rqoT38G2kfnzZL54yjc6o0Grz1SHHf5GzhuZSFKR3jIrCJncll7qfCpo12tZzjpHltenKJ7NWiZNS_EtoLqXuSwXXC8naBLqS0S3iPUkwOlstyo7f3VOHw4LB4ImFbpeF1lR7Ew0GKTTzEmPWCjm5FBw1av2RM-NDiNiB5AASB6gL33sPyghdcnqlO1U57zXHDXXrWbj-33W92UVcHl674-YiWrdDyrWFh3vAHWDo-ODdBB84VTapwv6KraOloBm1tQK2ZkEXxFS2eanYWSoxWr5vRA5YK0N9UNIsGeAFqRqLTF0QTV9LYkvgp92aiE8p-fGIhYC3sAYcBsFwjFAcjdAoQUmIkA8bwDEaToNknEwn4TRA0Sja-fDTBgiGE7s9DuIJmqARiiIfaFFqqd72n7r927tf3ds1zg)

>- [x] 问题：wfi以后怎么办？
>
>从核在走完启动流程后会wfi，然而不能对已经wfi的hart进行``sbi::hart_start``，会报告``SBI_ERR_ALREADY_AVAILABLE``已经启动。那么如何让从核到指定位置继续运行？
>
>答案是别管，直接侵入arceos部分，直接在wfi-loop前添加函数，让其跳转到新的入口，不进入wfi


# 3. hypervisor虚拟化

## a. cpu虚拟化

arceos启动完成后，跳入hypervisor的``main(hartid)``。hypervisor依次做以下工作：

```rust
// boot cpu
PerCpu::<HyperCraftHalImpl>::init(0, 0x4000);
```

首先为物理cpu创建结构。它会根据cpu数量，获取一块固定大小的内存，并将其按照cpu数量等分，形成类似于一个数组的结构。分配完成后，“数组”的首地址被存入``PER_CPU_BASE``中。

“数组”下标为``cpi_id``的元素存放着对应cpu的信息。这些信息包括cpu_id，栈地址和此物理cpu绑定的vcpu队列等（``marker: core::marker::PhantomData<H>``不懂是干啥的）。

这个结构配合``tp``寄存器，可以在执行流中找到当前的物理cpu信息。在每个物理cpu上执行一次``setup_this_cpu``，就可以让每个cpu的``tp``保存各自的``PerCpu``结构位置。那么在执行流的任意位置使用``this_cpu()``，就能通过``tp``获得当前物理cpu的``PerCpu``结构，进而得到当前运行在几号物理cpu上/此物理cpu绑定哪些vcpu等信息。

然后，hypervisor解析设备树文件，并据此设置两阶段内存转换页表。这部分内容放到内存虚拟化中详细说明。

```rust
// get current percpu
let pcpu = PerCpu::<HyperCraftHalImpl>::this_cpu();

// create vcpu
let gpt = setup_gpm(0x9000_0000).unwrap();
let vcpu = pcpu.create_vcpu(0, 0x9020_0000).unwrap();
```

然后，根据PerCpu创建vcpu，它实际上是在PerCpu的绑定队列里添加``vcpu_id``，然后根据vcpu的id和入口地址，创建一个新vcpu结构。

```rust
pub struct VCpu<H: HyperCraftHal> {
    vcpu_id: usize,
    regs: VmCpuRegisters,
    // gpt: G,
    // pub guest: Arc<Guest>,
    marker: PhantomData<H>,
}
```

vcpu结构最主要的功能是保存V模式/非V模式的寄存器。进入/退出guest时，会对``regs``进行读取/保存。``VCpu<H>::new(...)``会设置一些虚拟化相关寄存器，然后设置``a0/a1/sepc``，构造一个上下文环境，使得在``sret``后可以回到``entry``执行；在hypervisor的main中，这个entry实际上就是linux的入口``0x9020_0000``（因为arceos的启动命令默认会把linux.bin放到这个位置，详情可在make run时使用``-nB``参数打印所有命令查看）。而``a0/a1``需要保存启动hart_id和设备树文件地址，这是riscv linux的规定，它需要接受这些信息才能启动。

>todo: riscv linux 启动文档

回到main中，创建vcpus，为vcpus添加vcpu，然后根据vcpus和页表创建vm。

```rust
let mut vcpus = VmCpus::new();
// add vcpu into vm
vcpus.add_vcpu(vcpu).unwrap();
let mut vm: VM<HyperCraftHalImpl, GuestPageTable> = VM::new(vcpus, gpt).unwrap();
```

vcpus只是一个聚合多个vcpu的结构，为vm提供一层抽象。而vm在组合了vcpus和页表后，就具备了运行虚拟机的能力。

```rust
vm.init_vcpu(0);

// vm run
info!("vm run cpu{}", hart_id);
vm.run(0);
```

至此，可以启动vm运行guest了。``init_vcpu``会从``vcpus``中获取对应id的vcpu，然后设置其``regs``中的``hgatp``为两阶段转换页表地址，并直接通过``csrw hgatp``写入当前物理cpu的``hgatp``。

>todo: hgatp是每个cpu都有吗???

然后``vm.run``会调用vcpu的``run()``，实际上最后会来到``_run_guest(regs)``。``_run_guest(regs)``是一段汇编，它会保存hypervisor的寄存器到内存（即``regs``内），并替换为``regs``中guest的寄存器值，完成到guest的上下文切换，然后``sret``到之前构造好的linux入口环境中。

此处还会将``stvec``设置为``_guest_exit``，同样为一段汇编，就在``_run_guest``下方。``_guest_exit``类似于做``_run_guest``相反的操作，保存guest上下文，回到hypervisor中。设置``stvec``为``_guest_exit``，可以让linux在``ecall``时，跳转到``_guest_exit``，进而回到host即hypervisor中。

当从``_guest_exit``退出guest后，会回到``vcpu.run()``中，根据``scause``分析原因并进一步退回``vm.run(..)``。``vm.run(...)``根据不同的原因，执行不同的操作，将结果写回``vcpu``的``regs``中，在下一轮``_run_guest``时，这些结果会被linux获知。

在linux kernel眼中，自己就是执行了``ecall``，然后得到了结果而已。它并不清楚``ecall``召唤的原来是hypervisor，也不知道自己的寄存器值已经被换出去一轮了，就好像自己真的在物理cpu上运行一样。而hypervisor的任务就是模拟一个硬件环境，让guest不得而知，自己却能捕捉guest的行为。







## b. 内存虚拟化

两阶段地址翻译：GVA --> GPA --> HPA

其中guest os维护GVA --> GPA这部分转换，hypervisor需要维护GPA --> HPA这部分转换。

首先，arceos在页表初始化时会为虚拟化分配空间，总体为2GB：

```rust
unsafe fn init_boot_page_table() {
    #[cfg(feature = "hv")]
    {
        // 0x0000_0000..0x8000_0000, VRWX_GAD, 2G block
        BOOT_PT_SV39[0] = 0xef;
        BOOT_PT_SV39[1] = (0x40000 << 10) | 0xef;
    }
    // 0x8000_0000..0xc000_0000, VRWX_GAD, 1G block
    BOOT_PT_SV39[2] = (0x80000 << 10) | 0xef;
    // 0xffff_ffc0_8000_0000..0xffff_ffc0_c000_0000, VRWX_GAD, 1G block
    BOOT_PT_SV39[0x102] = (0x80000 << 10) | 0xef;
}
```

然后，hypervisor会从dtb读取地址布局，并建立GPA --> HPA的等值映射，详见``pub fn setup_gpm(dtb: usize) -> Result<GuestPageTable>``

大致布局如下：

![hv_f.png](_note_assets_/hv_f.png)

缺了mmio的一些映射，大体上应该差不多



## c. 中断虚拟化

以guest os一次设定计时器的操作为例：

1. arceos初始化时设置了虚拟化拓展环境，位于``crates/hypercraft/src/arch/riscv/mod.rs``，其中设置了hideleg，使所有中断都委托给了VS模式，也就是guest os所处的模式
2. guest os调用``sbi::set_timer``，设置定时器
3. ecall触发vmexit，此行为被捕获，hypervisor得知guest os调用了``sbi::set_timer``
4. 于是hypervisor调用``sbi::set_timer``，设置了定时器，并使能时钟中断
```rust
HyperCallMsg::SetTimer(timer) => {
    sbi_rt::set_timer(timer as u64);
    // Clear guest timer interrupt
    CSR.hvip.read_and_clear_bits(
        traps::interrupt::VIRTUAL_SUPERVISOR_TIMER,
    );
    //  Enable host timer interrupt
    CSR.sie
        .read_and_set_bits(traps::interrupt::SUPERVISOR_TIMER);
}
```
5. 一段时间后，定时器生效
6. 由于之前设置了hideleg，所有中断都委托给了VS模式；于是guest os触发了时钟中断，并vmexit，此时guest os本身并不知情。
7. 时钟中断被捕获，hypervisor得知guest os遇到了时钟中断
8. 于是hypervisor通过``hvip``向guest os注入中断，并清除中断使能
```rust
VmExitInfo::TimerInterruptEmulation => {
    // debug!("timer irq emulation");
    // Enable guest timer interrupt
    CSR.hvip
        .read_and_set_bits(traps::interrupt::VIRTUAL_SUPERVISOR_TIMER);
    // Clear host timer interrupt
    CSR.sie
        .read_and_clear_bits(traps::interrupt::SUPERVISOR_TIMER);
}
```
9. guest os继续运行，获知发生时钟中断

以上达成的效果是：guest os设置定时器，并在一段时间后获知了时钟中断的发生。

由于替换了``stvec``，因此guest os的异常/中断行为被hypervisor拦截/管控，hypervisor负责将其通知给guest os，称为中断注入。

>todo:
>对吗？


# 4. Linux SMP启动过程


[smp多核启动（riscv架构） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/653590588)



# 5. hypervisor多核

## a. 多核dtb

首先要让guest linux感知到有多核硬件的存在，这部分由设备树文件表示。正常情况下，设备信息应该是由firmware告知操作系统的，而我们现在只需要虚构一个合理的多核硬件平台就行，让guest linux以为真的存在这些硬件。

``/arceos/apps/hv/guest/linux``下已经有了``linux.dtb``和对应的``linux.dts``了，对``.dts``进行修改，给予一个双核心，注意对原文件进行备份。

```
// dts
cpus {
		#address-cells = <0x01>;
		#size-cells = <0x00>;
		timebase-frequency = <0x989680>;

		cpu@0 {
			phandle = <0x01>;
			device_type = "cpu";
			reg = <0x00>;
			status = "okay";
			compatible = "riscv";
			riscv,isa = "rv64ima";
			mmu-type = "riscv,sv39";

			interrupt-controller {
				#interrupt-cells = <0x01>;
				interrupt-controller;
				compatible = "riscv,cpu-intc";
				phandle = <0x02>;
			};
		};

		cpu@1 {
            phandle = <0x05>;
            device_type = "cpu";
            reg = <0x01>;
            status = "okay";
            compatible = "riscv";
            riscv,isa = "rv64ima";
            mmu-type = "riscv,sv39";

            interrupt-controller {
                #interrupt-cells = <0x01>;
                interrupt-controller;
                compatible = "riscv,cpu-intc";
                phandle = <0x06>;
            };
        };

		cpu-map {

			cluster0 {

				core0 {
					cpu = <0x01>;
				};

				core1 {
                	cpu = <0x05>;
           		};
			};
		};
	};
```

使用`dtc`（Device Tree Compiler）将 DTS 文件编译为 DTB 文件。命令格式如下：

```bash
dtc -I dts -O dtb -o output.dtb input.dts

dtc -I dts -O dtb -o linux_hart_x2.dtb linux_hart_x2.dts
```


## b. sbi_call: hart_start

修改``dtb``后运行应该会报错：

![sbi_error_start.png](_note_assets_/sbi_error_start.png)

```bash
[  0.157032 0 arceos_hv:66] vm run cpu0
[  0.193893 0 hypercraft::arch::vm:221] GetSepcificationVersion: 3
[  0.278965 0 hypercraft::arch::sbi:80] args: [1, 2418020454, 2533225328, 0, 0, 0, 0, 4739917]
[  0.280327 0 hypercraft::arch::sbi:81] args[7]: 0x48534d
[  0.281224 0 hypercraft::arch::sbi:82] EID_RFENCE: 0x52464e43
[  0.282422 0 axruntime::lang_items:5] panicked at 'explicit panic', /arceos_2024/arceos/crates/hypercraft/src/arch/riscv/vm.rs:96:25
[  0.284209 0 axhal::platform::qemu_virt_riscv::misc:3] Shutting down...
```

位置在``hypercraft::arch::sbi:80``的80行，这意味着遇到了未知的sbi_call。

此时应该回到sbi手册，据手册所说：

```
• a7 encodes the SBI extension ID (EID),
• a6 encodes the SBI function ID (FID) for a given extension ID encoded in a7 for any SBI extension defined in or after SBI v0.2.
```

根据以上报错信息，得知``EID = a7 = 0x48533d = 4739917``，``FID = a6 = 0``，查找手册得知对应的内容为``Chapter 9. Hart State Management Extension (EID #0x48534D "HSM")``中的小节``9.1. Function: HART start (FID #0)``。

签名为：
```c
struct sbiret sbi_hart_start(
	unsigned long hartid,
	unsigned long start_addr,
	unsigned long opaque
)
```

此函数要求目标hart以supervisor模式在``start_addr``地址开始运行，并且寄存器会被设置为以下格式（这都是从sbi手册上抄的）：

| Register Name | Register Value   |
| ------------- | ---------------- |
| satp          | 0                |
| sstatus.SIE   | 0                |
| a0            | hartid           |
| a1            | opaque parameter |

其余寄存器不做定义。

可以分析得出，guest linux在知晓第二个核心存在后，就调用``sbi_hart_start(1, 2418020454, 2533225328)``来启动第二个核心，让其从指定地址开始运行。

hypervisor捕获了这一行为，但暂时还没有添加此sbi类型。因此我们需要补齐这部分功能，让hypervisor正确识别``sbi_hart_start``，并做相应处理。

补充这部分功能后，应该能正常进入guest linux了，不过此时多核肯定还没有完成，可以看到开机信息中显示启动cpu失败：

```
[    0.046483] smp: Bringing up secondary CPUs ...
[    0.052778] CPU1: failed to start
[    0.054363] smp: Brought up 1 node, 1 CPU
```

毕竟我们还没有真的去运行``sbi_hart_start``中指定位置的代码。guest linux发出了启动hart的请求，却没有收到响应，那么它自然会认为此核心异常，并报告失败。

因此，我们应该启动一个vcpu，遵照上面的状态，让其从指定地址开始运行，给guest linux提供“真的有核心在听我话”的错觉。


## c. secondary运行

最终想要的效果是，当收到主核的``sbi::hart_start``后，根据参数设置寄存器初始状态，然后让第二个hart去运行``vm.run(1)``。

我们需要准备一个新的入口地址``hv_secondary_main``，让第二个hart跳转到此处，好进行下一部分设置。就目前的项目情况看，可以直接修改arceos的``rust_main_secondary``，添加``hv_secondary_main``extern函数调用，让hart直接跳转进去。

在第二个hart进入``hv_secondary_main``后，需要将``PerCpu``结构设置到tp寄存器，并向vm添加新的vcpu，然后进行``vm.init_cpu(1) & vm.run(1)``。

这涉及到一些同步问题。首先，要向vm添加新的vcpu，必须存在vm。这要求主核创建vm以后，从核才调用``vm.add_vcpu(...)``。其次，主核需要等待所有从核都向vm添加vcpu后，才能开始运行``vm.run(0)``，否则开始运行时vcpu数量是缺失的，可能会出问题。

最后，从核需要等待主核调用``sbi::hart_start``，获取目标地址等参数后，才能为vcpu设置对应寄存器，才能正式运行``vm.run(...)``。因此需要一个状态表示vcpu是否可以运行，并且让主核心来控制从核的这个状态。框架代码中包含了``VmCpuStatus``，可以修改后使用：

```rust
pub enum VmCpuStatus {
    /// The vCPU is not powered on.
    #[default]
    PoweredOff,
    /// The vCPU is available to be run.
    Runnable,
    /// The vCPU has benn claimed exclusively for running on a (physical) CPU.
    Running,
}
```

vcpu在运行前需要检查这个状态，来决定是否可以运行，同时主核心应该在处理``sbi::hart_start``时，根据参数设置vcpu寄存器，完成后将vcpu状态设置为可运行。

总体流程差不多是下面这样。目的就是让guest os产生“自己直接运行在硬件上”的错觉。

[![](https://mermaid.ink/img/pako:eNp1VF1v2jAU_SuWn0AKEUmcz4dKpau0SStIK93DSIXcxOBIxEGOQ8cQ_32-wQkN65CCOL7nnss9J8kJZ1XOcII3u-o941QqtJylIhVIf-rmbSvpnqNnllUip_K4Bsal2BKcFT-s675a0kK8dt0twR2N3mmhCrFF19OfTyiTVLHxeMD1Vik-lDbN8_Uh2zcjZ5zioRr5TO3Hy3x-P_v-eKPmX9RkIzqhrqRpiom839KZ2JO72jXInWjgGeC1JWIQgZIPfSABV-_QbLFY3pjDW3OMJdfZfdldZZJpF7Qdgy2599mWvcloOKVza7g9J9ftp7c2crDm4X758BXVb0WSgOC6Vvp75FjItu1_GoJVzRSCMQ6SbFsPqyHMgpqtWSCkmnrUp3KrFcHwxXz5bf7yiIZ_8T8BcUf7zk0-3IVIuAmIe1Ay-XDIh_sG-AACAwIAoQEhgKhXbwN3-8A_iAMRwr_Aj7nDhS1cMqnzzfXjcwJOihVnJUtxon_mbEObnUpxKs6aShtVPR9FhhMlG2ZhWTVbjpMN3dUaNftc3wlfCqrvprI_3VPxq6rKrkVDnJzwb5y4vm87JHYdEnqRE3iEWPioj6c20diJvSj0QxLG0dnCf1qFqR37XhyQyI0JIVHk6w6WF6qST5fnv30NnP8CIvAoow?type=png)](https://mermaid.live/edit#pako:eNp1VF1v2jAU_SuWn0AKEUmcz4dKpau0SStIK93DSIXcxOBIxEGOQ8cQ_32-wQkN65CCOL7nnss9J8kJZ1XOcII3u-o941QqtJylIhVIf-rmbSvpnqNnllUip_K4Bsal2BKcFT-s675a0kK8dt0twR2N3mmhCrFF19OfTyiTVLHxeMD1Vik-lDbN8_Uh2zcjZ5zioRr5TO3Hy3x-P_v-eKPmX9RkIzqhrqRpiom839KZ2JO72jXInWjgGeC1JWIQgZIPfSABV-_QbLFY3pjDW3OMJdfZfdldZZJpF7Qdgy2599mWvcloOKVza7g9J9ftp7c2crDm4X758BXVb0WSgOC6Vvp75FjItu1_GoJVzRSCMQ6SbFsPqyHMgpqtWSCkmnrUp3KrFcHwxXz5bf7yiIZ_8T8BcUf7zk0-3IVIuAmIe1Ay-XDIh_sG-AACAwIAoQEhgKhXbwN3-8A_iAMRwr_Aj7nDhS1cMqnzzfXjcwJOihVnJUtxon_mbEObnUpxKs6aShtVPR9FhhMlG2ZhWTVbjpMN3dUaNftc3wlfCqrvprI_3VPxq6rKrkVDnJzwb5y4vm87JHYdEnqRE3iEWPioj6c20diJvSj0QxLG0dnCf1qFqR37XhyQyI0JIVHk6w6WF6qST5fnv30NnP8CIvAoow)








## d. sbi_call: send_ipi

下面继续运行的话，会出现新的未知sbi_call。

![sbi_error_ipi.png](_note_assets_/sbi_error_ipi.png)

```bash
[  0.339238 0 hypercraft::arch::sbi:88] args: [1, 1, 0, 0, 0, 0, 0, 7557193]
[  0.340591 0 hypercraft::arch::sbi:89] args[7]: 0x735049
[  0.341543 0 hypercraft::arch::sbi:90] EID_RFENCE: 0x52464e43
[  0.343278 0 axruntime::lang_items:5] panicked at 'explicit panic', /arceos_2024S/arceos/crates/hypercraft/src/arch/riscv/vm.rs:128:25
[  0.346961 0 axhal::platform::qemu_virt_riscv::misc:3] Shutting down...
```

这是guest linux的主从核同步过程，主核向从核发送ipi让其跳出wfi。详见linux smp启动过程。

>TODO: linux smp

仿照``hart_start``，在hypervisor中进行处理，真正调用``send_ipi``即可。

根据sbi手册所述，发送核间中断会在目标hart触发一个``supervisor software interrupts``，这会表现为``Unhandled trap: Interrupt(SupervisorSoft)``，需要我们补充相关信息，仿照``vcpu.rs``和``vm.rs``中异常处理的相关内容即可。



## e. 中断注入

收到ipi的hart触发``supervisor software interrupts``后，会跳转中断向量，即在``_run_guest``中被修改为的``_guest_exit``，此时中断被hypervisor捕获，guest os并不知情。

因此我们需要将软件中断注入guest中。

>TODO
>hvip???



## f. sbi_call: remote fence

后面运行还会触发``RemoteSFenceVMA_ASID``，按照sbi手册内容，仿照之前的内容进行添加即可。



## g. success

![success_2cpu.png](_note_assets_/success_2cpu.png)

```bash
[    0.057543] smp: Bringing up secondary CPUs ...
[    0.061161] cacheinfo: Unable to detect cache hierarchy for CPU 1
[    0.066028] smp: Brought up 1 node, 2 CPUs
```

![success_2cpu_cpuinfo.png](_note_assets_/success_2cpu_cpuinfo.png)



# 6. 优化/完善

>TODO
>数据竞争风险？上锁？
>中断注入？

~~随机hart启动，启动hart不为0号的情况下~~，会出现模拟外部中断``irq == 0``？在debug_vm_pgtable后面，不知道有没有关系。

```bash
[    1.067972] Key type dns_resolver registered
[    1.101465] debug_vm_pgtable: [debug_vm_pgtable         ]: Validating architecture page table helpers
[  1.420814 0 axruntime::lang_items:5] panicked at 'assertion failed: irq != 0', /arceos_2024S/arceos/crates/hypercraft/src/arch/riscv/vm.rs:250:9
[  1.422409 0 axhal::platform::qemu_virt_riscv::misc:3] Shutting down...
```

所以plic这块是什么情况？


# 7. 进阶任务：多绑定

在一个物理核心上运行多个vcpu。这需要一个切换机制，可以简单选择时钟中断为切换信号。当收到时钟中断后，可以让vcpu/vm以一个特殊``VMExit``信息(???)退出，在hypervisor``main/secondary_main``中，通过``perCpu``的``vcpu_queue``获取其他绑定到此hart的vcpu，对其进行``vm.run(...)``。这要求``vm.run(...)``能够在退出前保存``gprs``，并在下次运行时恢复现场。



```rust
let pcpu = PerCpu::<HyperCraftHalImpl>::this_cpu();
loop {
    let next_vcpu_id = pcpu.next_vcpu();
    vm.init_vcpu(next_vcpu_id);
    vm.run(next_vcpu_id);
}
```

这种情况下，中断要面临的问题更复杂一些。

假设目前只有一个物理cpu，上面要运行两个vcpu。vcpu0作为主核启动了guest linux，在启动过程中，主核会向从核发送ipi，即向vcpu1发送ipi。然而，vcpu0/1都对应同一个物理核，我们不能像单绑定一样，直接对目标核心``sbi::send_ipi``，这等价于自己对自己发ipi。

因此hypervisor在捕获``sbi::send_ipi``行为后，要么立即切换vcpu1运行，要么将此行为缓存下来，等vcpu1运行时再发送？

> TODO:
> 注入软件中断？

理论上，所有涉及hart_id的sbi_call都需要进行类似的处理？？？







# 00. /bin

```mermaid
flowchart TB
	
	subgraph axhal 
	h1[mod.rs::extern #quot;C#quot; rust_entry]
	h2[mp.rs::fn start_secondary_cpu]
	h3[boot.rs::extern #quot;C#quot; fn _start_secondary]
	h4[mod.rs::extern #quot;C#quot; rust_entry_secondary]
	end

	h1-->r1
	h2-->h3
	h3-->h4

	subgraph axruntime
	r1[lib.rs::extern #quot;C#quot; rust_main]-->m1
	m1[mp.rs::fn start_secondary_cpus]-->h2
	end
	
	
	
```


```mermaid
flowchart TB

    subgraph axhal

        subgraph mod.rs

        h1[extern #quot;C#quot; rust_entry]

        h4[extern #quot;C#quot; rust_entry_secondary]

        end

  

        subgraph mp.rs

        h2[start_secondary_cpu]

        h5[rust_main_secondary]

        end

  

        subgraph boot.rs

        h0[extern #quot;C#quot; _start]

        h3[extern #quot;C#quot; _start_secondary]

        end

  

        he[wfi]

    end

  

    h0-->h1

    h1-->r1

    h2--"sbi::hart_start"-->h3

    h3-->h4

    h4-->h5

    h5-->he

  

    subgraph axruntime

        subgraph lib.rs

            r1[extern #quot;C#quot; rust_main]

        end

  

        subgraph mp_.rs

            m1[start_secondary_cpus]

        end

    end

  

    r1-->m1

    m1-->h2
    
```














