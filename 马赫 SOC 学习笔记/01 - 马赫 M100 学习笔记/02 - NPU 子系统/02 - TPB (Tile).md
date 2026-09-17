# 1 简介

Tile（在 M100 中称为 TPB）是组件中进行绝大部分计算的焦点模块。材料这一章以 Tile 为粒度介绍其内部结构。按上面的 M100 模块图并结合簇架构图，TPB 对外的端口是：指令从 Cluster 侧 CIQ 输入；数据面有两个口——**DRB Terminal 经一对线连 Tile Cluster DRBN**、**GSDU 经 AXI 连 TPB Cluster NoC 再上行 Mesh**；中断上报 Cluster RISC-V CPU；以及到 Cluster CPU 的 VCIX。

Tile 内部主要由以下部分组成：

- **LU（Linear Unit，线性单元）**：可执行 SIMD 指令。LU 中有一个**宽度 64、深度 8 的 MAC cell 阵列**（共 64 × 8 = 512 个 MAC cell）；一个 MAC cell 每一个 cycle 可以做一个 INT8 DP4 操作。LU 用于矩阵乘和卷积之类大计算密度的操作。
  > [!tip] 理论 INT8 算力推导
  > 512 个 MAC cell 每 cycle 各完成 1 次 DP4（4 对点积，含 4 乘 4 加共 8 OP），即每 cycle **512 次 DP4 = 4096 INT8 OP/cycle**，再乘以主频即 INT8 理论峰值（材料未给主频）。
- **NLU（Non-Linear Unit，非线性处理单元）**：对从 MAC 阵列转移出来的累加结果做**量化和反量化**操作，并且计算**激活函数**。
- **CVU（Configurable Vector Unit，可配置向量单元）**：神经网络中各种对性能影响大的 element-wise 计算，比如 softmax、layernorm 等，用 CVU 执行。
- **Vector CPU**：剩下那些对性能影响不大的 element-wise 计算，可以在 vector CPU 上执行。该向量单元的向量寄存器宽度 **VLEN = 512 bits**，向量计算 ALU 宽度 **DLEN = 256 bits**。
- **2MB 共享内存（M100 图中标注为 HBSM，即第 5 章的 UM）**：由多个物理 SRAM banks 组成；用地址划分，可以支持多个逻辑端口。
- **寄存器文件数据缓冲区**：靠近 LU 和 NLU 的数据端口，由寄存器文件组成，用于把数据在**内存存储格式和计算所需格式**之间进行转换。
- **多个 DMA 单元**：负责在内存、寄存器文件以及 Tile 的总线端口之间搬运数据，图中具体为 **DTDU**（数据转换 DMA）与 **GSDU**（Gather/Scatter DMA，和 Custom Engine 同块）。
- **Data Ring Bus Terminal（DRB 终端）**：TPB 内的数据环总线终端节点（即 4.2 节 DRBN 在 TPB 内的落点），与 Tile Cluster DRBN 双向相连；环上含地址和长度信息的数据包可由它**直接写入 HBSM**（无需执行单元介入）。
- **CSU（CPU Starter Unit）**：接收 CIQ 派发的 **CPU 类指令**；有一根中断线（图中紫色）上报到 Tile Cluster RISC-V CPU；并向 GSDU 发送配置 / 状态类信号（图中金色线）。
- **同步单元 SU（Sync. Unit）**：同步变量机制的硬件落点——图中各执行单元把 **Sync. counter updates** 送入 SU（青色线），SU 再把 **Sync. counters 分发给 watchers**，与 4.2.4.5 节的 watcher / producer 是同一套变量体系。Tile 中不同的执行单元（线性单元、DMA 单元等）都有自己的指令，通过执行指令完成各种运算；**同一个 Tile 内同一种类型的指令顺序执行，不同类型的指令可以乱序执行**；两条指令之间有依赖、需要保序时，即借助上述同步变量机制。

这些执行单元的指令从 **Instruction Chain Bus（ICB，见 4.1）** 传输到 Cluster，由 Cluster 的**指令队列 CIQ** 负责派发给 4 个 Tile 中的各个计算单元（M100 即每个 Cluster 4 个 TPB）。下面是 Tile 的模块图（M100 命名版本）：

![[M100 TPB 架构.excalidraw | 1000]]

# 2 主要数据通路

材料原图用红色标出 Tile 中的主要数据通路，并用数字 1～10 标明每条通路的位置。上面的 M100 版本模块图**保留了红色数据线**（右上角另有完整配色图例），但未保留 1～10 的数字编号，以下按材料编号逐一介绍，并在每条下注明其在 M100 图中的对应连线：

