# Check 1 Report

StudentID: 23307110192  
Name: 朱文凯  

## 1. Part 1: Understanding TCP Reassembly

### 1.1. Problem Overview

**The Challenge**: TCP receives data as independent packets (substrings) that may arrive out-of-order, contain duplicates, or have gaps. The Reassembler class must convert these unreliable segments into a reliable, ordered byte stream.

**Core Concept**:  

- Network layer: unreliable datagrams (may lose, reorder, duplicate data)  
- TCP layer: must provide reliable byte streams to applications  
- Reassembler: buffers out-of-order data and delivers in-order bytes

## 2. Part 2: Reassembler Structure and Design

### 2.1. Data Structures Chosen

The key design decision is the internal data structure for buffering:

```cpp
private:
  // Map automatically maintains sorted order by index
  std::map<uint64_t, std::string> buffer_ {};
  
  // Track the next expected byte to be assembled
  uint64_t first_unassembled_index_ { 0 };
  
  // Maintain count of pending bytes (for efficiency)
  uint64_t bytes_pending_ { 0 };
  
  // Stream termination tracking
  bool has_last_ { false };
  uint64_t last_index_ { 0 };
```

**Why std::map?**  

- Automatically maintains sorted order by index
- O(log n) insertion, deletion, lookup operations
- Natural semantic fit: one substring per starting index
- Enables efficient overlap detection and merging

### 2.2. Three-Stage Processing Pipeline

The insert() function follows a clear three-stage approach:

**Stage 1: Boundary Clipping**  

```txt
Input: first_index, data, is_last_substring
  ↓
Clip left boundary   (remove already-assembled bytes)
Clip right boundary  (remove bytes beyond capacity)
Track stream end     (if is_last_substring = true)
  ↓
Output: clipped data or early return if empty
```

**Stage 2: Immediate Push or Buffer**  

```txt
If first_index == first_unassembled_index_:
    ↓
    Push to ByteStream immediately
    Update first_unassembled_index_
    Check buffer_ for consecutive data
Else:
    ↓
    Store in buffer_ (with overlap handling)
```

**Stage 3: Overlap Resolution and Merge**  

```txt
Before inserting:
    Check backward (previous entry in map)
    Check forward (all overlapping entries)
    Merge regions (no duplicate bytes)
    Update bytes_pending_
```

### 2.3. Implementation Code Structure

Key function implementation excerpt:

```cpp
void Reassembler::insert(uint64_t first_index, string data, 
                         bool is_last_substring)
{
  // Handle last substring flag
  if (is_last_substring) {
    has_last_ = true;
    last_index_ = first_index + data.length();
  }

  // Calculate boundaries for clipping
  uint64_t end_index = first_index + data.length();
  uint64_t first_unacceptable_index = 
      first_unassembled_index_ + output_.writer().available_capacity();

  // Clip left: discard already-assembled bytes
  if (first_index < first_unassembled_index_) {
    uint64_t offset = first_unassembled_index_ - first_index;
    data = data.substr(offset);
    first_index = first_unassembled_index_;
  }

  // Clip right: discard bytes beyond capacity
  if (first_index >= first_unacceptable_index) {
    return;  // Entire substring beyond capacity
  }
  if (end_index > first_unacceptable_index) {
    uint64_t new_length = first_unacceptable_index - first_index;
    data = data.substr(0, new_length);
  }

  // Try immediate push
  if (first_index == first_unassembled_index_) {
    output_.writer().push(data);
    first_unassembled_index_ += data.length();

    // Cascading push: check buffer for consecutive data
    while (!buffer_.empty()) {
      auto it = buffer_.begin();
      if (it->first == first_unassembled_index_) {
        output_.writer().push(it->second);
        first_unassembled_index_ += it->second.length();
        bytes_pending_ -= it->second.length();
        buffer_.erase(it);
      } else {
        break;  // Gap exists
      }
    }
  } else if (first_index > first_unassembled_index_) {
    // Out-of-order: buffer with overlap handling
    // Check and merge with existing entries...
  }

  // Check if stream should close
  if (has_last_ && first_unassembled_index_ >= last_index_) {
    output_.writer().close();
  }
}
```

## 3. Part 3: Implementation Challenges

### 3.1. Challenge 1: Handling Overlapping Substrings

**Problem**: When a new substring overlaps with one or more existing buffered entries, we must merge them without storing duplicate bytes.

**Solution Strategy**:  

```txt
1. Find insertion point using lower_bound()
2. Check previous entry for overlap
3. Check all forward entries for overlap
4. Merge regions: combine data, extend ranges
5. Update bytes_pending_ for each merge
```

### 3.2. Challenge 2: Capacity Management

**Problem**: Must ensure total bytes (ByteStream buffer + Reassembler buffer) never exceed capacity.

**Solution**:  

```cpp
// Calculate BEFORE any modifications
uint64_t first_unacceptable_index = 
    first_unassembled_index_ + output_.writer().available_capacity();

// Discard anything at or beyond this index
if (end_index > first_unacceptable_index) {
  data = data.substr(0, first_unacceptable_index - first_index);
}
```

