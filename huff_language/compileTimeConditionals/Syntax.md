# Huff `if` Conditional — Complete Reference

## Table of Contents

- [Syntax](#syntax)
- [Features](#features)
- [Limitations](#limitations)

---

## Syntax

```huff
if (condition) {
    // statements
}
```

```huff
if (condition) {
    // statements
} else {
    // statements
}
```

```huff
if (condition) {
    // statements
} else if (condition2) {
    // statements
} else {
    // statements
}
```

---

## Features

- **Compile-time expansion:** Statements được expand trong quá trình compilation và không tồn tại trong bytecode cuối cùng. Compiler evaluate condition thành `true`/`false` lúc compile để quyết định có emit code trong block hay không.
- **Constant expressions:** Conditions phải là constant expressions, tính toán được tại compile-time.
- **Comparison operators:** `==`, `!=`, `<`, `>`, `<=`, `>=`
- **Logical NOT:** `!` operator
- **Nested conditionals:** `if` statements có thể lồng nhau.

---

## Limitations

### 1. Constant Expressions Only

Conditions phải evaluate được tại compile-time. Runtime values không hợp lệ:

```huff
// ✓ Valid — compile-time constants
if ([CONSTANT] > 5) { }
if (<arg> == 0x03) { }
if ([CONST_A] + [CONST_B] < 0xFF) { }

// ✗ Invalid — runtime value, không biết lúc compile
if (calldataload(0x00) > 5) { }
```

### 2. Label Scoping

Labels trong một branch không thể reference từ branch khác:

```huff
// ✗ Invalid — success không visible trong else branch
if ([FLAG]) {
    success:
        0x01
} else {
    success jump  // Error: success not visible
}
```

Mỗi branch là một scope riêng biệt. Compiler chỉ emit code cho branch thắng condition — branch còn lại bị loại bỏ hoàn toàn, kéo theo labels bên trong cũng không tồn tại.

### 3. No Side Effects

Conditions phải là **pure expressions** — không được gọi macro:

```huff
// ✗ Invalid — macro có thể tạo side effects trên stack
if (some_macro()) { }

// ✓ Valid — pure expressions
if (<i> == 0x03) { }
if ([CONST] > 0x10) { }
if (<start> + <end> < 0xFF) { }
```

`if` trong Huff là **compile-time construct** (preprocessor directive), không phải runtime `JUMPI`. Compiler cần evaluate condition thành `true`/`false` lúc compile. Macro thì chỉ evaluate được lúc expand — nó push opcodes lên bytecode, không trả về giá trị compile-time.