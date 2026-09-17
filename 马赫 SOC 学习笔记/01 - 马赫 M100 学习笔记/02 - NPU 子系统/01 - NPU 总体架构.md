# 1 NPU 架构概览

![[M100 NPU 参考架构.excalidraw | 1000]]
上面这是一个方便记忆的参考架构，后续我们从多方面详细学习。

# 2 NPU 总体架构

![[M100 NPU 详细总体架构.excalidraw | 1000]]
这是比较详细的 NPU 内部展开图

# 3 NPU 数据流

## 3.1 Tile 外部数据流

### 3.1.1 可以直接访问的方式（AXI）

![[M100 外部数据流 AXI.excalidraw| 1000]]
由 Host CPU 或者 CCPU 直接控制（写相关地址，或者 load/store  指令）

- Host CPU CSR访问
- Vector CPU 访问 DRAM
- Vector CPU 访问 Tile SRAM

### 3.1.2 通过 DMA 搬运的方式（DMA）

![[M100 外部数据流 DMA.excalidraw | 1000]]
由 Host CPU 或者 CCPU 来控制，由 DMA 搬运

- DRAM 和 HIB SRAM 之间的数据传输
- DRAM 和 Tile SRAM之间的数据传输

### 3.1.3 数据广播到 Tiles（DRB）

![[M100 外部数据流 DRB.excalidraw| 1000]]
这里通过 Data Ring Bus 进行广播，由硬件自动选择是否接收数据（需要发送指令里带有相关标识）

### 3.1.4 Vector CPU 发送指令到 Tiles（ICB）

![[M100 外部数据流 ICB.excalidraw | 1000]]
ICB是一个非闭环的单向总线，也具有广播能力

## 3.2 Tile 内部数据流

（见 TPB 文章内容）

# 4 NPU 总线

## 4.1 指令链状总线（ICB）

### 4.1.1 简介

在马赫 1.0 架构中，ICB 被设计用于从 HIB 向一个或多个 Tile（TPB）单向传输指令。NPU 运行时，HIB 中的 vector CPU 会根据业务要求生成要执行的马赫 1.0 指令，随后通过 HIB 中的 ICBN（Instruction Chain Bus Node）向 ICB 发送该指令。

每条马赫 1.0 指令都由一个 128-bit 的包头 + 指令相应的 Payload 两部分组成。包头定义如下：

| 位范围 | 宽度（bits） | 域名 | 描述 |
| --- | --- | --- | --- |
| 63:0 | 64 | Tile 掩码 | 标明指令目的地的掩码，相应的位置 '1' 代表需要发送给相应的 Tile |
| 71:64 | 8 | opcode | 唯一的指令标识，定义详见《马赫 1.0 指令集架构》 |
| 95:72 | 24 | payload 长度 | payload 的长度（bytes） |
| 127:96 | 32 | 指令标签 | 用于 debug |

ICB 负责将这些指令送达指令指定的接收者。**设计规格：**

1. 支持 1 个发送端和 14 个接收端，总线宽度 64 bits。（M100 实际挂接 14 个 TPB Cluster，即 14 个接收节点；64-bit Tile 掩码可支持最多 64 个 Tile，实际为 56 个 TPB。）
2. 支持 ECC 纠错，纠错码采用汉明码，自动纠正 1 bit 的数据错误；超出纠错能力时支持上报错误。
3. 支持 4 个虚拟通道，各通道间传输互不干扰。禁止使用不同通道同时给同一个 Tile 发送指令，检测到该行为时支持上报错误。
4. 支持 Harvest（容错裁剪）。

### 4.1.2 结构及基本运作逻辑
![[M100 NPU 总线 ICB.excalidraw | 1000]]


ICB 是一条非闭环的菊花链（daisy chain）：起点是 HIB 中的 ICBN，由 HIB 内的 RISC-V core 把指令送入该节点；之后链上依次串接各个 Tile Cluster 的 ICBN。Cluster 的 ICBN 收到发给本 Cluster 的指令后，把指令分发到 Cluster 内 4 个 Tile 各自的 Tile Instruction Queue（IQ）；指令在链路上继续向下游传播，直到掩码被清零。整条链路单向传输，天然具备广播 / 多播能力。