1. **Local UM to local UM 数据搬运**：从本地统一内存（UM）到本地 UM 的数据通路，由 **U2U DMA 引擎（M100 图中的 DTDU）** 掌管；同时支持一些数据摆放模式的转换，比如 padding 和矩阵转置。读写地址都是 Tile local UM 地址，即 **0 到 2M（不包含 2M）连续的线性地址**。图中对应 DTDU 与 HBSM 之间的一对红色箭头。
2. **Local UM to remote UMs 广播或者多播数据搬运**：从本地 UM 出发，通过 **Data Ring Bus（DRB，见 4.2）** 到一个或多个远端 Tile UMs 的数据通路，同样由 U2U DMA 引擎掌管，支持 padding、矩阵转置等摆放模式转换。读写地址都是 Tile UM 地址，即 0～2M（不含 2M）连续线性地址。图中路径为 **DTDU → Data Ring Bus Terminal → 上方 Tile Cluster DRBN**（红色箭头逐级相连）。
3. **Data ring bus 写入 UM**：DRB 是 NPU 中主要的数据广播和多播通道。Tile 从 DRB 上接收到的数据包包含地址和数据长度信息，Tile 的 DRB 节点（图中即 **Data Ring Bus Terminal**）可以**直接把数据写入 UM**，图中对应 DRB Terminal → HBSM 的红色箭头。这里的地址信息是 Tile local UM 地址，即 0～2M（不含 2M）连续线性地址。
4. **Tile 外部 agent 读写 local UM**：连在 NoC / Mesh Bus（MN，见 4.3）上的 Tile 外部 agent，可通过 **AXI Bus Bridge** 访问本地 UM。Agent 使用 **40-bit 全局物理地址**，经 Mesh 路由到 TPB Cluster NoC，再经 AXI Bus Bridge，最后使用 0～2M（不含 2M）的地址访问 Tile UM——即第 5 章所述全局地址到 UM local 地址的转换。图中该入口即 GSDU 底部标注 *To & From TPB Cluster NoC (Mesh)* 的一对红线。
5. **CVU 从 UM 读出数据进行计算**：CVU 从 UM 读数据做 vector 计算，UM 的读地址由指令配置好的 **TTU** 产生，是 0～2M（不含 2M）的本地 Tile UM 地址。
6. **CVU 往 UM 写入计算结果数据**：CVU 的计算结果写回 UM，UM 的写地址同样由指令配置好的 TTU 产生，为 0～2M（不含 2M）本地 Tile UM 地址。图中 CVU ↔ HBSM 的读 / 写两条逻辑通路合并画为一条红色箭头（箭头方向 CVU → HBSM，对应写回）。
7. **Tile Cluster CPU 通过 Custom Engine 访问 UM**：为了得到更小的延迟，Tile Cluster CPU 可通过 **VCIX 接口**（图中 GSDU 引出的蓝色粗线）使用 Custom Engine 读写 UM；还可以控制 Custom Engine 中的 **Gather/Scatter DMA（GS DMA，GSDU）** 访问 UM——图中 GSDU 与 HBSM 之间画了**两条红色数据箭头**，CSU 另有一根金色配置 / 状态线连到 GSDU。
8. **Custom Engine 通过 NoC/Mesh Bus 访问本地或远端内存**：Custom Engine 中的 GS DMA（GSDU）在 Tile Cluster CPU 控制下，经 **TPB 内 AXI → TPB Cluster NoC → Mesh** 访问本地或远端内存；远端内存可以是 **Tile SRAM、HIB SRAM 或 DRAM**，远端地址是一个 40-bit 全局物理地址。
   > [!note] 原图勘误
   > 原图 GSDU 底部端口被误标为 *To & From Tile Cluster DRBN*（与顶部 DRB Terminal 的文字完全相同，应为复制粘贴错误），实际该口走的是 AXI/NoC/Mesh——簇架构图可佐证：每个 TPB 只有一根绿色 DRB 线（连 Cluster Data Ring Bus Node，归 DRB Terminal），另有一根 AXI 短箭头接中央 TPB Cluster NoC 上行 Mesh；且 DRB 是只写推送环，做不了 gather（读），也寻址不到 DRAM。两张图中的该标注均已改为 *To & From TPB Cluster NoC (Mesh)*。
9. **TCC 计算数据通道**：执行张量收缩（tensor contraction）计算的 TCC（图中整合在 TCU 内）主要有 5 个数据通道，图中概括为 HBSM ↔ TCU 之间的一对红色箭头（一进一出）：
   - a. 从 UM 中读取权重（或右矩阵）数据；
   - b. 从 UM 中读取激活（或左矩阵）数据；
   - c. 从 UM 中读取权重稀疏编码数据；
   - d. 把张量收缩计算结果写入 UM；
   - e. 从 UM 中读取数据到 NLU 中的 Bias Memory。
10. **非线性函数计算数据通道**：CVU 与非线性函数数据单元（**Spline Unit**）之间有一个数据请求和计算结果返回的通道（图中 Spline Unit 未单独画出，包含在 TCU / CVU 侧内部）。

# 3 指令通路

材料原图中 Tile 的主要指令通路用蓝色线条表示；在上面的 M100 图中，对应路径是从顶部 *From Cluster CIQ* 进入、扇出到各执行单元的**棕褐色（Instruction）箭头**（该图配色里蓝色粗线表示 VCIX，不是指令）。

## 3.1 指令发送

HIB 中的 CPU 负责给 Tiles 发送指令，这些 Tile 执行的指令包括张量操作指令、Tile 内 DMA 指令等。这些指令对于一个在 NPU 上执行的任务程序来说，是**保存在内存中的数据**。发送流程：

1. HIB 中的 CPU 创建一个**指令包包头**，其中包含目的地 Tiles 的信息；
2. 把 Tile 指令从内存中读入寄存器；
3. 组装成一个指令包，发送到 instruction ring bus（即指令链状总线 **ICB，见 4.1**）上去；
4. 相关的 Tiles 据此收到并执行这些 Tile 指令（链上按包头中的目的 Tile 信息过滤，与 4.1 的广播/Harvest 机制一致）。

