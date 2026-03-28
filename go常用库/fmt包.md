`fmt` 是 Go 语言中用于**格式化输入输出**的标准库，提供了类似 C 语言 `printf` 和 `scanf` 的功能，但更安全、更易用。

---

## 1. 核心输出函数

| 函数          | 作用                                   | 是否换行 | 是否支持格式化 |
|---------------|----------------------------------------|----------|----------------|
| `Print(a ...any)`     | 将参数的默认格式输出到标准输出         | 否       | 否             |
| `Println(a ...any)`   | 输出参数，**自动添加空格**，并在末尾**换行** | 是       | 否             |
| `Printf(format string, a ...any)` | 根据**格式化字符串**输出，不自动换行    | 否       | 是             |
| `Sprintf(...)`        | 与 `Printf` 相同，但**返回字符串**，不输出 | -        | 是             |
| `Fprintf(w io.Writer, ...)` | 与 `Printf` 相同，但输出到指定的 `io.Writer` | 否       | 是             |

### 示例对比
```go
package main

import "fmt"

func main() {
    name := "Alice"
    age := 30

    // Print：直接输出，无换行
    fmt.Print("Name:", name, " Age:", age)   // Name:Alice Age:30

    // Println：自动加空格，末尾换行
    fmt.Println("Name:", name, "Age:", age)  // Name: Alice Age:30<换行>

    // Printf：格式化输出，不换行（需要显式 \n）
    fmt.Printf("Name: %s, Age: %d\n", name, age)

    // Sprintf：返回格式化的字符串
    s := fmt.Sprintf("Name: %s, Age: %d", name, age)
    fmt.Println(s)  // 输出结果同上
}
```

---

## 2. 输入函数

类似地，`fmt` 也提供了从标准输入（或 `io.Reader`）读取数据的函数：

| 函数          | 作用                                   |
|---------------|----------------------------------------|
| `Scan(a ...any)`      | 从标准输入读取，按空白分隔，存入参数   |
| `Scanln(a ...any)`    | 类似 `Scan`，但遇到换行或文件结束停止  |
| `Scanf(format string, a ...any)` | 按格式读取                           |
| `Sscan`, `Sscanln`, `Sscanf` | 从字符串读取                       |
| `Fscan`, `Fscanln`, `Fscanf` | 从 `io.Reader` 读取                |

---

## 3. 常用格式化动词（用于 `Printf` / `Sprintf` / `Fprintf`）

| 动词  | 含义                         | 示例                            |
|-------|------------------------------|---------------------------------|
| `%v`  | 默认格式（值）               | `fmt.Printf("%v", user)`        |
| `%+v` | 结构体时包含字段名           | `{Name:Alice Age:30}`           |
| `%#v` | Go 语法表示（类似代码）      | `main.Person{Name:"Alice", Age:30}` |
| `%T`  | 类型                         | `main.Person`                   |
| `%d`  | 十进制整数                   | `30`                            |
| `%s`  | 字符串                       | `"Alice"`                       |
| `%q`  | 带引号的字符串               | `"Alice"`                       |
| `%f`  | 浮点数                       | `3.141593`                      |
| `%t`  | 布尔值                       | `true` / `false`                |
| `%p`  | 指针地址                     | `0xc000010200`                  |

更多细节可参考 `fmt` 文档。

---

## 4. 为什么需要 `Sprintf`？

`Sprintf` 与 `Printf` 的唯一区别是**它不直接输出，而是返回格式化的字符串**。当你需要构造一个字符串用于后续处理（如日志、网络传输、拼接复杂文本）时，`Sprintf` 非常方便。

```go
// 错误用法：想拼接字符串却直接输出
fmt.Printf("User %s logged in", name)  // 直接输出到控制台

// 正确用法：保存到变量
msg := fmt.Sprintf("User %s logged in", name)
log.Println(msg)  // 或写入文件等
```

---

## 5. 其他重要特性

- **自动处理 `error`**：`fmt` 会调用 `Error()` 方法（若实现了 `error` 接口）来输出错误信息。
- **`Stringer` 接口**：如果类型实现了 `String() string` 方法，`%v` 等动词会使用该方法返回的字符串。
- **宽度和精度**：可以在格式化动词中指定，例如 `%10s`（右对齐，宽度10）、`%.2f`（保留两位小数）。

---

## 总结

- `Print` / `Println` / `Printf` 用于直接输出到标准输出。
- `Sprintf` 用于生成格式化的字符串，**不输出**，适合字符串拼接、日志构造等场景。
- `Fprintf` 用于将格式化内容写入任意 `io.Writer`（如文件、网络连接）。