每一个接收到这条指令的 ICBN 会做下面的事情：

1. 指令头中的 Tile 掩码不可能是零。如果是零，这是一个错误。
2. 判断指令头中的 Tile 掩码是否包含本 Tile：
   1. 如果包含，把这条指令放入相应指令 buffer，然后把本 Tile 对应的掩码位置零，再看剩下的掩码是不是零：
      1. 如果是零，这条指令就不用继续往下传了。
      2. 如果不是零，把这条指令继续往下一个节点传。
   2. 如果不包含，直接把这条指令继续往下一个节点传。

### 4.1.3 接口定义

如前所述，ICB 是一个单向传输的总线，主要由传输线路和 ICBN 组成。在一个 ICBN 中，与上一个 ICBN 相连的接口称为**上游接口**，与下一个 ICBN 相连的接口称为**下游接口**，指令沿上游 → 下游方向流动。

![[M100 NPU 总线 ICB2.excalidraw]]

各信号定义如下：

| 信号名 | 属性 | 宽度（bits） | 描述 |
| --- | --- | --- | --- |
| data | 上游 IN / 下游 OUT | 64 | 总线数据 |
| ecc | 上游 IN / 下游 OUT | 8 | ECC 校验码 |
| vcid | 上游 IN / 下游 OUT | 2 | 虚拟通道 ID |
| valid | 上游 IN / 下游 OUT | 1 | 脉冲信号，代表数据已经 ready |
| credit | 上游 OUT / 下游 IN | 1 | 下游返回的信用（credit）脉冲 |
| credit_vcid | 上游 OUT / 下游 IN | 2 | 虚拟通道 ID（响应） |

ICBN 和它的下游邻居之间使用 **Credit-based 协议**通信：Credit 的数目由两个 ICBN 之间的延时决定，Credit 的数量应该是两个 ICBN 之间延时 cycle 数的两倍。

举例：如果一个 ICBN 和它下游的 ICBN 之间因为距离关系，数据和 credit 需要 2 个 cycles 才能传到，那么下游 ICBN 需要维护一个 4 个 entries 的缓冲，而上游 ICBN 的 credit 会初始化为 4。

### 4.1.4 特性描述

#### 4.1.4.1 传输马赫 1.0 指令

ICB 总线宽度为 64 bits，ICBN 在发送数据时应将指令拆分成多个 64 bits 宽的数据包发送到 ICB 上：

| 周期（cycle） | 数据 |
| --- | --- |
| 1 | Tile 掩码[63:0] |
| 2 | opcode[7:0] + payload 长度[31:8] + header 保留字段[64:32] |
| 3 | payload slice 1 |
| … | … |

在一个虚拟通道内，指令传递具有原子性，即必须完整传完一个指令后，才能传递下一个指令。不同虚拟通道间没有原子要求；因为 ICB 每一个虚拟通道的数据源头只有一个，所以 ICBN 传输指令的原子性是自然保障的。

#### 4.1.4.2 虚拟通道

NPU 可以同时运行最多 4 个程序，为了保证这 4 个指令流不会互相阻塞，ICB 支持 4 个虚拟通道。

一个 Tile 不会同时接受来自多于一个虚拟通道的指令。也就是说，在一个虚拟通道的指令完全进入 Tile 的 IQ 之前，不会有另一个虚拟通道的指令也到达 Tile；如果发现这种情况，ICBN 可以记录并报告错误。

HIB 中有 4 个 Vector CPU，可以最多同时运行 4 个任务，这 4 个 Vector CPU 绑定到 4 个不同虚拟通道上：

1. Vector CPU 0 固定使用虚拟通道 0 发送指令。
2. Vector CPU 1 固定使用虚拟通道 1 发送指令。
2. Vector CPU 2 固定使用虚拟通道 2 发送指令。
4. Vector CPU 3 固定使用虚拟通道 3 发送指令。

#### 4.1.4.3 Harvest

在 NPU 有物理 defect、需要进行 harvest（容错裁剪）的时候，ICB 依然需要维持基本功能。换句话说，NPU 中的 ICBN 逻辑和 ICBNs 之间连线的 defect 不能超出设计的纠错能力。

## 4.2 数据环状总线（DRB）

### 4.2.1 简介