## 3.2 指令队列

当一个 Tile 接收到一个 Tile 指令时，首先确定指令类型，然后放入相应的指令队列。Tile 执行 5 种类型的指令，因此有 **5 个指令队列**来缓冲：

- **CVU 指令队列**
- **Tensor 指令队列**
- **Vector CPU 指令队列**
- **UM2TCC 指令队列**
- **U2U 指令队列**

指令队列是由 **SRAM blocks 实现的 FIFO**：ICB node（instruction ring bus node）负责往 FIFO 里写入指令，对应的 Tile 执行单元负责从 FIFO 中读出指令、译码执行。FIFO 的宽度不能太低，否则指令读出延迟过大；也不能太高，否则造成物理实现和布线的困难。

> [!info] 五类指令在 M100 图中的去向（以图为准）
> 上图从 *From Cluster CIQ* 扇出的五根棕褐色指令箭头明确标注为：**CPU Inst. → CSU**、**TCU Inst. → TCU**、**SU Inst. → SU**、**CVU Inst. → CVU**、**DTDU Inst. → DTDU**。与材料五类队列的对应关系为：
> - CVU 指令队列 → CVU Inst.（一致）；
> - Tensor 指令队列（含 UM2TCC 喂数职能）→ TCU Inst.，材料中单列的 **UM2TCC 队列在图中未单独出现**，并入 TCU；
> - U2U 指令队列 → DTDU Inst.；
> - Vector CPU 指令队列 → CPU Inst.，但接收方在图中是 **CSU**（TPB 内无 vector CPU 同名模块）；
> - 图中另外把 **SU 单列为一类指令接收者**（SU Inst.），材料的队列清单中没有对应项（材料中同步行为由同步变量机制承担）。

# 4 张量遍历单元（TTU）

## 4.1 简介

张量是一个多维数组，每一个张量成员都可以用它在每个维度上的位置索引（index）来定位；但在硬件中，张量数据存储在一维地址空间中。所以访问张量成员，需要把一组多维度的索引转换成一个一维内存地址——方法是**把张量成员的位置索引做线性叠加**。

**TTU（Tensor Traversal Unit，张量遍历单元）** 就是一个专门把张量成员位置索引转换成内存地址的硬件单元，更精确地说，它是为张量遍历操作设计的**地址产生器**：在软件的控制下，它可以灵活而高效地产生对应一系列张量成员的地址序列，这个地址序列也对应着软件嵌套循环中使用的张量成员地址。

时序上，TTU 的流水线（pipeline）要花几个时钟周期来产生一个地址，但它**每个时钟周期都可以产出一个新地址**（流水化）。在一个 Tile 中，每一条流水线都有自己专有的 TTU；按流水线类型分两类：

1. **张量操作 TTU**（服务 TCU 等张量流水线，如数据通路第 9 条的 TCC 喂数）；
2. **DMA 操作 TTU**（服务 DMA 流水线，如数据通路第 5、6 条中 CVU 读写 UM 的地址产生）。

在 TTU 对应流水线中执行的 Tile 指令，通过配置 TTU 中的控制寄存器来定制地址序列。配置在**指令发射时**完成，此后 TTU 产生的一系列地址在该指令执行的整个生命周期中使用。例如每条张量操作指令发射时都会配置张量**初始地址（base）**以及各层循环的 **limit** 与 **stride** 寄存器，这些配置在指令结束前不会改变。

不同指令类型由相对独立的流水线执行，它们对内存读写端口的请求被动态仲裁：赢得内存端口的流水线若需要一个读写地址，就**取用 TTU 当前产生的地址**，并通过 `iterate_loop_nest` 接口让 TTU 产生下一个地址。TTU 能保存最多 **N 层**张量循环的遍历状态；张量操作和 DMA 操作需要支持的层数可能不同，所以 TTU 的具体实现是**参数化**的。

TTU 结构示意图如下（对应材料配图重绘）：

![[M100 TTU 地址生成.excalidraw | 1000]]

结合上图说明其组成：

- **控制寄存器分三组，各 N 份**：`position[i]`（当前层位置，绿色）、`stride[i]`（步幅，黄色）、`limit[i]`（限制，粉色）；N 组即对应最多 N 维张量。另有独立的 **Base Address** 寄存器存放张量起始地址，同样由指令配置。这些寄存器**软件可见**。
- **地址 = 所有 position 寄存器的值线性相加，再加上 base**：图中左侧是一棵 3 级加法树，把 8 个 position 分组两两相加（N=8 的参数化实例）；绿色方块是**流水线寄存器**，对应"几个周期产生首个地址、之后每周期出一个"。相加结果再与 Base Address 相加得到 **Intermediate Address**。
- **Pingpong 选择**：Intermediate Address 末端的二选一 MUX 在 **0** 与 **Pingpong Offset** 之间选择（由 *Use Pong Buffer* 信号控制），叠加后得到 **Final Address**——用于双缓冲（ping-pong buffer）场景下在两份缓冲间切换地址偏移；材料未展开该机制细节。
- **strides / limits 不进加法树**：它们是迭代更新 position 时用的控制寄存器（见下方伪代码），不是地址的组成部分。

迭代（iterate）规则：收到"往下循环"信号时，**最内层**循环的 position 寄存器加上一个 stride；对所有层，只要翻滚（rollover）条件成熟，它们都会在**同一个时钟周期**内完成翻滚（进位是组合逻辑级联，类似硬件进位链）。

