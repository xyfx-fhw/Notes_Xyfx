![[M100 CCB 架构.excalidraw | 1000]]

HIB 主要负责管理、调度 Host 下发的计算任务，主要通过以下几种方式与 Host 系统交互：

1. Host 通过 Mesh Bus 访问 HIB 中的 CSR 和 SRAM；HIB Vector CPU、HIB DMA 通过 Mesh Bus 访问 DRAM。
2. 发送中断给 Host CPU。

> [!note]
> HMI 也可以发送中断给 HIB Vector CPU。

## 0.1 HIB 的组成

HIB 主要包含：

1. **2 个 HIB DMA 模块**，主要用于 NPU 和 DRAM 之间、HIB SRAM 和 Tile SRAM 之间的数据搬运。
2. **4 个 RISC-V CPU**，主要负责根据 Host 下发的推理任务，分发指令给参与计算的 Tiles。每个 CPU 同时能够运行 1 个推理任务，最大可并发运行 4 个推理任务。
3. **4 个 HIB CE（Custom Engine）**，分属 4 个 RISC-V CPU。
4. **4 个 SRAM blocks**，每个 SRAM 大小为 8 MBytes，共 32 MBytes。
5. **一个 HMI（Host Miscellaneous Interface）模块**，负责通过中断方式通知 Host CPU / HIB Vector CPU，也协助处理一些像 Barrier 之类的 Tile 间同步操作。
6. **一个 PointAcc 模块**，用于加速 LiDAR 点云运算。

## 0.2 模块接口

1. **HIB NoC 模块**提供 2 对 AXI master/slave 接口与 Mesh Bus 相连。
2. **Data Ring Bus Node（后文缩写 DRBN）**提供 2 个 Data Ring Bus 接口（1 入 / 1 出）与 Data Ring Bus 相连；内部连接 2 个 HIB DMA，这 2 个 HIB DMA 都可能是数据传输的起点，但不会是数据传输的终点。
3. **Instruction Chain Bus Node（后文缩写 ICBN）**提供 1 个 Instruction Chain Bus 接口（1 出）与 ICB 相连；内部与 4 个 HIB CE 相连，4 个 CE 都可能是数据传输的起点，但不会接受指令。
4. **HMI** 提供多个中断接口与 Host 中断管理器相连。

具体接口定义请参考对应章节。
