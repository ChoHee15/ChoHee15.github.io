---
title: rcore-arceos VSCode调试方法
date: 2024/05/20 22:37:32
tags:
  - rust
  - rcore
  - os
categories:
  - rcore
---

# 1. Makefile

于arceos的Makefile可以发现它提供了一键debug

```makefile
debug: build
    $(call run_qemu_debug) &
    sleep 1
    $(GDB) $(OUT_ELF) \
      -ex 'target remote localhost:1234' \
      -ex 'b rust_entry' \
      -ex 'continue' \
      -ex 'disp /16i $$pc'
```

于是只需run改debug，就可以直接在终端使用gdb调试

```bash
make ARCH=riscv64 A=apps/hv HV=y LOG=info debug
```

然而默认是release编译，不带debug信息。如Makefile头部所写：

```makefile
# Arguments
ARCH ?= x86_64
SMP ?= 1

# ↓ 默认为release
MODE ?= release
# ↑ 默认为release

LOG ?= warn
```

这里的``MODE``会影响到``/script/make/cargo.mk``中的cargo build参数，并由此决定是否使用``cargo rustc --release``，``--release``使得编译结果不带有调试信息。此时指定``MODE``为debug即可带有调试信息进行编译。

```bash
make ARCH=riscv64 A=apps/hv HV=y LOG=info MODE=debug debug
```

Makefile中``debug``目标描述的行为其实是先运行qemu，这个qemu使用了``-s -S``参数来启动一个gdb server，此时qemu就会等待gdb client连接，等到连接后再继续执行；然后再启动gdb，读取elf文件获得debug信息，使用``target remote localhost:1234``来连接gdb server。连接成功后，qemu继续执行，gdb也可以通过此连接沟通qemu来调试qemu上的程序。






# 2. VSCode

然而如果要一直在终端gdb，随着debug次数增加，想必人可能会癫掉。

如jyy所说，在前期花费一点时间折腾基础设施/工具，能够节约大量时间。

我们完全可以利用更现代一点的ide/editor，来帮助调试。

以vscode为例，我们主要希望借用vscode的gui来调试。如很多其他编辑器一样，vscode也只是“调用”gdb而已，只是它可以将信息更友好地显示出来；我们要做的就是告诉vscode，调用哪个gdb，调试文件在哪，gdb server的地址在哪……

这些内容在``.vscode/launch.json``配置，按下F5进行调试的行为就在此配置，若F5时没有发现``launch.json``则会提示生成。

下面是我的配置：

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
            "program": "${workspaceFolder}/apps/hv/hv_qemu-virt-riscv.elf",
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
                    "ignoreFailures": true

                },
                {
                    "description": "将反汇编风格设置为 Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ],
            // arceos makefile中默认用的gdb-multiarch，由于我这用gdb-multiarch有点问题所以随便找了个能用的其他版本，可以按需切换
            "miDebuggerPath": "riscv64-linux-gdb",
            // qemu启动gdb server的地址，通常为localhost:1234
            "miDebuggerServerAddress": "localhost:1234",
            // 如果出现蜜汁问题可以打开gdb的log，在调试控制台查看更详细的信息
            // "logging": { "engineLogging": true },
        }

    ]

}
```

以上内容所描述的行为差不多相当于执行：
```bash
$ gdb
$ file ${workspaceFolder}/apps/hv/hv_qemu-virt-riscv.elf
$ target remote localhost:1234
```

makefile也应随之修改，把启动gdb的内容删掉：

```makefile
debug: build
    $(call run_qemu_debug)
```

到了这一步，只需要先``make ARCH=riscv64 A=apps/hv HV=y LOG=info debug``启动qemu，如果正常的话qemu会暂停等待连接；然后再于vscode启动调试，待连接完成，qemu会继续执行，我们也可以直接在vscode进行单步/断点等操作了。





# 3. rust-gdb

参考[【笔记】rCore (RISC-V)：GDB 使用记录 | 苦瓜小仔 (zjp-cn.github.io)](https://zjp-cn.github.io/posts/rcore-gdb/#%E4%BD%BF%E7%94%A8-rust-gdb)

使用``rust-gdb``可以更好地显示 Rust 的类型，不一定必须。

通过环境变量启动（gdb版本可能不同，改成自己要用的就行）：
```bash
$ RUST_GDB=riscv64-linux-gdb rust-gdb
```

此时能看到启动后的target信息``This GDB was configured as "--host=x86_64-pc-linux-gnu --target=riscv64-buildroot-linux-musl".``

```bash
GNU gdb (GDB) 13.2
Copyright (C) 2023 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.

** This GDB was configured as "--host=x86_64-pc-linux-gnu --target=riscv64-buildroot-linux-musl". **

Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
```
这时候``set architecture``应该会提示有riscv的架构了。

到VSCode部分，根据之前的``launch.json``进行部分修改，其中使用``"environment"``好像没用，于是就自己添加了环境变量``RUST_GDB=riscv64-linux-gdb``，把``"environment"``注释掉了。

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
            "program": "${workspaceFolder}/apps/hv/hv_qemu-virt-riscv.elf",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${fileDirname}",
            // gdb wrapper
            // 用这个不知道为啥没成功，只好自己在~/.bashrc手动添加了。可以亲自试试有没有用
            // "environment": [
            //     // {"name" : "RUST_GDB", "value": "riscv64-linux-gdb"},
            //     // {
            //     //     "name": "RUST_GDB",
            //     //     "value": "riscv64-linux-gdb"
            //     // }
            // ],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description": "将反汇编风格设置为 Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                },
                // {
                //     "description": "Set architecture to riscv64",
                //     "text": "-gdb-set architecture"
                // },
                {
                    "description": "Set architecture to riscv64",
                    "text": "-gdb-set architecture riscv:rv64"
                }
            ],
            // arceos makefile中默认用的gdb-multiarch，由于我这用gdb-multiarch有点问题所以随便找了个能用的其他版本，可以按需切换
            // "miDebuggerPath": "riscv64-linux-gdb",
            // "miDebuggerPath": "RUST_GDB=riscv64-linux-gdb rust-gdb",
            "miDebuggerPath": "rust-gdb",
            // qemu启动gdb server的地址，通常为localhost:1234
            "miDebuggerServerAddress": "localhost:1234",
            // "serverStarted": "localhost:1234",
            // 如果出现蜜汁问题可以打开gdb的log，在调试控制台查看更详细的信息
            "logging": { "engineLogging": true },

            // "preLaunchTask":{
            //     "task": "arceos_debug",
            //     "type": "shell"
            // }
            // "preLaunchTask": "arceos_debug"
            
        }
    ]
}
```




# 4. todo

我尝试在``launch.json``添加``preLaunchTask``来完成一键启动，即让vscode全套执行先启动qemu+后启动gdb的过程，从而达到一键debug。然而我不知道如何操作``preLaunchTask``的同步，看上去vscode执行完启动qemu的指令后就立刻启动了gdb，它没有等待qemu进入连接状态后在启动gdb，即使我在指令后添加了``sleep 5``也没有效果。我暂时没有找到办法完成这一点。


