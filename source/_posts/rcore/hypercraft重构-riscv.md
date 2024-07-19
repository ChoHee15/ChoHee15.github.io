---
title: hypercraft重构-riscv
date: 2024/07/10 17:16:17
tags:
  - 操作系统
  - os
categories:
  - rcore
share: "true"
---

# 00. tmp

```bash
cd /arceos-umhv/arceos

make A=../arceos-vmm run ACCEL=y LOG=info

make A=../arceos-vmm run ACCEL=y BLK=y LOG=info

make A=../arceos-vmm run ARCH=riscv64 ACCEL=y LOG=info

make A=../arceos-vmm run ARCH=riscv64 LOG=info BLK=y

qemu-system-riscv64 -m 128M -smp 1 -machine virt -bios default -kernel ../arceos-vmm/arceos-vmm_riscv64-qemu-virt.bin -device virtio-blk-device,drive=disk0 -drive id=disk0,if=none,format=raw,file=disk.img -nographic

qemu-system-riscv64 -m 128M -smp 1 -machine virt -bios default -kernel ../arceos-vmm/arceos-vmm_riscv64-qemu-virt.bin -device virtio-blk-device,drive=disk0 -drive id=disk0,if=none,format=raw,file=disk.img -nographic -d int,in_asm -D /tmp/qemu.txt

make ARCH=riscv64 A=../arceos-vmm LOG=off BLK=y MODE=debug run

make -C ../arceos/ A=$(pwd) run LOG=info ARCH=riscv64 BLK=y


make ARCH=riscv64 LOG=off BLK=y MODE=debug run
```


# 1. 神秘卡死

获取最新分支后，理论上应该能跑了。观察到代码在执行``.run()``后就一去不复返，没有exit。让qemu打印中断和指令日志后发现，程序进入虚拟化后收到了一个时钟中断，进入m mode处理，设置了mip注入交由s mode处理，此时系统正常trap到s mode，进入由hypervisor设置的stvec地址即``_guest_exit``。然而在``_guest_exit``运行到一半时，s mode的时钟中断又被触发，并伴随page_fault，此后就是无限的时钟中断和page_fault。

首先可以明确的是，在训练营版本的riscv-hypercraft设计下，时钟中断完全由hypercraft管理，即hypercraft调用``set_timer``来设置定时器，hypercraft来处理``timer interrupt``；arceos不应该设置定时器。

arceos在不涉及虚拟化的情况下，打开irq feature，会设置一个目标时间为0的时钟。

```rust
// arceos/modules/axhal/src/platform/riscv64_qemu_virt/time.rs
pub(super) fn init_percpu() {
    #[cfg(feature = "irq")]
    sbi_rt::set_timer(0);
}
```

理论上，它会立即产生一个时钟中断（此前中断使能已经打开），然后跳转到stvec，进入arceos的中断处理，其中时钟中断部分在``rust_main()``下方的``init_interrupt()``中被注册。

```rust
#[cfg(feature = "irq")]
fn init_interrupt() {
	...
    fn update_timer() {
        let now_ns = axhal::time::current_time_nanos();
        ...
        axhal::time::set_oneshot_timer(deadline);
    }

    axhal::irq::register_handler(TIMER_IRQ_NUM, || {
        update_timer();
        #[cfg(feature = "multitask")]
        axtask::on_timer_tick();
    });

    // Enable IRQs before starting app
    axhal::arch::enable_irqs();
}
```

``update_timer()``会被注册为时钟中断的处理函数，它会设置下一个时间片的定时器。等到下一个时钟中断到来，重复这一过程。

然而如果在这种打开irq的情况下运行hypercraft，问题就来了。在进入vm前，一切都正常：arceos设置时钟，收到中断后进入stvec，即arceos的irq_hanlder，设置下一个时钟。然而进入vm后，由于hypercraft修改了stvec为``_guest_exit``的地址，收到时钟中断后进入的stvec不再是arceos的irq_hanlder，而是``_guest_exit``；因此不会有设置下一个时钟的动作来更新``MTIMECMP``，``MTIME``始终大于``MTIMECMP``，时钟中断会不断触发，程序继续不断进入``_guest_exit``，就卡住了。

因此目前的设计导致riscv-hypercraft不能接受来自arceos的定时器。riscv-hypercraft必须自己设置定时器，然后进入vm，时钟中断到来后退出vm，回到hypercraft进行处理。

>然而这又有一些问题，假如riscv-hypercraft根据guest os的要求设置了一个定时器，然后进入vm，又因为其他原因退回了host，此时时钟中断到来了，由于目前在host，stvec还指向arceos的中断处理程序，那么此时钟中断就会交由arceos处理，hypercraft没有获知此中断，guest os就丢失了一个时钟中断？
>这是怎么处理的？

所以，对于riscv来说，不应该开启irq，把这部分feature关了就能正常运行了。


# 2. 修改引用

把对arceos的本地相对路径引用改成github url。

像这样：

