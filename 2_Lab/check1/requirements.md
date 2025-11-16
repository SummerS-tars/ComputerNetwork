你好！我是您的**计算机网络助手**，很高兴能为您提供帮助。

[cite_start]您上传了关于 **Lab Checkpoint 1: stitching substrings into a byte stream** 的实验说明。这是一个关于实现 TCP 接收端中 **重组器（Reassembler）** 核心模块的实验 [cite: 1, 2, 52, 57]。

我会帮您总结实验要求、基础知识、设计思路和实现计划。

---

## 💻 实验总结：Reassembler 实现

### 🎯 实验目标

[cite_start]本次实验的核心任务是实现 `Reassembler` 类 [cite: 57, 68][cite_start]，作为 TCP 接收端的一部分，它负责接收乱序、可能重复或丢失的字节串（substrings），并将它们重新组装成一个可靠的、有序的字节流（ByteStream） [cite: 52, 56]。

### 📝 提交要求

1.  [cite_start]**代码合并**：将 Checkpoint 0 通过的代码覆盖到 Checkpoint 1 的同名文件中，确保代码环境一致 [cite: 8]。
2.  [cite_start]**文件提交**：提交包含 `minnow` 文件夹和实验报告 `report.pdf` 的压缩包 `学号-姓名-check1.tar.gz` [cite: 9, 16]。
3.  [cite_start]**注意**：提交前务必删除 `minnow` 文件夹中的 `build` 目录 [cite: 10, 11, 12]。
4.  [cite_start]**实验报告**：`report.pdf` 中需包含测试通过的截图 [cite: 13][cite_start]，以及对代码结构、设计选择、实现挑战和剩余 Bug 的详细描述（基于 `writeups/check1.md` 模板） [cite: 50, 144, 145]。
5.  **截止日期**：
    * [cite_start]**正常提交**：2025 年 11 月 16 日 23:59 [cite: 19]。
    * [cite_start]**允许补交**：2025 年 11 月 23 日 23:59 前，得分按 $\text{实际得分} \times 0.7$ 计算 [cite: 20, 21]。

---

## 💡 基础知识回顾与关联

这个实验紧密关联了计算机网络中最核心的 **传输层** 概念——**可靠数据传输**。

| 概念 | 实验中的体现 | 知识关联与延伸 |
| :--- | :--- | :--- |
| **可靠字节流** | [cite_start]`ByteStream` 和 `Reassembler` 的目标 [cite: 26, 33, 56, 60]。 | [cite_start]**TCP** (Transmission Control Protocol) 的核心服务。TCP 将应用层数据视为一个连续的字节流，并负责确保数据的**有序**、**无损**到达 [cite: 33]。 |
| **乱序与分组** | [cite_start]TCP 发送方将字节流分割成短片段（substrings），网络可能使其**乱序**或**丢失** [cite: 54, 55]。 | [cite_start]TCP 将数据分割成**报文段（Segments）**。报文段在网络层被封装成 IP **数据报（Datagrams）**，这是**不可靠**的，可能乱序或丢失 [cite: 27, 30, 33]。|
| **字节索引** | [cite_start]每个字节都有一个唯一的**索引**，从 0 开始计数 [cite: 59, 94]。 | [cite_start]TCP 使用**序列号（Sequence Number, SN）**来标识每个字节在整个流中的位置，这是实现可靠和有序重组的基础。`first_index` 就是这个序列号的概念 [cite: 58]。 |
| **流量控制** | [cite_start]`Reassembler` 必须限制内部存储的字节数（`capacity`） [cite: 78, 80, 85]。 | 接收方需要防止发送方发送过快导致自己**缓冲区溢出**。`capacity` 概念（可用于缓冲区窗口）是 TCP **流量控制**机制的基础之一。 |

---

## 🧠 核心设计思路

[cite_start]`Reassembler` 必须有效地处理三种类别的传入字节 [cite: 73]：

1.  [cite_start]**可立即推送的字节**：如果传入的字节片段从 **下一个期望的字节索引** 开始，应立即推送到 `ByteStream` 的 `Writer` 端 [cite: 74, 75, 104, 105]。
2.  [cite_start]**需内部存储的字节**：如果字节片段位于当前已知字节之后，但在**容量限制**内，但由于**缺失了前面的字节**而不能立即推送，则必须在 `Reassembler` 内部存储（红色区域） [cite: 76, 77, 83]。
3.  [cite_start]**超出容量的字节**：位于**第一个不可接受的索引（first unacceptable index）**之后的字节，应被丢弃 [cite: 78, 86]。