## 4.2 硬件描述伪代码

材料给出的硬件行为伪代码（Python 风格）：

```python
# Instruction sets up ttu_init, ttu_stride registers & ttu_base register
def config(stride, limit, base):
    ttu_base = base
    for i in range(0, ttu_depth):
        ttu_position[i] = 0
        ttu_stride[i] = stride[i]
        ttu_limit[i] = limit[i]

def iterate():
    # Most inner loop starts with 0
    address = ttu_base
    for i in range(0, ttu_depth):
        address += ttu_position[i]
    rollover = True
    for i in range(0, ttu_depth):
        if rollover:
            rollover = (ttu_position[i] == ttu_limit[i])
        if ttu_position[i] >= ttu_limit[i]:
            ttu_position[i] = 0
        else:
            ttu_position[i] += ttu_stride[i]
    tensor_traversal_complete = rollover
```

几个语义要点：

- **地址先算、状态后更新**：本次 `iterate()` 输出的地址由更新前的 position 决定；随后各层 position 才按 stride/limit 更新，供下一次使用。
- **limit 是"最后一个有效位置"**：以最内层为例，position 依次取 0, 1*stride, 2*stride … 等于 limit 时该值仍参与一次寻址，随后 `>= limit` 判据把它清零并向外层进位（翻滚）。
- **rollover 级联**：`rollover` 初值为 True，从最内层向外逐层判断 `position[i] == limit[i]`；只有内层全部到达边界，外层才在本次更新中加 stride，否则保持不变。所有成熟的翻滚在同一周期完成。
- **遍历完成**：当最外层也判定 `position == limit` 时，`tensor_traversal_complete` 置位，表示整组嵌套循环走完一遍。

## 4.3 简单实例

一个 6×6 的二维矩阵，**行优先**存储，每格内是其线性地址（材料原表格最后一行最后一格写作 ~~25~~，应为 **35**，此处按正确值给出）：

| 0 | 1 | 2 | 3 | 4 | 5 |
| --- | --- | --- | --- | --- | --- |
| 6 | ==**7**== | ==**8**== | ==**9**== | 10 | 11 |
| 12 | ==**13**== | ==**14**== | ==**15**== | 16 | 17 |
| 18 | ==**19**== | ==**20**== | ==**21**== | 22 | 23 |
| 24 | 25 | 26 | 27 | 28 | 29 |
| 30 | 31 | 32 | 33 | 34 | 35 |

要依次取出高亮的 3×3 子块（地址 7,8,9 / 13,14,15 / 19,20,21），TTU 配置为：`ttu_depth = 2`、`base = 7`、`stride[0] = 1`、`limit[0] = 2`、`stride[1] = 6`、`limit[1] = 12`。二维配置与逐次迭代结果如下：

| iteration | stride[0] | limit[0] | position[0] | stride[1] | limit[1] | position[1] | base | address = base + position[0] + position[1] |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | 1 | 2 | 0 | 6 | 12 | 0 | 7 | ==**7**== |
| 1 |  |  | 1 |  |  | 0 |  | ==**8**== |
| 2 |  |  | 2 |  |  | 0 |  | ==**9**== |
| 3 |  |  | 0 |  |  | 6 |  | ==**13**== |
| 4 |  |  | 1 |  |  | 6 |  | ==**14**== |
| 5 |  |  | 2 |  |  | 6 |  | ==**15**== |
| 6 |  |  | 0 |  |  | 12 |  | ==**19**== |
| 7 |  |  | 1 |  |  | 12 |  | ==**20**== |
| 8 |  |  | 2 |  |  | 12 |  | ==**21**== |

读法：最内层 position[0] 在 0→1→2 之间走（列方向，stride = 1 个元素），到 2（= limit[0]）后翻滚为 0，同时外层 position[1] 加 6（行间距 = 一行 6 个元素）；position[1] 依次为 0、6、12。当 position[0] 与 position[1] 同时等于各自 limit（iteration 8 输出地址后），`tensor_traversal_complete` 置位，循环结束——这就是软件里两层嵌套循环在硬件上的等效执行，9 次 `iterate()` 恰好覆盖一个 3×3 子块。

# 5 统一内存（UM）

> M100 图中该 SRAM 标注为 **HBSM**；每个 TPB 容量 2MB，与本节一致（见前文第 1 节）。

## 5.1 简介

Tile 内部的 SRAM 之所以称为**统一内存（Unified Memory，UM）**，是因为 AI 计算用到的**激活（activation）和权重（weight）数据都放在这同一个统一的内存空间**中，不再分设独立存储。

两条访问规则：

- **本地性**：每个 Tile 中的计算单元只能访问**本 Tile** 的内存；Tile A 需要 Tile B 的数据时，必须用 **DMA** 把那一段数据从 Tile B 搬运到 Tile A（Tile 间搬运走 Mesh，对应数据通路第 8 条）。
- **地址来源**：Tile 内计算单元访问统一内存的地址一般由 **TTU** 产生（见第 4 章）。

容量与组织（参数取自材料）：

