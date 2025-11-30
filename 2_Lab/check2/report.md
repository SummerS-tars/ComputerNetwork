# Check 2 Report: TCP Receiver & Wrapping Integers

StudentID: 23307110192  
Name: 朱文凯  

## 1. 概述 (Overview)

本阶段实现两个核心组件：

1. 序列号封装与解封装 `Wrap32` (`wrap` / `unwrap`)
2. `TCPReceiver`：接收乱序 TCP 段，将其有效负载按流索引交给 `Reassembler`，并生成正确的 `ackno` 与 `window_size`。

目标：正确处理 32 位序列号循环、SYN/FIN 占位、窗口大小与流结束、RST 错误传播。

## 2. Wrap32 设计与算法

### 2.1 问题

TCP 使用 32 位序列号，会绕回；实验内部使用 64 位绝对序列号/流索引。需要双向转换：

- 绝对序列号 → 32 位 (wrap)
- 32 位序列号 → 最接近某检查点的绝对序列号 (unwrap)

### 2.2 wrap 实现

公式：`wrapped = (zero_point + n) mod 2^32` 由于 C++ `uint32_t` 自然溢出，直接转换即可。

### 2.3 unwrap 实现

步骤：

1. 计算相对偏移：`offset = (raw - zero_point) mod 2^32`
2. 取 checkpoint 的高 32 位作为基：`candidate = (checkpoint & ~((1<<32)-1)) + offset`
3. 根据与 checkpoint 的差值，若超过半个环（2^31），向上或向下调整 `±2^32`，但避免无符号下溢。
4. 返回距离 checkpoint 最近的绝对序列号。

### 2.4 边界与修正

- 特殊用例：`seqno = UINT32_MAX`, `checkpoint = 0` — 防止错误下调导致溢出。
- 调整逻辑加入“半环距离”判断和 `>= 0` 检查。

## 3. TCPReceiver 结构与行为

### 3.1 内部状态

- `Reassembler reassembler_`：重组字节流
- `optional<Wrap32> isn_`：初始序列号（首个 SYN 的 seqno）

### 3.2 receive 处理流程

1. 若段含 `RST`：直接 `set_error()` 并返回。
2. 若首次看到 `SYN`：记录 ISN。
3. 未建立 ISN 前忽略非 SYN 数据。
4. 解封装：用 `writer().bytes_pushed() + 1` 作为 `checkpoint` 调用 `unwrap`。
5. 流索引计算：
   - 若段含 `SYN`：其 payload 起始流索引 = 0
   - 否则：`stream_index = abs_seqno - 1`（因为绝对序列号 0 为 SYN 占位）
6. 插入：`reassembler_.insert(stream_index, payload, FIN)`。

### 3.3 send 生成逻辑

- `window_size = min(writer().available_capacity(), 65535)`
- 若 ISN 未设：`ackno` 为空。
- 若 ISN 已设：
    - 下一个待接收流索引 = `writer().bytes_pushed()`
    - 绝对序列号基值 = `bytes_pushed + 1` （含 SYN）
    - 若 `writer().is_closed()`（FIN 已重组）：再 +1 确认 FIN
    - 封装：`ackno = wrap(a_first, isn_)`
- 错误：若底层流 `has_error()` 置 `RST = true`

### 3.4 FIN 与 RST 细节

- FIN 确认：只要 FIN 所在字节被完全组装（Writer 关闭），ACK 前移一个序号。
- RST：接收方若收到含 RST 段，立即标记错误；发送方读取时返回带 RST 的接收者消息。

## 4. 与 Reassembler 的协作

- 流索引不包含 SYN/FIN，占用逻辑：SYN 和（成功组装后确认的）FIN 各自占一个绝对序号。
- `writer().bytes_pushed()` 可靠表示已组装的连续字节长度，用于：
    - ack 推进
    - unwrap checkpoint 去歧义

## 5. 测试结果摘要

```bash
❯ cmake -S . -B build && cmake --build build --target check2
-- Building in 'Debug' mode.
-- Configuring done
-- Generating done
-- Build files have been written to: /usr/src/myapp/ComputerNetwork/2_Lab/check2/minnow/build
Test project /usr/src/myapp/ComputerNetwork/2_Lab/check2/minnow/build
      Start  1: compile with bug-checkers
 1/29 Test  #1: compile with bug-checkers ........   Passed    1.65 sec
      Start  3: byte_stream_basics
# ...
      Start 39: reassembler_speed_test
             Reassembler throughput: 13.03 Gbit/s
29/29 Test #39: reassembler_speed_test ...........   Passed    0.14 sec

100% tests passed, 0 tests failed out of 29

Total Test time (real) =   6.72 sec
Built target check2
```

## 6. 关键设计取舍

| 目标 | 方案 | 原因 |
|------|------|------|
| ack 不依赖应用消费 | 使用 writer.bytes_pushed | Reader 未 pop 不阻塞 ACK 前进 |
| unwrap 去歧义 | 使用高 32 位 + 半环判定 | 简单且符合最近原则 |
| FIN 处理 | 基于 writer.is_closed | 是否完成重组与应用读取解耦 |
| 错误传播 | RST 段即 set_error | 满足测试对即时错误语义 |