```toml
axvcpu = { git = "https://github.com/arceos-hypervisor/axvcpu.git", branch = "main" }

# System independent crates provided by ArceOS, these crates could be imported by remote url. 
# axerrno = { path = "../../arceos/crates/axerrno" }
# spinlock = { path = "../../arceos/crates/spinlock" }
# memory_addr = { path = "../../arceos/crates/memory_addr" }
# page_table = { path = "../../arceos/crates/page_table" }
# page_table_entry = { path = "../../arceos/crates/page_table_entry" }
# percpu = { path = "../../arceos/crates/percpu" }
axerrno = { git = "https://github.com/arceos-hypervisor/arceos.git", rev = "b42df3f" }
spinlock = { git = "https://github.com/arceos-hypervisor/arceos.git", rev = "b42df3f" }
memory_addr = { git = "https://github.com/arceos-hypervisor/arceos.git", rev = "b42df3f" }
page_table = { git = "https://github.com/arceos-hypervisor/arceos.git", rev = "b42df3f" }
page_table_entry = { git = "https://github.com/arceos-hypervisor/arceos.git", rev = "b42df3f" }
percpu = { git = "https://github.com/arceos-hypervisor/arceos.git", rev = "b42df3f" }
```



# 3. 取消arceos的submodule引用

并不是改了链接就好了，最终目的是在arceos-vmm里就能直接make运行，不需要本地有arceos。我没理解到位，很呆。

除了修改各种``.toml``，还需要把Makefile扒过来。这相当于只有arceos-vmm这个一个应用，所以不需要设置APP参数之类的，我直接写``A ?= $(CURDIR)``了。然后``make clean``也改了一下，因为没有其他应用了，所以直接删除``target``文件夹和``.bin/.elf``就行。

还有就是由于没有``crates``/``modules``，所以会报``ls: No such file or directory``。来源在``cargo.mk``，这里的``all_packages``会尝试列出所有包，我们没有这些文件所以会报错。

```makefile
# arceos-vmm/scripts/make/cargo.mk
all_packages := \
  $(shell ls $(CURDIR)/crates) \
  $(shell ls $(CURDIR)/modules) \
  axfeat arceos_api axstd axlibc
```

但是好像也不影响什么，暂时就不动了。






# 1. main

``arceos-vmm``的``main``：

```rust
#[percpu::def_percpu]
pub static mut AXVM_PER_CPU: AxVMPerCpu<AxvmHalImpl> = AxVMPerCpu::new_uninit();

fn main() {
    println!("Starting virtualization...");
    info!("Hardware support: {:?}", axvm::has_hardware_support());
    let percpu = unsafe { AXVM_PER_CPU.current_ref_mut_raw() };
    percpu.init(0).expect("Failed to initialize percpu state");
    percpu
        .hardware_enable()
        .expect("Failed to enable virtualization");

    let gpm = setup_gpm().expect("Failed to set guest physical memory set");
    debug!("{:#x?}", gpm);

    let config = AxVMConfig {
        cpu_count: 1,
        cpu_config: AxVCpuConfig {
            arch_config: AxArchVCpuConfig {
                setup_config: (),
                create_config: (),
            },
            ap_entry: GUEST_ENTRY,
            bsp_entry: GUEST_ENTRY,
        },
        // gpm: gpm.nest_page_table_root(),
        // gpm : 0.into(),
    };

    let vm = AxVM::<AxvmHalImpl>::new(config, 0, gpm.nest_page_table_root()).expect("Failed to create VM");
    info!("Boot VM...");
    vm.boot().unwrap();
    panic!("VM boot failed")
}
```

（看上去非常简洁非常解耦

``AxvmPerCpu``来自``axvm/src/lib.rs``，它使用``pub type``来定义为来自某架构的``percpu::AxVMPerCpu<arch::AxVMArchPerCpuImpl<H>>``，arch的``mod.rs``中对相应架构进行了选择；例如：

```rust
cfg_if::cfg_if! {
    if #[cfg(target_arch = "riscv64")] {
        mod riscv64;
        pub use self::riscv64::*;
    }
}
```

而riscv64中的``mod.rs``进一步确定了使用什么结构，如：

```rust
// crates/axvm/src/arch/riscv64/mod.rs
pub use self::PerCpu as AxVMArchPerCpuImpl;
use crate::percpu::AxVMArchPerCpu;
...
pub struct PerCpu<H: AxVMHal> {
    _marker: core::marker::PhantomData<H>,
}
```

相当于确定了``AxvmPerCpu``为``percpu::AxVMPerCpu<PerCpu<H>>``

~~``perCpu``创建vcpu通过``AxvmVcpu::new(&self.arch, entry, npt_root)``，需要将``PerCpu``自身的``arch: ArchPerCpuState<H>``传入。``AxvmVcpu``由相应架构通过``pub use self::vcpu::VmxVcpu as AxvmVcpu;``提供，例如``crates/axvm/src/arch/x86_64/vmx/mod.rs``。~~

>~~遇到第一个问题就很怪，riscv需要``hardware_enable()``这种操作吗？~~