- 总大小 **2MB**，由多个物理 SRAM bank 组成；一组物理 SRAM bank 合起来构成一个**逻辑 bank（logic bank）**。
- 一个逻辑 bank 对外提供**一个读写端口**，是 Tile 内执行单元共享同一内存资源的**最小仲裁/访问粒度**。
- 每个逻辑 bank 的数据访问宽度为 **32 Byte**；统一内存共 **32 个逻辑 bank**。
- 故每个逻辑 bank 的深度为 $2^{21-5-5} = 2^{11} = 2048$（2MB = $2^{21}$ B；低 5 位是 bank 内 32B 偏移、再 5 位选择 32 个 bank，剩余 11 位即深度）；每个 bank 容量 = 32B × 2048 = **64KB**，32 × 64KB = 2MB。
- 物理上用 **4 个 64(bit 宽) × 2048(深) 的 SRAM hard macro** 拼出一个逻辑 bank：4 × 64 bit = 256 bit = 32B，深度同为 2048。结构如下图（对应材料配图重绘）：

![[M100 UM 存储阵列.excalidraw | 820]]

每个执行单元根据需要，**一次可以请求访问一个逻辑 bank，也可以同时请求多个逻辑 bank**。多个单元访问同一内存时，优先级是**固定**的；统一内存的访问仲裁逻辑据此决定在某一时刻哪些执行单元可以访问哪些逻辑 bank（仲裁发生在 bank 端口这一粒度上）。

## 5.2 Requests 到 SRAM 和 Regfile 的端口映射

统一内存对外的请求方（agent）共 15 个端口，方向/带宽如下（材料原表）：

| # | agent | 方向 | 带宽（Byte） | Byte mask |
| --- | --- | --- | --- | --- |
| 0 | DRBN | wr | 128 |  |
| 1 | U2U_w | wr | 32 | 有 |
| 2 | CVU_w | wr | 64 |  |
| 3 | MBN_w | wr | 64 |  |
| 4 | CE_w | wr | 32 |  |
| 5 | N2U | wr | 64 |  |
| 6 | U2U_r | rd | 32/64 |  |
| 7 | CVU_0_r | rd | 64 |  |
| 8 | CVU_1_r | rd | 64 |  |
| 9 | MBN_r | rd | 64 |  |
| 10 | CE_r | rd | 32 |  |
| 11 | U2N | rd | 32 |  |
| 12 | U2A | rd | 32/64 |  |
| 13 | U2W | rd | 64 |  |
| 14 | U2S | rd | 32 |  |

这些请求按来源主要分为 **7 类**（各类与 agent 的对应按端口命名推断：MBN=Mesh bus node、DRBN=DRB/ring node、U2A/U2W/U2S=UM 到 act/weight/sparsity(mask) regfile；`CVU_*`、`CE_*` 在材料 7 类文字中未逐一展开）：

**(1) TensorOp**

- `Tensor_NLU_activation` **wr port (64B)**
- `Tensor_NLU_activation` **rd port (128B)**

该端口可从 2.2.1 节 Tile 结构图中得到印证：只有 **NLU 的 output buffer 能往 SRAM 写数据**；LU 采用 8 合 1 结构（LU 输出具有 32 个 shift register，链接 32 个 NLU 进行运算，见数据通路第 9 条）。

**(2) Mesh bus**（对应 `MBN_w` / `MBN_r`，各 64B）

- `Mesh bus rd port (64B)`、`Mesh bus wr port (64B)`，两种功能：
  1. **Tile 间 SRAM 数据相互交换**，读写单位为一个 logic bank，即 32B；
  2. **Host CPU 经 CSR bus 访问 Tile**：CSR request 被 CSR block 接收后，会被放到 mesh bus 上去访问硬件资源。
- 该读写端口由 Mesh bus node 转换而来；Mesh bus node 的读写请求可来自：本地 Tile 中 vector core 的读写请求、**其他 Tile** 中 vector core 的读写请求、Host CPU 的 CSR bus 读写请求。

**(3) U2U bus**（对应 `U2U_r` / `U2U_w`，32B）

- `U2U rd port (32B)`、`U2U wr port (32B)`：用于**对一个 Tile 内 SRAM 中的数据进行重排序**（on-tile 转置/重排，DTDU 的职能），读写单位为一个 logic bank，即 32B。

**(4) Ring bus（512 bit / 1024 bit）**（对应 `DRBN` wr 128B）

- `Ring bus wr port (128B)`：将 DMA 从 ring bus node 接收到的、属于本 Tile 的运算数据**写入 SRAM**，操作基本单位为 **4 个 logic bank，即 128B**。

**(5) UM2RFA DMA**（对应 `U2A` rd 32/64B）

- `UM2RFA_activation rd port (64B)`：把 Unified Memory 中的 **activation** 数据搬运到 **Regfile_act** 寄存器文件。搬运方式有两种（与 LU 的两种计算模式对应）：
  - 有效数据位宽 **32B**：LU 以 **8 行 × 4B** 的方式读取 Regfile_act；
  - 有效数据位宽 **64B**：LU 以 **8 行 × 8B** 的方式读取。

**(6) UM2RFW DMA**（对应 `U2W` rd 64B）

- `UM2RFW_weight rd port (64B)`：把 **weight** 数据搬运到 **Regfile_weight** 寄存器文件，两种搬运方式（同样对应 LU 两种模式）：
  - **32B** 方式：以 **8 个 register（1 register = 1B）为间隔**，把 32B 数据填充写入 256B 位宽的 Regfile_weight；即每 **8 个 clock** 取一个 **8×32** 的矩阵小块（量化单位 1B）。
  - **64B** 方式：以 **4 个 register 为间隔**把 64B 数据填入 256B 位宽的 Regfile_weight；即每 **4 个 clock** 取一个 **4×64** 的矩阵小块（量化单位 1B）。