数据环状总线（下文简称 DRB）的核心设计目标，是把数据包单播或者多播到多个指定的 Tile（TPB）上。任何一个 Tile 或者 HIB，都可以往 DRB 发送数据包。DRB 的数据包由数据包头和数据 payload 组成，数据包头中包含指定接收端的掩码 `destination_mask`，用于指定接收该数据包的 Tiles。

**设计规格：**

1. 支持 16 个 DRBN（14 个 Cluster + 2 个 HIB DMA），总线宽度 512 bits。（M100 实际为 **14 个 TPB Cluster + 2 个 HIB DMA，共 16 个 DRBN**；64-bit 目的地掩码对应 56 个 TPB。）
2. 支持 ECC 纠错，纠错码采用汉明码，自动纠正 1 bit 的数据错误；超出纠错能力时支持上报错误。
3. 支持 4 个虚拟通道，通道间传输互不影响。软件必须保证一个 DRBN 节点不会同时接收多于一个虚拟通道上的数据包，硬件碰到这种情况应该报错。
4. 支持目的地同步。
5. 支持 Harvest（容错裁剪）。

### 4.2.2 部署结构与基本运作规则

DRB 是由 2 种 DRBN 的若干部署实例首尾相连形成的环状总线，如下图所示。与 ICB 的开放菊花链不同，DRB 是闭合环，没有链头和链尾：HIB 内的2 个 HIB DMA DRBN 也串接在环上，DMA 可以把数据包注入环中；每个 TPB Cluster 中的 Cluster DRBN 收到发给本簇的数据包后，把数据分发给簇内 4 个 Tile。

![[M100 NPU 总线 DRB.excalidraw | 1000]]

为了保证任何一个 DRBN 都可以向 DRB 发送数据包，环上通过 token（令牌）仲裁发送权。DRB 的基本运作规则如下：

1. 只有拥有 token 的 DRBN 可以发送数据。
2. 任何一个 DRBN 在收到 token 时，应按如下步骤顺序执行：
   1. 如果本 DRBN 有数据包要发送，发送当前 DRBN 的一个或多个数据包（发送数量参考 DRBN Control 寄存器）；否则本步骤结束。
   2. 向下游传递 token。
3. 任何一个节点在收到数据包时，应按如下步骤顺序执行：
   1. 如果数据包头中的 `destination_mask` 包含自己，接收该数据包的数据并去除 `destination_mask` 中自己对应的 mask；否则本步骤结束。
   2. 如果 `destination_mask` 不为零，则继续向下游传递该数据包；否则本步骤结束，数据包传输完毕。

DRB 会在 token 的驱动下逐个节点完成数据发送：token 绕环轮转一周，每个节点都公平地获得一次发送机会；数据包则沿环逐跳转发，靠掩码清零决定在哪个节点终止——这也是"环状总线"完成单播 / 多播的方式。

### 4.2.3 接口定义

在一个 DRBN 中，与上一个 DRBN 相连的接口称为**上游接口**，与下一个 DRBN 相连的接口称为**下游接口**，数据沿上游 → 下游方向在环上逐跳流动。

![[M100 NPU 总线 DRB2.excalidraw]]

各信号定义如下：

| 信号名 | 属性 | 宽度（bits） | 描述 |
| --- | --- | --- | --- |
| data | 上游 IN / 下游 OUT | 512 | 总线数据 |
| ecc | 上游 IN / 下游 OUT | 36 | 由汉明码生成的校验数据 |
| vcid | 上游 IN / 下游 OUT | 2 | 虚拟通道 ID |
| valid | 上游 IN / 下游 OUT | 1 | 脉冲信号，代表数据已经 ready |
| credit | 上游 OUT / 下游 IN | 1 | 下游返回的信用（credit）脉冲 |
| credit_vcid | 上游 OUT / 下游 IN | 2 | 虚拟通道 ID（响应） |
| us_token_vc0 | 上游 IN / 下游 OUT | 1 | 虚拟通道 0 的 token 信号 |
| us_token_vc1 | 上游 IN / 下游 OUT | 1 | 虚拟通道 1 的 token 信号 |
| us_token_vc2 | 上游 IN / 下游 OUT | 1 | 虚拟通道 2 的 token 信号 |
| us_token_vc3 | 上游 IN / 下游 OUT | 1 | 虚拟通道 3 的 token 信号 |

