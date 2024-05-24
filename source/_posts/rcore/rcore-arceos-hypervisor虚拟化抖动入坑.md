---
title: rcore-arceos-hypervisor虚拟化抖动入坑
date: 2024/05/21 18:26:51
tags:
  - rcore
  - rust
  - os
categories:
  - rcore
---
# 0. tmp

```bash
make ARCH=riscv64 A=apps/hv HV=y SMP=2 LOG=debug MODE=debug debug
```

# 1. 基础设施

- [x] vsc debug
- [ ] vsc-ra跳转/识别


# 2. arceos SMP启动过程

好东西-->https://mermaid.live/

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


[![](https://mermaid.ink/img/pako:eNqVU8tuwyAQ_BVre3UiG9t5UKmHttee2lPjyiI2CZYMuBiUpFH-veC8bCVKUyTEDjvs7C6whVwWFDAsKrnKGVHa-3hORardbMx8qUjNPLJmpPJS4R3GycFlMVTN2cHCGV1rqoT38G2kfnzZL54yjc6o0Grz1SHHf5GzhuZSFKR3jIrCJncll7qfCpo12tZzjpHltenKJ7NWiZNS_EtoLqXuSwXXC8naBLqS0S3iPUkwOlstyo7f3VOHw4LB4ImFbpeF1lR7Ew0GKTTzEmPWCjm5FBw1av2RM-NDiNiB5AASB6gL33sPyghdcnqlO1U57zXHDXXrWbj-33W92UVcHl674-YiWrdDyrWFh3vAHWDo-ODdBB84VTapwv6KraOloBm1tQK2ZkEXxFS2eanYWSoxWr5vRA5YK0N9UNIsGeAFqRqLTF0QTV9LYkvgp92aiE8p-fGIhYC3sAYcBsFwjFAcjdAoQUmIkA8bwDEaToNknEwn4TRA0Sja-fDTBgiGE7s9DuIJmqARiiIfaFFqqd72n7r927tf3ds1zg?type=png)](https://mermaid.live/edit#pako:eNqVU8tuwyAQ_BVre3UiG9t5UKmHttee2lPjyiI2CZYMuBiUpFH-veC8bCVKUyTEDjvs7C6whVwWFDAsKrnKGVHa-3hORardbMx8qUjNPLJmpPJS4R3GycFlMVTN2cHCGV1rqoT38G2kfnzZL54yjc6o0Grz1SHHf5GzhuZSFKR3jIrCJncll7qfCpo12tZzjpHltenKJ7NWiZNS_EtoLqXuSwXXC8naBLqS0S3iPUkwOlstyo7f3VOHw4LB4ImFbpeF1lR7Ew0GKTTzEmPWCjm5FBw1av2RM-NDiNiB5AASB6gL33sPyghdcnqlO1U57zXHDXXrWbj-33W92UVcHl674-YiWrdDyrWFh3vAHWDo-ODdBB84VTapwv6KraOloBm1tQK2ZkEXxFS2eanYWSoxWr5vRA5YK0N9UNIsGeAFqRqLTF0QTV9LYkvgp92aiE8p-fGIhYC3sAYcBsFwjFAcjdAoQUmIkA8bwDEaToNknEwn4TRA0Sja-fDTBgiGE7s9DuIJmqARiiIfaFFqqd72n7r927tf3ds1zg)

>- [ ] 问题：wfi以后怎么办？
>
>从核在走完启动流程后会wfi，然而不能对已经wfi的hart进行``sbi::hart_start``，会报告``SBI_ERR_ALREADY_AVAILABLE``已经启动。那么如何让从核到指定位置继续运行？


# 3. Linux SMP启动过程













# 00. bin

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