**(7) UM2RFM DMA（复用 UM2NLU DMA）**（对应 `U2S` rd 32B）

- `UM2RFM_mask rd port`：由 UM2RFM DMA（也叫 UM2NLU DMA）把 Unified Memory 中用于**稀疏化操作的 mask 数据**搬运到 **Regfile_mask** 寄存器文件；搬运模式与 mask 的表示方式有关。

**SRAM memory 端口优先级（固定，从高到低）**——同一个 logic bank 可能被多个 port 同时访问，需要仲裁，材料给出的优先级顺序为：

1. UM2RFA_activation rd port（64B）
2. UM2RFW_weight rd port（64B）
3. UM2RFM_mask rd port（**32B**）
4. Tensor_NLU_activation rd port（128B）
5. Tensor_NLU_activation wr port（64B）
6. Meshbus rd port（64B）
7. U2U rd port（32B）
8. Meshbus wr port（64B）
9. U2U wr port（32B）
10. Ringbus wr port（**64B**）

> [!note] 材料两处带宽前后不一致（记录备查）
> - **UM2RFM_mask**：第 (7) 类正文写 `rd port (64B)`，但优先级清单第 3 项写 **32B**；上方端口表中对应 agent `U2S` 的带宽也是 **32**。当以 32B 为实际配置、64B 为材料笔误更自洽。
> - **Ringbus wr**：第 (4) 类正文写 **128B**（且与 `DRBN wr 128`、"4 个 logic bank 为单位"一致），优先级清单第 10 项却写 **64B**。

## 5.3 SRAM memory 的存储阵列结构

材料此节未再展开，原文即"Sram memory 的存储阵列结构如简介所示"——即 5.1 节的阵列图：32 个逻辑 bank（4×8 排布），每个逻辑 bank 由 4 片 64×2048 的 SRAM hard macro 拼成 32B×2048 的读写端口。

## 5.4 SRAM memory 的仲裁逻辑

统一内存的访问仲裁逻辑原理如下面的表 1、表 2、表 3 所示。仲裁过程分为 **3 个 stage**，目前三个 stage 都是**组合逻辑**；在考虑 timing 的情况下，也可以切成 3 级流水线。

- **Stage 1（地址译码）**：对**地址译码**并结合端口的**访问位宽**，为每个 port 产生一个 **32 bit 的 bank req 信号**——32 bit 对应 32 个 logical bank（某位为 1 即该拍请求访问对应 bank，一次可占多个 bit）。如果该 port 没有请求，则 32 bit 全 0。
- **Stage 2（两两冲突）**：用 stage1 产生的 32 bit bank req，把**当前优先级 port** 与**所有比它更高优先级 port** 的 bank req **两两按位与**，再做**或（OR）归约**，得到每个 port 的请求在各 logical bank 上是否存在冲突——即一张下三角形的表（表 2）。
- **Stage 3（级联授权）**：基于 stage2 的表，产生每个 port 请求的**仲裁使能（授权）信号**。该过程是**级联**的：判断某个更高优先级请求是否挡住当前请求时，必须先知道这个更高优先级请求自己是否被比它还高的请求挡住——也就是它的仲裁使能是否已有效：若有效，它才参与"当前请求是否被挡"的比较；若无效（被更高的挡住或本来没有请求），它就不参与比较。

**表 1**：各 port 译码出的 bank 请求向量 $r_i$（32 bit；材料只画了 lb_0/lb_1/lb_2/…/lb_31 几列，并在 r_0～r_2 给了示例值）：

|  | lb_0 | lb_1 | lb_2 | … | lb_31 |
| --- | --- | --- | --- | --- | --- |
| r_0 | 0 | 1 | 0 |  | 0 |
| r_1 | 0 | 1 | 0 |  | 0 |
| r_2 | 1 | 1 | 1 |  | 0 |
| r_3 |  |  |  |  |  |
| r_4 |  |  |  |  |  |
| r_5 |  |  |  |  |  |
| r_6 |  |  |  |  |  |
| r_7 |  |  |  |  |  |
| r_8 |  |  |  |  |  |
| r_9 |  |  |  |  |  |
| r_10 |  |  |  |  |  |

**表 2**：port 间的 bank 冲突表（下三角；`*` 表示该位置存在两两冲突关系；`x01=1`、`x02=1` 是用表 1 示例算出的实际值）：

| req | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | * | * | * | * | * | * | * | * | * | * | * |
| 1 | x01=1 | * | * | * | * | * | * | * | * | * | * |
| 2 | x02=1 | * | * | * | * | * | * | * | * | * | * |
| 3 |  |  | * | * | * | * | * | * | * | * | * |
| 4 |  |  |  | * | * | * | * | * | * | * | * |
| 5 |  |  |  |  | * | * | * | * | * | * | * |
| 6 |  |  |  |  |  | * | * | * | * | * | * |
| 7 |  |  |  |  |  |  | * | * | * | * | * |
| 8 |  |  |  |  |  |  |  | * | * | * | * |
| 9 |  |  |  |  |  |  |  |  |  | * | * |
| 10 |  |  |  |  |  |  |  |  |  |  | * |