与 ICB 相同，DRBN 和它的下游邻居之间使用 **Credit-based 协议**通信：Credit 的数目由两个 DRBN 之间的延时决定，应该是两个 DRBN 之间延时 cycle 数的两倍。例如一个 DRBN 和下游 DRBN 之间数据和 credit 需要 2 个 cycles 才能传到，那么下游 DRBN 需要维护一个 4 个 entries 的缓冲，上游 DRBN 的 credit 会初始化为 4。

相比 ICB，DRB 的接口多出 4 根 `us_token_vc*` 信号：token 按虚拟通道独立轮转，4 个通道各有一枚令牌沿环传递，节点只有持有某个通道的 token 时，才有权在该通道上发送数据包。

### 4.2.4 特性描述

#### 4.2.4.1 传输数据包

DRB 的数据包由数据包头和数据 payload 组成。数据包头与总线宽度相同，即 512 bits，结构如下：

| 域名 | 位范围 | 宽度（bits） | 描述 |
| --- | --- | --- | --- |
| destination_mask | 63:0 | 64 | 标明数据目的地的掩码，相应的位置 '1' 代表需要发送给相应的 Tile |
| starting_address | 84:64 | 21 | 写入 UM（Tile 本地统一存储）的起始地址，详细约束见下方 Note |
| byte_num | 105:85 | 21 | 一共的有效字节数，详细约束见下方 Note |
| watcher_0_en | 106 | 1 | 0：不需要 watch；1：需要 watch |
| watcher_0_sv_sel | 112:107 | 6 | 同步变量选择 |
| watcher_0_sv_val | 133:113 | 21 | 同步变量的值 |
| watcher_1_en | 134 | 1 | 0：不需要 watch；1：需要 watch |
| watcher_1_sv_sel | 140:135 | 6 | 同步变量选择 |
| watcher_1_sv_val | 161:141 | 21 | 同步变量的值 |
| producer_en | 162 | 1 | 目的地 Tile DRB 写入 UM 同步 producer 配置：0 不更新 DRB 写入 UM 同步变量；1 更新 DRB 写入 UM 同步变量 |
| reserved | 511:163 | 349 | 填充 0 |

> [!note] 地址对齐
> 1. `starting_address` 应该是 64 Bytes 对齐的。如果不是 64-Byte 对齐，第一个数据 payload 会占用 64-Byte payload 的高位：比如 `starting_address` 是 17，那么第一个数据 payload 的有效字节是 [63:17]，低位字节 [16:0] 是无效的。
> 2. 如果 `starting_address + byte_num` 不是 64-Byte 对齐，最后一个数据 payload 会占用 64-Byte payload 的低位：比如 `starting_address + byte_num = 533`，那么最后一个数据 payload 的有效字节是 [20:0]，高位字节 [63:21] 是无效的。

在发送数据包时，第 1 个 cycle 固定传输数据包头；随后在有 payload 的情况下应立即开始传输 payload，软件应将 payload 切分成 512 bits 的 slice，每个 cycle 发送一个 slice。包头中的 `watcher_*` / `producer_en` 字段用于把 DRB 数据写入与生产者/消费者同步变量（SC）关联起来，配合前述同步机制实现数据到达即触发 watch。

#### 4.2.4.2 虚拟通道及仲裁

为了避免某个 NPU 程序独占 DRB 太长时间、导致其它 NPU 程序无法执行，DRB 可以同时支持最多 4 个虚拟通道。软件必须保证一个 DRBN 节点不会同时接收多于一个虚拟通道上的数据包，硬件碰到这种情况应该报错。

**仲裁：** 任意一个 cycle，若 DRBN 需要传输的数据通道超过 1 个，则 DRBN 需要仲裁出唯一的胜者。仲裁协议为 round robin（轮询）：若上一个 cycle 被选中的虚拟通道为 n，则当前 cycle 各虚拟通道的优先级从高到低排序为：

```text
(n+1) mod 4
(n+2) mod 4
(n+3) mod 4
(n+0) mod 4
```

