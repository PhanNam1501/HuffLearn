# Huff `for` Loop — Complete Reference

## Table of Contents

- [Syntax](#syntax)
- [Features](#features)
- [Basic Usage](#basic-usage)
- [Variable Interpolation](#variable-interpolation)
- [Loop Bounds](#loop-bounds)
- [Common Use Cases](#common-use-cases)

---

## Syntax

```huff
for(variable in start..end) {
    // loop body
}

for(variable in start..end step N) {
    // loop body with custom step
}
```

---

## Features

- **Compile-time expansion:** Loops được expand hoàn toàn trong quá trình compilation và không tồn tại trong bytecode cuối cùng.
- **Variable interpolation:** Dùng `<variable>` để chèn giá trị iteration hiện tại dưới dạng hex literal.
- **Constant expressions:** Loop bounds có thể sử dụng constants và arithmetic.
- **Nested loops:** Loops có thể lồng nhau.

---

## Basic Usage

### Simple Repetition

Tạo một chuỗi literal values:

```huff
#define macro PUSH_SEQUENCE() = takes(0) returns(3) {
    for(i in 0..3) {
        <i>
    }
    // Expands to: 0x00 0x01 0x02
}
```

### With Opcodes

Sử dụng loop variable với explicit opcodes:

```huff
#define macro PUSH_VALUES() = takes(0) returns(3) {
    for(i in 1..4) {
        push1 <i>
    }
    // Expands to: push1 0x01 push1 0x02 push1 0x03
}
```

### Custom Step

Dùng step value khác 1:

```huff
#define macro EVEN_NUMBERS() = takes(0) returns(5) {
    for(i in 0..10 step 2) {
        <i>
    }
    // Expands to: 0x00 0x02 0x04 0x06 0x08
}
```

---

## Variable Interpolation

### Cơ chế hoạt động

Cú pháp `<variable>` sẽ **thay thế biến loop bằng giá trị hex literal** tại mỗi vòng lặp. Nó hoạt động giống hệt cách `<arg>` dùng để truyền macro arguments trong Huff.

```huff
for(i in 0..3) {
    <i>
}
```

Compiler sẽ **unroll** (mở vòng lặp) thành:

```
0x00    // i = 0
0x01    // i = 1
0x02    // i = 2
```

Về bản chất, `<i>` chỉ là **text substitution** — compiler thay `<i>` bằng giá trị hex tương ứng trước khi compile bytecode.

### Shadowing

Nếu macro có argument tên `i` và loop cũng dùng biến `i`, thì **trong scope của loop, `<i>` luôn là loop variable**, không phải macro arg:

```huff
#define macro FOO(i) = takes(0) returns(0) {
    <i>              // → macro argument i
    for(i in 0..2) {
        <i>          // → loop variable (0x00, 0x01), NOT macro arg
    }
    <i>              // → macro argument i (trở về bình thường)
}
```

### Limitations

`<variable>` chỉ là **literal substitution** — nó chèn một giá trị hex vào vị trí cần một giá trị. Nó **không phải string concatenation** cho identifiers.

**Được hỗ trợ** — vì đây là vị trí cần literal value:

```huff
<i>         // push giá trị i lên stack
push1 <i>   // tường minh push1 với giá trị i
```

**Không được hỗ trợ** — vì compiler không làm string interpolation cho tên:

```huff
[SLOT_<i>]   // ✗ Không thể ghép "SLOT_" + "0" thành constant name "SLOT_0"
label_<i>:   // ✗ Không thể ghép "label_" + "0" thành label name "label_0"
```

> **Nói đơn giản:** `<i>` cho ra một **con số** (hex literal), không phải một **chuỗi ký tự** để ghép vào tên. Nếu cần access `SLOT_0`, `SLOT_1`, `SLOT_2` trong loop, bạn phải tìm cách khác (ví dụ tính offset bằng arithmetic trên stack thay vì dùng constant name).

---

## Loop Bounds

### Nguyên tắc cốt lõi

Loop bounds **phải xác định được tại compile-time**. Huff compiler cần biết chính xác số vòng lặp để unroll hoàn toàn thành bytecode — không có runtime loop trong EVM bytecode output.

### 1. Constant References — bắt buộc dùng `[BRACKET]`

```huff
#define constant END = 10

for(i in 0..[END]) { }   // ✓ Correct — bracket notation required
for(i in 0..END) { }     // ✗ Incorrect — bare constant name not allowed
```

**Tại sao?** Trong Huff, `[NAME]` là cú pháp chuẩn để dereference constant. Bare name `END` không có ý nghĩa trong context này — compiler cần `[]` để biết đây là constant lookup, không phải label hay identifier khác.

Áp dụng ở **mọi vị trí**:

- Start bound: `for(i in [START]..end)`
- End bound: `for(i in start..[END])`
- Step value: `for(i in start..end step [STEP])`

### 2. U256 Range & Hex Notation

Vì EVM làm việc với 256-bit words, loop bounds hỗ trợ toàn bộ dải u256:

```huff
for(i in 0x00..0x100 step 0x20) {
    <i>
}
// Unroll: 0x00, 0x20, 0x40, 0x60, 0x80, 0xA0, 0xC0, 0xE0
// (8 iterations)
```

Thực tế thường dùng cho việc lặp qua **storage slots** hoặc **memory offsets** theo bước cố định (ví dụ `0x20` = 32 bytes = 1 word).

Dùng với constants:

```huff
#define constant START = 0x00
#define constant END = 0xFF
#define constant STEP = 0x10

#define macro USE_CONSTANTS() = takes(0) returns(16) {
    for(i in [START]..[END] step [STEP]) {
        <i>
    }
}
```

Hỗ trợ full u256 range:

```
// Works with any u256 value up to:
// 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
```

### 3. Arithmetic in Bounds

Compiler evaluate biểu thức arithmetic tại compile-time:

```huff
#define constant SIZE = 5

for(i in 0..[SIZE] * 2) {
    <i>
}
// [SIZE] * 2 = 10, loop từ 0 đến 9
```

Đây là **compile-time constant folding** — kết quả phải là số cụ thể trước khi unroll.

### 4. Macro Arguments in Bounds

Đây là tính năng mạnh nhất — cho phép tạo **generic, reusable loop macros**.

**Basic usage:**

```huff
#define macro LOOP(count) = takes(0) returns(0) {
    for(i in 0..<count>) {
        <i>
    }
}

#define macro MAIN() = takes(0) returns(0) {
    LOOP(0x03)  // Expands to: 0x00 0x01 0x02
}
```

**Quy trình compiler:**

1. `LOOP(0x03)` → thay `<count>` = `0x03`
2. Thành `for(i in 0..0x03)`
3. Unroll thành `0x00 0x01 0x02`

**With start and end arguments:**

```huff
#define macro RANGE(start, end) = takes(0) returns(0) {
    for(i in <start>..<end>) {
        <i>
    }
}

#define macro MAIN() = takes(0) returns(0) {
    RANGE(0x05, 0x08)  // Expands to: 0x05 0x06 0x07
}
```

**With step argument:**

```huff
#define macro STEPPED(count, step) = takes(0) returns(0) {
    for(i in 0..<count> step <step>) {
        <i>
    }
}

#define macro MAIN() = takes(0) returns(0) {
    STEPPED(0x0A, 0x02)  // Expands to: 0x00 0x02 0x04 0x06 0x08
}
```

**With arithmetic:**

```huff
#define macro COMPUTED(base, extra) = takes(0) returns(0) {
    for(i in 0..(<base> + <extra>)) {
        <i>
    }
}

#define macro MAIN() = takes(0) returns(0) {
    COMPUTED(0x02, 0x03)  // Loop from 0 to 5
}
```

**Mixed with constants:**

```huff
#define constant BASE = 0x10

#define macro ADD_RANGE(offset) = takes(0) returns(0) {
    for(i in [BASE]..([BASE] + <offset>)) {
        <i>
    }
}

#define macro MAIN() = takes(0) returns(0) {
    ADD_RANGE(0x05)  // Loop from 0x10 to 0x15
}
```

### 5. Iteration Limit: 10,000

```huff
// This will error — exceeds 10,000 iterations
for(i in 0..0x10000) {  // 65,536 iterations
    <i>
}
```

**Tại sao cần limit?** Vì loop được **unroll hoàn toàn** thành bytecode. 65,536 iterations = ít nhất 65,536 PUSH instructions = bytecode khổng lồ. Limit 10,000 là safety guard chống:

- Compile time quá lâu
- Bytecode vượt quá contract size limit (24KB trên Ethereum)
- OOM errors khi compile

> **Thực tế:** Nếu bạn cần >10,000 iterations, hầu như chắc chắn bạn nên dùng runtime loop (JUMPI pattern) thay vì compile-time unroll.

---

## Common Use Cases

### 1. Initialize Storage Slots

```huff
#define macro INIT_STORAGE() = takes(0) returns(0) {
    for(slot in 0..10) {
        0x00 <slot> sstore  // Store zero at each slot
    }
}
```

Unroll thành:

```
0x00 0x00 sstore   // slot 0 = 0
0x00 0x01 sstore   // slot 1 = 0
0x00 0x02 sstore   // slot 2 = 0
...
0x00 0x09 sstore   // slot 9 = 0
```

**Use case:** Reset hoặc initialize nhiều storage slots liên tiếp. Ví dụ khi deploy contract cần clear một vùng storage, hoặc init default values cho một array cố định. Không cần runtime loop logic (`JUMP`/`JUMPI`), tiết kiệm gas overhead.

### 2. Generate Function Selector Table

```huff
#define macro SELECTOR_TABLE() = takes(0) returns(0) {
    for(idx in 0..5) {
        dup1
        <idx>
        eq
        handler jumpi
    }
}
```

Unroll thành:

```
dup1 0x00 eq handler jumpi
dup1 0x01 eq handler jumpi
dup1 0x02 eq handler jumpi
dup1 0x03 eq handler jumpi
dup1 0x04 eq handler jumpi
```

**Use case:** Tạo dispatch table — so sánh giá trị trên stack với từng index rồi jump đến handler. Thực tế trong Huff, function dispatching thường so sánh `msg.sig` (4-byte selector) với các function signature. Ví dụ này đơn giản hóa bằng index, nhưng pattern tương tự dùng cho selector matching.

### 3. Compile-time Unroll vs Runtime Loop

```huff
// Runtime loop — mỗi iteration tốn gas cho JUMP + condition check
loop:
    // ... logic
    // kiểm tra điều kiện
    loop jumpi    // jump ngược lại → tốn gas mỗi lần

// Compile-time unroll — không có jump overhead
for(i in 0..[COUNT]) {
    // ... logic được copy ra COUNT lần
}
```

**So sánh:**

|                        | Runtime loop              | Compile-time unroll          |
| ---------------------- | ------------------------- | ---------------------------- |
| **Bytecode size**      | Nhỏ (1 copy + jump)      | Lớn (N copies)               |
| **Gas per iteration**  | Cao hơn (JUMP + JUMPI)   | Thấp hơn (no jump overhead)  |
| **Flexibility**        | N có thể dynamic          | N phải biết lúc compile      |

**Khi nào nên unroll?** Khi số iterations nhỏ và cố định, và bạn muốn tối ưu gas tối đa. Đây là lý do chính `for` loop tồn tại trong Huff — nó là **compile-time code generation tool**, không phải runtime control flow. Bạn đánh đổi bytecode size để lấy gas efficiency.