**表 3**：授权（grant）信号的布尔递推（`x_ji`、`xx_ji` 都是 32 bit 逐位向量，按位运算）：

| grant | req | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| g0=r_0 | 0 | * | * | * | * | * | * | * | * | * | * | * |
| g1=!x01 | 1 | xx01=x01 | * | * | * | * | * | * | * | * | * | * |
| g2=!(xx02\|xx12) | 2 | xx02=x02 | xx12=x12&g1 | * | * | * | * | * | * | * | * | * |
| g3=!(xx03\|xx13\|xx23) | 3 | xx03=x03 | xx13=x13&g1 | …&g2 | * | * | * | * | * | * | * | * |
|  | 4 | xx04=x04 | xx14=x14&g1 | …&g2 | …&g3 | * | * | * | * | * | * | * |
|  | 5 | xx05=x05 | xx15=x15&g1 | …&g2 | …&g3 | …&g4 | * | * | * | * | * | * |
|  | 6 | xx06=x06 | xx16=x16&g1 | …&g2 | …&g3 | …&g4 | …&g5 | * | * | * | * | * |
|  | 7 | xx07=x07 | xx17=x17&g1 | …&g2 | …&g3 | …&g4 | …&g5 | …&g6 | * | * | * | * |
|  | 8 | xx08=x08 | xx18=x18&g1 | …&g2 | …&g3 | …&g4 | …&g5 | …&g6 | …&g7 | * | * | * |
| g9=!(xx09\|xx19\|…\|xx89) | 9 | xx09=x09 | xx19=x19&g1 | …&g2 | …&g3 | …&g4 | …&g5 | …&g6 | …&g7 | …&g8 | * | * |
| ga=!(xx0a\|xx1a\|…\|xx9a) | 10 | xx0a=x0a | xx1a=x1a&g1 | …&g2 | …&g3 | …&g4 | …&g5 | …&g6 | …&g7 | …&g8 | …&g9 | * |

三个表与电路的对应关系如下图（对应材料 stage1 & stage2 / stage3 两张配图重绘为一张）：

![[M100 UM 仲裁逻辑.excalidraw | 900]]

- **stage1**：12 个 `Add_decoder` 对 addr0～addr11 译码，各输出 32 bit 的 `reqN[31:0]`（即表 1 的 $r_N$）。
- **stage2**：11 个 `[A&&B].iinst` 阶梯块做两两按位与，输出 66 个冲突向量 $x_{ji}=r_j \& r_i$（$0\le j<i\le 11$），即表 2 下三角、图中 11 个彩色标签条；再 OR 归约判断冲突是否存在。
- **stage3**：每列 i 有 i 个绿色 `&` 产生 $xx_{ji}=x_{ji}\&g_j$，`|` 归约或后取反输出 $g_i$；图中画出 g0～g5 与最末的 g11 列，中间省略（同材料画法）。

符号与递推的读法：

- $x_{ji}$：仅表示两端口"**有没有撞同一批 bank**"；
- $xx_{ji}=x_{ji}\&g_j$：还要高优先级端口 j **自己确实获授权**，冲突才作数；
- $g_0=r_0$：最高优先级只要请求就授权；$g_i=\overline{\bigvee_{j<i}xx_{ji}}$：没有任何"自身获授权的更高优先级端口"与 i 撞 bank 时，i 才获授权——这正是 stage3 必须级联的原因。

以表 1 示例（lb_1 上 r_0=r_1=r_2=1）串一遍：$x_{01}$ 在 lb_1 = 1、$x_{02}$ = 1、$x_{12}$ = 1；于是 g0=1；$g_1=\overline{x_{01}}=0$（1 被 0 挡住）；$g_2=\overline{x_{02}\mid(x_{12}\&g_1)}=\overline{1\mid(1\&0)}=0$（2 仍被 0 挡；1 自己都没获授权，$x_{12}=1$ 也不参与判决）。

> [!note] 材料编号口径不完全一致（记录备查）
> 端口表有 15 个 agent，优先级清单列了 10 项；表 1～表 3 用 r_0～r_10 / req 0～10（11 个请求），而电路图画了 12 个 Add_decoder（addr0～addr11）和 g0～g11（12 个请求）。仲裁器本身是参数化结构，三处数字不必强对，以"按优先级固定、级联判决"的逻辑为准。

## 5.5 Configurable hashing（地址到 bank 的可配置散列）

UM 地址位宽为 **n = 14 bit**（对应 2MB/32B 寻址空间），而 32 个 bank 需要 **m = 5 bit** 索引（$2^m=32$）。所以要把 14 bit 地址 **hash 映射成 5 bit** bank 索引。采用 bit-vector（位向量/连续位块）的设计思路：从 n bit 地址中选取连续的 m bit 块作为 bank 索引地址。按选取方式分四大类，材料给了其中三种的电路与种类数。

### 5.5.1 bit-vector permutation + bit-vector XOR

![[M100 UM hash bit-vector.excalidraw | 680]]

上图是 bit-vector XOR hash 的电路：14 位地址 a13～a0 中选出**两个连续的 m=5 位块**（图中为 a12～a8 与 a7～a3），其中一块与 **m bit 的 mask 按位与**（5 个绿色 `&`，mask 沿左侧总线送入），再与另一块**按位异或**（5 个绿色 `^`），得到 b0～b4 共 5 位 bank 索引。

**1) bit-vector permutation hash function**