即刚被服务过的通道 n 在本轮优先级最低。DRBN 的仲裁以此保证 4 个虚拟通道之间的公平性。

#### 4.2.4.3 数据传递原子性

DRB 允许多个 Tiles 同时使用同一个虚拟通道往 data ring bus 上发送单播或者多播数据。每一个 DRBN 在一个虚拟通道上最多有两个数据源：

1. 前面节点传过来的数据；
2. 本地产生的数据。

DRBN 无论处理哪一个数据源的数据，都必须保证传送完一个虚拟通道上的一个完整数据包，然后才能从另外一个数据源接收并传送另一个数据包。这一点与 ICB 的原子性要求相同，但 ICB 每个虚拟通道只有一个数据源头，原子性自然成立；DRB 是多主环形结构（上游转发 + 本地注入两个源），必须靠节点自身按包边界切换数据源来保证。


> [!NOTE] 提示
> 这里的数据一致性应该是全局的，而不是上下跳之间。因此 sv 是全局的


#### 4.2.4.4 防止死锁情况产生

DRB 设计允许所有节点发送数据包，且要保证数据包的原子性；如果不设计相应的总线协议，很容易出现死锁。

举例来说，假定有一个 DRB 由 A~D 4 个 DRBN 组成，TOPO 结构为 A → B → C → D → A。假定在同一时间、同一通道发生了如下访问：

1. A 在不知道 C 已经开始传输的情况下，开始往 D 发数据包 a；
2. C 在不知道 A 已经开始传输的情况下，开始往 B 发数据包 c。

当数据包 a 到达 C 时，因为 C 已经在发送 c 了，所以 a 被堵在 A、B 上；而数据包 c 到达 A 时，因为 A 已经在发送 a 了，所以 c 被堵在 C、D 上。DRB 进入死锁状态。

为了避免这种情况，DRBN 用 token ring 协议来仲裁对 DRB 的使用（整体运作逻辑见 4.2.2 部署结构与基本运作规则），Token 细节补充如下：

1. DRBN 在处理 Token 时应保证在处理完后始终向下游传递，**Token 永不消失**；
2. NPU 初始化后，只有 HIB 中的 DRBN 拥有各个通道的 Token，并在完成初始化后立即发送。

这样每个通道在环上同一时刻只有一枚令牌，只有持 token 的节点才能注入本地数据包，环路上不可能同时出现两个互相占用、互相等待的发送，死锁在协议层面被消除。

#### 4.2.4.5 同步并发访问

HIB DMA 的 DRBN 和 RISC-V CPU 可能并发访问 SRAM。为了避免在使用 DRB 时发生预期外的数据覆盖、保证数据完整性，DRB 支持使用同步变量来同步上述并发访问。

- 每个 DRB 数据包包含 2 个 watcher（`watcher_0` 和 `watcher_1`），只有两个 watcher 的条件都被满足后，数据才会被发送端的 DRBN 发送（对应包头中的 `watcher_*_en` / `watcher_*_sv_sel` / `watcher_*_sv_val` 域）；
- 在完成写入后，如果数据包配置了 `producer_en`，DRB 会根据目的地址更新特定的同步变量，即作为 producer 推进写入端的同步进度。

watcher（消费端条件）+ producer_en（生产端推进）配合，使 DRB 的 DMA 写入与 CPU 的 SRAM 访问可以通过同步变量自动握手，无需软件轮询。

## 4.3 数据网格总线 MeshBus（MN）

### 4.3.1 简介

数据网格总线（MeshBus，简称 MN）是 NPU 中**点到点**的主要数据通道，采用 Arteris FlexNoc（AI package），是一个支持 AXI 接口的片上网络（NoC）。NPU 内部署如下图所示：

![[M100 NPU 总体架构.excalidraw | 1000]]

4 行 4 列、一共 16 个节点，每个节点上有一对 AXI4 master/slave 接口，其中：

1. 14 个分别连接到 16 个 Tile Clusters；
2. 2 个连接到 HIB；

**设计规格：**

1. 读写带宽为 64 Bytes（512 bits）；
2. 支持不少于 32 个 outstanding transactions。

### 4.3.2 接口定义