### 关键状态变量

为了实现上述逻辑，您的 `Reassembler` 至少需要维护以下关键状态：

* [cite_start]**`first_unassembled_index`**：表示流中**下一个期望到达的字节**的索引。所有索引小于此值的字节都已经组装并推送到 `ByteStream` [cite: 87]。
* [cite_start]**`capacity`**：`ByteStream` 加上 `Reassembler` 内部存储的最大总容量 [cite: 85]。
* [cite_start]**`first_unacceptable_index`**：第一个**超出容量限制**的索引，即 $`first\_unassembled\_index` + \text{可用容量}$ [cite: 86]。

### 数据结构选择（性能考量）

[cite_start]由于子串（substrings）可能乱序到达、重叠，且需要高效地插入、合并和删除，因此内部存储的数据结构至关重要 [cite: 95]。

* [cite_start]**目标**：实现 $0.1 \text{ Gbit/s}$ 或更高的吞吐量 [cite: 98]。
* [cite_start]**要求**：**禁止存储重叠的子串**，以维持对容量的准确控制 [cite: 110, 111, 112]。
* **建议思路**：
    * 选择一个能够**高效查找、插入和删除**的结构，例如 `std::map` 或 `std::set`。
    * 可以将乱序的子串存储为 `<first_index, data>` 对。由于 `first_index` 必须是唯一的，这很适合作为 `std::map` 或 `std::set` 的键。
    * 在插入新子串时，必须处理它与现有子串的**重叠和合并**，确保没有重复存储字节。

---

## 🛠️ 具体实现计划

[cite_start]为了实现 `void insert(uint64_t first_index, std::string data, bool is_last_substring)` [cite: 63]，您可以遵循以下步骤：

### 步骤一：预处理和边界检查

1.  **计算子串范围**：确定传入子串的起始索引 (`first_index`) 和结束索引。
2.  **裁剪和丢弃**：
    * **检查左边界**：如果子串的一部分（或全部）的索引**小于** `first_unassembled_index`，表示这部分是重复的或已处理的。应**丢弃**这部分字节，并更新子串的起始索引和数据。
    * [cite_start]**检查右边界**：如果子串的索引**大于或等于** `first_unacceptable_index`，表示这部分超出容量。应**裁剪**子串的尾部，使其不超过 `first_unacceptable_index` [cite: 78, 79]。
3.  **空子串检查**：如果裁剪后子串变为空，则无需进一步处理。

### 步骤二：内部存储（处理乱序/缺失）

1.  [cite_start]**立即推送**：如果处理后的子串**从 `first_unassembled_index` 开始**，则将其内容推送到 `ByteStream` 的 `Writer` [cite: 74, 75]。推送成功后，更新 `first_unassembled_index`。
    * *循环检查*：在推送后，**检查内部存储**中是否有子串现在可以从新的 `first_unassembled_index` 开始拼接，如有则继续推送和更新。

2.  [cite_start]**存储乱序部分**：如果子串**不是**从 `first_unassembled_index` 开始（即它是乱序到达），且其索引在 `[first_unassembled_index, first_unacceptable_index)` 范围内，则将其插入内部存储结构 [cite: 76, 77]。
    * [cite_start]**合并重叠**：在插入前，检查新子串是否与内部存储中的现有子串**重叠**。如果重叠，必须进行合并、裁剪或替换，以确保**不存储重复的字节** [cite: 110, 111, 112]。

### 步骤三：结束标志处理

1.  [cite_start]如果传入的子串带有 `is_last_substring = true` 标志 [cite: 63]，你需要记录这个结束标志的位置。
2.  一旦所有在这个结束位置**之前**的字节都被成功推送到 `ByteStream` 中，就调用 `ByteStream` `Writer` 的相应方法来发送**流结束**信号。

### 辅助函数实现

* [cite_start]实现 `bytes_pending()` [cite: 64]：返回内部存储（红色区域）中当前存储的字节总数。

这个实验是对 TCP 核心逻辑的精彩实践。如果您在实现过程中遇到任何具体代码或概念上的难题，随时都可以向我提问！您是想让我进一步解释 **`capacity` 和 `first_unacceptable_index`** 的关系，还是想讨论 **`std::map` 作为内部存储的具体实现细节**？