### 3.3. Challenge 3: Stream Closure Timing

**Problem**: Cannot close stream immediately when is_last_substring=true. Must wait for ALL bytes up to last_index_ to be assembled.

**Solution**:  

```cpp
// Track stream end separately
if (is_last_substring) {
  has_last_ = true;
  last_index_ = first_index + data.length();
}

// Check after EVERY push operation
if (has_last_ && first_unassembled_index_ >= last_index_) {
  output_.writer().close();
}
```

## 4. Part 4: Test Results

```bash
❯ cmake --build build --target check1
Test project /usr/src/myapp/ComputerNetwork/2_Lab/check1/minnow/build
      Start  1: compile with bug-checkers
 1/17 Test  #1: compile with bug-checkers ........   Passed    1.01 sec
      Start  3: byte_stream_basics
 2/17 Test  #3: byte_stream_basics ...............   Passed    0.01 sec
      Start  4: byte_stream_capacity
 3/17 Test  #4: byte_stream_capacity .............   Passed    0.01 sec
      Start  5: byte_stream_one_write
 4/17 Test  #5: byte_stream_one_write ............   Passed    0.01 sec
      Start  6: byte_stream_two_writes
 5/17 Test  #6: byte_stream_two_writes ...........   Passed    0.01 sec
      Start  7: byte_stream_many_writes
 6/17 Test  #7: byte_stream_many_writes ..........   Passed    0.03 sec
      Start  8: byte_stream_stress_test
 7/17 Test  #8: byte_stream_stress_test ..........   Passed    0.01 sec
      Start  9: reassembler_single
 8/17 Test  #9: reassembler_single ...............   Passed    0.01 sec
      Start 10: reassembler_cap
 9/17 Test #10: reassembler_cap ..................   Passed    0.01 sec
      Start 11: reassembler_seq
10/17 Test #11: reassembler_seq ..................   Passed    0.01 sec
      Start 12: reassembler_dup
11/17 Test #12: reassembler_dup ..................   Passed    0.02 sec
      Start 13: reassembler_holes
12/17 Test #13: reassembler_holes ................   Passed    0.01 sec
      Start 14: reassembler_overlapping
13/17 Test #14: reassembler_overlapping ..........   Passed    0.01 sec
      Start 15: reassembler_win
14/17 Test #15: reassembler_win ..................   Passed    0.15 sec
      Start 37: compile with optimization
15/17 Test #37: compile with optimization ........   Passed    2.53 sec
      Start 38: byte_stream_speed_test
             ByteStream throughput: 2.42 Gbit/s
16/17 Test #38: byte_stream_speed_test ...........   Passed    0.11 sec
      Start 39: reassembler_speed_test
             Reassembler throughput: 15.36 Gbit/s
17/17 Test #39: reassembler_speed_test ...........   Passed    0.13 sec

100% tests passed, 0 tests failed out of 17

Total Test time (real) =   4.06 sec
Built target check1
```

### 4.1. Performance Analysis

- **All 17 tests pass** without any failures
- **Reassembler throughput: 15.36 Gbit/s** (exceeds 0.1 Gbit/s requirement by 150x)
- **ByteStream throughput: 2.42 Gbit/s** (solid baseline)
- Handles all edge cases: capacity limits, overlaps, duplicates, gaps

## 5. Part 5: Key Design Insights

### 5.1. Cascading Push Pattern

After pushing immediate data, we loop through the buffer_ checking if newly contiguous regions can now be pushed. This "waterfall" approach maximizes throughput for out-of-order arrivals:

```cpp
// Cascading push through buffer
while (!buffer_.empty()) {
  auto it = buffer_.begin();
  if (it->first == first_unassembled_index_) {
    // This segment is now contiguous, push it!
    output_.writer().push(it->second);
    first_unassembled_index_ += it->second.length();
    bytes_pending_ -= it->second.length();
    buffer_.erase(it);
  } else {
    break;  // Gap detected, stop
  }
}
```

### 5.2. Efficient Boundary Calculations

Key insight: Calculate first_unacceptable_index BEFORE any data modifications to ensure consistent capacity enforcement:

```cpp
// This calculation happens ONCE per insert()
uint64_t first_unacceptable_index = 
    first_unassembled_index_ + output_.writer().available_capacity();

// Then all clipping uses this value
if (end_index > first_unacceptable_index) {
  // Clip to not exceed capacity
}
```

## 6. Part 6: Code Quality

- ✅ Passes clang-tidy static analysis (compile with bug-checkers)
- ✅ Formatted with clang-format for consistency
- ✅ Zero compiler warnings or errors
- ✅ Well-commented explaining complex overlap logic

## 7. Part 7: Why This Design Works

**Simplicity**: Three-stage pipeline is easy to understand and reason about

**Correctness**: Handles all edge cases systematically:  

- Empty substrings ✓
- Fully redundant substrings ✓
- Capacity violations ✓
- Complex overlaps ✓
- Out-of-order delivery ✓
- Stream termination ✓

**Performance**: O(log n) per operation with excellent empirical throughput

**Maintainability**: Clear code structure with minimal special cases