MN 节点对外就是标准的 **AXI4 master/slave 接口**，信号定义见 NoC 章节定义（标准 AXI4 通道：AW/W/B 与 AR/R），这里不再罗列。正因为对外是标准 AXI4，MN 上可以直接挂接 DMA、SRAM、Vector CPU 等各种 AXI 主从设备。

# 5 地址映射

NPU 地址映射分为三层：**CPU 端口地址 → 系统物理地址 → NPU 内部 SRAM／CSR 子窗口**。同一资源可以通过不同 CPU 端口访问，但地址前缀和访问权限可能不同。

![[M100 NPU 地址映射.excalidraw | 1000]]

图中 P1～P4 对应系统物理窗口，N0～N6 对应 NPU 内部子窗口。橙色表示 **HCPU Only**，红色表示 **CCPU Only**，蓝色表示 **CCPU & HCPU available**，绿色表示 **Intra Cluster Only**。

地址区间包含首尾，窗口大小为 `结束地址 − 起始地址 + 1`。容量采用 KiB、MiB、GiB 等二进制单位；地址窗口用于资源分配，不等同于实际存储容量，窗口中的空洞和保留区不能当作普通内存访问。

## 5.1 CPU 地址窗口

Memory Port 和 System Port 各占一个 **128 GiB 地址窗口**。它们是 CPU 访问资源的两种端口视图，不是两份独立的存储器。

### 5.1.1 Memory Port

| 编号 | 资源 | CPU 地址 | 访问限制 |
| --- | --- | --- | --- |
| L0 | Local SRAM | `0x20_0000_0000` ～ `0x20_0003_FFFF` | CCPU Only |
| L1 | Local UM | `0x20_0004_0000` ～ `0x20_007F_FFFF` | CCPU Only |
| P1 | NPU Slave APB | `0x20_2B00_0000` ～ `0x20_2B00_1FFF` | HCPU Only |
| P2 | NPU SRAM／CSR | `0x20_8000_0000` ～ `0x20_FFFF_FFFF` | CCPU Only，HIB SRAM 子段除外 |
| P2a | HIB SRAM 子段 | `0x20_8800_0000` ～ `0x20_89FF_FFFF` | CCPU & HCPU |
| P3 | System SRAM | 起始地址 `0x2F_6000_0000` | CCPU & HCPU |
| P4 | DDR／DRAM | `0x30_0000_0000` ～ `0x3F_FFFF_FFFF` | CCPU & HCPU |

**P2a 包含在 P2 内部**，不能重复计算容量。Memory Port 的 NPU SRAM／CSR 窗口主要供 CCPU 使用，其中 HIB SRAM 子段同时允许 HCPU 访问。

L0、L1 是 CCPU 局部窗口：L0 占 256 KiB，L1 占 7.75 MiB，合计覆盖该端口起始处的 8 MiB。它们不能直接套用系统物理窗口的地址转换公式，也不能与 N0 的 Cluster Local UM 窗口简单等同。

### 5.1.2 System Port

| 编号 | 资源 | CPU 地址 | 访问限制 |
| --- | --- | --- | --- |
| P1 | NPU Slave APB | `0x40_2B00_0000` ～ `0x40_2B00_1FFF` | HCPU Only |
| P2 | NPU SRAM／CSR | `0x40_8000_0000` ～ `0x40_FFFF_FFFF` | CCPU & HCPU |
| P3 | System SRAM | 起始地址 `0x4F_6000_0000` | CCPU & HCPU |
| P4 | DDR／DRAM | `0x50_0000_0000` ～ `0x5F_FFFF_FFFF` | CCPU & HCPU |

System Port 的 NPU SRAM／CSR 父窗口允许 CCPU 和 HCPU 访问，但目标子窗口仍有自身限制，例如 N0、N1 只供 Cluster 内访问。**确定 CPU、端口和目标子窗口后，才能判断访问是否允许。**

## 5.2 系统物理地址与端口地址

系统物理地址记为 `PA`，主要窗口如下：

| 编号 | 资源 | 物理地址 PA | 窗口大小 |
| --- | --- | --- | --- |
| P1 | NPU Slave APB | `0x2B00_0000` ～ `0x2B00_1FFF` | 8 KiB |
| P2 | NPU SRAM／CSR | `0x8000_0000` ～ `0xFFFF_FFFF` | 2 GiB |
| P3 | System SRAM | 起始地址 `0x0F_6000_0000` | — |
| P4 | DDR | `0x10_0000_0000` ～ `0x1F_FFFF_FFFF` | 64 GiB |