基本原理：从 n bit 地址中选一个连续的 m bit 块，直接作为 $2^m$ 个 bank 的索引地址（即第二个块不用、或 mask 全 0 的退化情形）。因为只是"连续块滑动"，可选 hash function 的种类数为：

$$n-m+1 = 14-5+1 = 10 \text{ 种}$$

**2) bit-vector XOR hash function**

在 permutation 的基础上选**两个**连续的 m bit 块：其中一块与匹配的 m bit mask 按位与，再和另一块按位异或，所得 m bit 作为 bank 索引。种类数 = 不带 mask 连续块的（n−m+1）种 × 带 mask 连续块的 n 种位置 × mask 的 $2^m$ 种取值：

$$(n-m+1)\cdot n\cdot 2^m = 10\times14\times32 = 4480 \text{ 种}$$

### 5.5.2 bitwise permutation hash function

![[M100 UM hash bitwise.excalidraw | 620]]

基本原理：在 n bit 地址中**任意挑选 m bit**（不再要求连续）去索引 $2^m$ 个 bank——图中绿色梯形即多路选择：b0 是 14 选 1，b1～b4 各自独立 14 选 1。思想就是简单的随机组合，hash function 的种类数为组合数：

$$\binom{n}{m}=\frac{n!}{m!\,(n-m)!}=\binom{14}{5}=2002 \text{ 种}$$

### 5.5.3 bitwise XOR hash function

先对 n bit 地址做**两两异或**，再从异或结果中选 m bit 作为 bank 索引。本来一对 n bit 的数选 2 个 1 bit 做 XOR 有 $n^2$ 种，但 XOR 具有对称性（$a_i \oplus a_j=a_j \oplus a_i$），去掉重复后实际的异或结果数值种类如下表所示（对角为 $a_i$ 自身，右上三角为 $a_i \oplus a_j$（表中按材料原样写作 `a_i^a_j`）；右下区域材料用 `*` 标注、且 $a_i \oplus a_0$ 一列显式列出）：

|  | a13 | a12 | a11 | a10 | a9 | a8 | a7 | a6 | a5 | a4 | a3 | a2 | a1 | a0 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| a13 | **a13** | a13^a12 | a13^a11 | a13^a10 | a13^a9 | a13^a8 | a13^a7 | a13^a6 | a13^a5 | a13^a4 | a13^a3 | a13^a2 | a13^a1 | a13^a0 |
| a12 |  | **a12** | a12^a11 | a12^a10 | a12^a9 | a12^a8 | a12^a7 | a12^a6 | a12^a5 | a12^a4 | a12^a3 | a12^a2 | a12^a1 | a12^a0 |
| a11 |  |  | **a11** | a11^a10 | a11^a9 | a11^a8 | a11^a7 | a11^a6 | a11^a5 | a11^a4 | a11^a3 | a11^a2 | a11^a1 | a11^a0 |
| a10 |  |  |  | **a10** | a10^a9 | a10^a8 | a10^a7 | a10^a6 | a10^a5 | a10^a4 | a10^a3 | a10^a2 | a10^a1 | a10^a0 |
| a9 |  |  |  |  | **a9** | a9^a8 | a9^a7 | a9^a6 | a9^a5 | a9^a4 | a9^a3 | a9^a2 | a9^a1 | a9^a0 |
| a8 |  |  |  |  |  | **a8** | a8^a7 | a8^a6 | a8^a5 | a8^a4 | a8^a3 | a8^a2 | a8^a1 | a8^a0 |
| a7 |  |  |  |  |  |  | **a7** | * | * | * | * | * | * | a7^a0 |
| a6 |  |  |  |  |  |  |  | **a6** | * | * | * | * | * | a6^a0 |
| a5 |  |  |  |  |  |  |  |  | **a5** | * | * | * | * | a5^a0 |
| a4 |  |  |  |  |  |  |  |  |  | **a4** | * | * | * | a4^a0 |
| a3 |  |  |  |  |  |  |  |  |  |  | **a3** | * | * | a3^a0 |
| a2 |  |  |  |  |  |  |  |  |  |  |  | **a2** | * | a2^a0 |
| a1 |  |  |  |  |  |  |  |  |  |  |  |  | **a1** | a1^a0 |
| a0 |  |  |  |  |  |  |  |  |  |  |  |  |  | **a0** |

去重后的候选异或结果共 $n(n+1)/2$ 种（n 个自身位 + $\binom{n}{2}$ 个两两异或）：

$$\frac{n(n+1)}{2}=\frac{14\times15}{2}=105 \text{ 种}$$

最后再从这 105 个结果中随机组合选出 **m = 5 个**作为 bank 索引位，去索引 $2^m=32$ 个存储 bank（材料未给出最终组合的总数公式）。

> [!summary] 三类 hash 种类数对照（n=14, m=5）
> - bit-vector permutation：$n-m+1=10$
> - bit-vector XOR：$(n-m+1)\cdot n\cdot 2^m=4480$
> - bitwise permutation：$\binom{n}{m}=2002$
> - bitwise XOR：先得 $n(n+1)/2=105$ 个候选异或结果，再任选 m 个组合

# 6 TCC
![[M100 NPU TCC | 1000]]
![[M100 NPU TCC MAC Array | 1000]]
注意，这里在 DP4下，每8个 cycle 产生64个数据，而 NLU 也是8个 cycle 处理64个数据，因此刚好形成2级流水线