P1～P4 对应窗口中的有效物理地址，通过以下偏移得到 CPU 端口地址：

```text
Memory Port CPU 地址 = PA + 0x20_0000_0000
System Port CPU 地址 = PA + 0x40_0000_0000
```

例如，NPU Slave APB 的物理起始地址为 `0x2B00_0000`，对应 Memory Port 的 `0x20_2B00_0000` 和 System Port 的 `0x40_2B00_0000`，两个端口都只允许 HCPU 访问。

这组公式用于系统窗口的对应关系，不适用于 L0、L1 局部窗口；计算出的地址仍需满足目标窗口的访问限制。

**`0x8000_0000～0xFFFF_FFFF` 是 NPU SRAM／CSR 的 2 GiB 物理窗口。** DDR 位于 `0x10_0000_0000～0x1F_FFFF_FFFF`，地址跨度为 64 GiB。System SRAM 是另外一个独立窗口，不属于这段 NPU SRAM／CSR 分配。

## 5.3 NPU SRAM／CSR 内部分配

P2 内部按用途分为局部资源、全局存储窗口、控制寄存器和保留区：

| 编号 | 资源 | 物理地址范围 | 窗口大小 | 用途 |
| --- | --- | --- | --- | --- |
| N0 | Cluster Local UM | `0x8000_0000` ～ `0x807F_FFFF` | 8 MiB = 4 × 2 MiB | Cluster 内部 UM 访问 |
| N1 | Cluster Local CSR | `0x8100_0000` ～ `0x81FF_FFFF` | 16 MiB | Cluster 内部 CSR 访问 |
| N2 | HIB SRAM | `0x8800_0000` ～ `0x89FF_FFFF` | 32 MiB = 4 × 8 MiB | HIB SRAM 窗口 |
| N3 | UM 全局窗口 | `0x9000_0000` ～ `0x97FF_FFFF` | 128 MiB = 64 × 2 MiB | 全局 UM 访问 |
| N4 | Cluster CSR | `0xC000_0000` ～ `0xCFFF_FFFF` | 256 MiB = 16 × 16 MiB | 各 Cluster 的 CSR 窗口 |
| N5 | HIB CSR | `0xE000_0000` ～ `0xE1FF_FFFF` | 32 MiB | HIB 子模块 CSR 窗口 |
| N6 | Scheduler 保留区 | `0xF000_0000` ～ `0xFFFF_FFFF` | 256 MiB | 调度器保留 |

- **局部视图**：N0、N1 只供 Cluster 内部访问。N0 对应一个 Cluster 中 4 个 Tile 的 UM，N1 对应 Cluster 局部 CSR。
- **全局视图**：N3 为 UM 全局窗口，N4 按 Cluster 编号划分控制寄存器地址。局部和全局视图不能简单相加为总 SRAM 容量。
- **HIB 资源**：数据存储使用 N2，控制寄存器使用 N5，两者是不同窗口。
- **保留空间**：N6 不作为普通 SRAM 使用；各窗口之间的空洞也不属于上述资源。

16 个 Cluster、64 个 Tile 表示地址布局规模，实际可访问单元取决于芯片启用配置。

## 5.4 Cluster CSR 子窗口

### 5.4.1 Cluster 基址

N4 共 256 MiB，按 Cluster 0～15 划分，每个 Cluster 占 16 MiB：

```text
base(k) = 0xC000_0000 + k × 0x0100_0000，k = 0…15
Cluster k 范围 = base(k) ～ base(k) + 0x00FF_FFFF
```

| Cluster | 物理地址范围 | 窗口大小 |
| --- | --- | --- |
| Cluster 0 | `0xC000_0000` ～ `0xC0FF_FFFF` | 16 MiB |
| Cluster 1 | `0xC100_0000` ～ `0xC1FF_FFFF` | 16 MiB |
| … | … | … |
| Cluster 15 | `0xCF00_0000` ～ `0xCFFF_FFFF` | 16 MiB |

### 5.4.2 Cluster 内部偏移

下表使用相对于 `base(k)` 的偏移。**物理地址 = Cluster 基址 + 子模块偏移**；需要 CPU 端口地址时，再加对应端口偏移。

| 子模块 | 起始偏移 | 结束偏移（含） | 窗口大小 |
| --- | --- | --- | --- |
| Tile 0 CSR | `+0x0000_0000` | `+0x001F_FFFF` | 2 MiB |
| Tile 1 CSR | `+0x0020_0000` | `+0x003F_FFFF` | 2 MiB |
| Tile 2 CSR | `+0x0040_0000` | `+0x005F_FFFF` | 2 MiB |
| Tile 3 CSR | `+0x0060_0000` | `+0x007F_FFFF` | 2 MiB |
| cmisc | `+0x0080_0000` | `+0x0080_0FFF` | 4 KiB |
| cdrbn | `+0x0080_1000` | `+0x0080_1FFF` | 4 KiB |
| ciq | `+0x0080_2000` | `+0x0080_2FFF` | 4 KiB |
| cich | `+0x0080_3000` | `+0x0080_3FFF` | 4 KiB |
| ccpu | `+0x0080_4000` | `+0x0080_4FFF` | 4 KiB |
| cvm | `+0x0080_5000` | `+0x0080_5FFF` | 4 KiB |
| ccru | `+0x0080_6000` | `+0x0080_6FFF` | 4 KiB |
| ccpm | `+0x0080_7000` | `+0x0080_7FFF` | 4 KiB |

Tile CSR 起点步长为 2 MiB；`cmisc` 到 `ccpm` 的控制块起点步长为 4 KiB。**CSR 窗口是寄存器的地址分配，不是额外的 UM 容量**；具体寄存器访问需使用模块定义的有效偏移。

Cluster local SRAM 的起始地址为 `base(k) + 0x008C_0000`。

## 5.5 HIB CSR 子窗口

N5 的范围为 `0xE000_0000～0xE1FF_FFFF`，共 32 MiB，包含以下子模块：

| 子模块 | 物理地址范围 | 窗口大小 |
| --- | --- | --- |
| HDMA | `0xE000_0000` ～ `0xE01F_FFFF` | 2 MiB = 2 × 1 MiB |
| HMI | `0xE020_0000` ～ `0xE020_0FFF` | 4 KiB |
| HCE | `0xE020_1000` ～ `0xE020_4FFF` | 16 KiB = 4 × 4 KiB |
| HDRBN | `0xE020_5000` ～ `0xE020_5FFF` | 4 KiB |
| HCRU | `0xE020_6000` ～ `0xE020_6FFF` | 4 KiB |

HDMA 分配两个 1 MiB 子窗口，HCE 分配四个 4 KiB 子窗口。父窗口的 32 MiB 大于这些子窗口的容量之和，其余范围不能直接当作有效寄存器访问。

## 5.6 寻址示例

### 5.6.1 Cluster 1 的 Tile 2 CSR

先确定 Cluster 1 的基址，再加 Tile 2 CSR 偏移：

```text
Cluster 1 基址 = 0xC000_0000 + 0x0100_0000 = 0xC100_0000
Tile 2 CSR 起点 = 0xC100_0000 + 0x0040_0000 = 0xC140_0000
Tile 2 CSR 终点 = 0xC140_0000 + 0x0020_0000 − 1 = 0xC15F_FFFF
```

所以物理地址范围为 **`0xC140_0000～0xC15F_FFFF`**。访问某个具体寄存器时，还要加上寄存器在 Tile 2 CSR 子窗口内的有效偏移。

### 5.6.2 HMI 的端口地址

HMI 子窗口从物理地址 `0xE020_0000` 开始，对应两种 CPU 端口地址：

| 地址视角 | 起始地址 | 访问限制 |
| --- | --- | --- |
| 系统物理地址 PA | `0xE020_0000` | HIB CSR 中的 HMI 子窗口 |
| CPU Memory Port | `0x20_E020_0000` | CCPU Only |
| CPU System Port | `0x40_E020_0000` | CCPU & HCPU |

三个地址指向同一目标。HCPU 访问 HMI 应使用允许 HCPU 访问的 System Port 窗口，不能仅通过替换前缀改用 Memory Port。
