# png2ascii

将 PNG 图像转换为 ASCII 字符画并直接输出到终端。

---

## 目录

- [关于项目](#关于项目)
- [构建](#构建)
- [用法](#用法)
- [实现原理](#实现原理)
- [贡献指南](#贡献指南)
- [代码风格](#代码风格)
- [许可证](#许可证)

---

## 关于项目

本项目主要为 SimbaRobotics 机器人实验室队内同学练习 Github 的基本协作操作、代码提交流程和基本编码训练所设。
请参考构建、贡献指南和代码风格来完成文档中的任务。仓库中已上传了一张影绘作为贡献或构建完成的测试图片。
本项目没有第三方依赖，满足 C++17 要求的编译器足以完成所有的任务要求。

---

## 构建

### 依赖要求

| 项目 | 最低版本 |
|------|----------|
| C++ 编译器 | C++17（GCC 8+，Clang 7+，MSVC 2017+） |
| 构建工具 | 任意（可直接用编译器，或配合 XMake） |

### 使用 XMake（推荐）

```bash
xmake build
```

### 直接编译

```bash
g++ -O2 -std=c++17 -o png2ascii png2ascii.cpp
```

使用 Clang：

```bash
clang++ -O2 -std=c++17 -o png2ascii png2ascii.cpp
```

Windows（MSVC）：

```bat
cl /O2 /std:c++17 /EHsc png2ascii.cpp /Fe:png2ascii.exe
```

---

## 用法

```
png2ascii [选项] image.png

选项：
  -w <int>     输出宽度（字符数，默认：80）
  -H <int>     输出高度（字符数，0 = 按比例自动计算，默认：0）
  -s <style>   字符集风格（默认：standard）
  -i           反转明暗
  -c           ANSI 24-bit 彩色输出
  --help       显示帮助
```

### 字符集风格

| 风格 | 字符集示例 | 适用场景 |
|------|-----------|----------|
| `standard` | ` .,:;i1tfLCG08@#` | 通用，层次均匀 |
| `dense` | ` .'、^":;Il!i><~+…MW&8%B@$` | 细节丰富，宽屏场景 |
| `simple` | ` -+#@` | 简洁，适合图标/徽标 |
| `binary` | ` @` | 纯黑白剪影效果 |

### 示例

```bash
# 默认（80 列，standard 风格）
./png2ascii photo.png

# 宽屏 dense 风格
./png2ascii -w 160 -s dense photo.png

# 固定 60×30 输出区域
./png2ascii -w 60 -H 30 photo.png

# 反色（适合深色图片或浅色背景终端）
./png2ascii -i logo.png

# 24-bit 真彩色（需终端支持）
./png2ascii -w 120 -c photo.png

# 组合使用
./png2ascii -w 100 -s dense -i -c portrait.png
```

> **终端字体提示**：字符画的宽高比取决于所用终端字体。常见等宽字体的字符高宽比约为 2:1，程序默认自动补偿（`-H 0` 时）。如输出比例失真，可手动传入 `-H` 调整。

---

## 实现原理

整个实现分为三个自包含的层次，均在单一源文件 `src/main.cpp` 中。

### 1. deflate 解压器

PNG 的像素数据以 zlib/deflate 格式压缩（RFC 1950 / RFC 1951）。完整的解压逻辑是：

- **BitReader**：LSB 优先的位流读取器，支持字节对齐
- **Huffman**：基于码长表构建解码树（`build()`），逐位匹配解码（`decode()`）
- **inflate()**：处理三种块类型——BTYPE 0（非压缩）、BTYPE 1（固定 Huffman）、BTYPE 2（动态 Huffman），以及长度/距离回指复制

### 2. PNG 解码器

按照 PNG 规范（ISO/IEC 15948）解析文件结构：

- 校验 8 字节文件签名
- 逐块读取 `IHDR`（图像元信息）、`PLTE`（调色板）、`IDAT`（压缩数据）、`IEND`
- 拼接多个 IDAT chunk 后统一解压
- 重建五种行过滤器：None / Sub / Up / Average / Paeth
- 将所有格式（灰度、RGB、RGBA、索引色、16-bit）归一化为 8-bit 每通道 RGB

### 3. ASCII 渲染

- 按目标字符网格划分源图像，对每个单元格内的所有像素取平均
- 用 ITU-R BT.601 系数（`0.299R + 0.587G + 0.114B`）计算感知亮度
- 将亮度线性映射到字符集索引，可选反转
- 彩色模式下输出 `\033[38;2;R;G;Bm` ANSI 转义序列

---

## 贡献指南

欢迎任何形式的贡献！在提交之前，请先阅读以下规则。

### 提交流程

1. Fork 本仓库，并从 `main` 创建功能分支：
   ```bash
   git checkout -b feature/your-feature-name
   ```
2. 编写代码，确保通过编译且不引入警告（见下方编译命令）
3. 本地测试覆盖你的改动（见[测试](#测试)一节）
4. 提交时使用清晰的提交信息（见下方格式）
5. 发起 Pull Request，描述改动动机与测试方式

### 提交信息格式

```
<类型>: <简短描述（≤72字）>

[可选正文：详细说明改动原因、设计决策等]
```

类型取值：`feat`（新功能）、`fix`（修复）、`refactor`（重构）、`docs`（文档）、`test`（测试）、`chore`（杂项）。

示例：
```
feat: add support for PNG interlacing (Adam7)

Implements the Adam7 deinterlacing pass reorder so that
progressive PNGs can be rendered correctly.
```

### 测试

目前没有正式的测试框架。贡献者应手动验证以下场景，并在 PR 描述中注明：

- 改动涉及解码的，请用覆盖所有颜色模式（灰度、RGB、RGBA、索引色）的 PNG 文件测试
- 改动涉及渲染的，请分别测试四种 `-s` 风格和 `-c` 彩色模式
- 改动涉及参数解析的，请测试边界输入（超大/超小宽高、非法风格名等）

欢迎贡献自动化测试。

### 欢迎的贡献类型

- PNG 格式扩展支持（Adam7 交错扫描、tRNS 透明度块、16-bit 全精度渲染）
- 抖动（dithering）算法支持
- 输出到文件（`-o output.txt`）
- 更多字符集风格或自定义字符集（`--charset " .#"`）
- 性能优化（Huffman 查找表替代 `std::map`）
- 在 XMake 基础上的兼容性 CMakeLists.txt / Makefile / GitHub Actions CI 配置

### 不欢迎的改动

- 引入任何第三方库
- 大规模重构而无明确收益
- 降低单文件可读性的过度模块化拆分（大约 500 行为界）

---

## 代码风格

本项目遵循以下风格规范。如无特殊说明，以现有代码风格为准。

### 基础规范

- **标准**：C++17，不使用任何非标准扩展
- **缩进**：4 个空格
- **行宽**：软限制 100 列，超出不强制折行，但应保持可读
- **编码**：UTF-8，仅在注释和字符串字面量中使用非 ASCII 字符

### 命名约定

| 实体 | 风格 | 示例 |
|------|------|------|
| 类型（struct/class/using） | `PascalCase` | `BitReader`, `Huffman` |
| 函数 | `snake_case` | `inflate_codes()`, `load_png()` |
| 局部变量 | `snake_case` | `out_row`, `final_block` |
| 常量 / 全局数组 | `SCREAMING_SNAKE_CASE` | `LBASE`, `CS_DENSE` |
| 类型别名（基础整数） | 简短小写 | `u8`, `u16`, `u32` |

### 注释

- 函数级注释写在定义前，简述用途，复杂逻辑在行内标注
- 引用规范文档时注明来源：`// RFC 1951 §3.2.7`

### 结构约定

- 类型和结构体的成员按：字段 → 构造函数 → 方法的顺序排列
- 相关函数组之间用 `// ──── 标题 ────` 分隔注释划分区块
- 优先使用 `static` 函数而非匿名命名空间（保持文件作用域清晰）
- 使用 `std::runtime_error` 报告解析错误，不使用返回码

### 警告与健壮性

提交前请确保以下命令无任何警告输出：

推荐使用 XMake ：
```bash
xmake build
```

或是直接编译 ：
```bash
g++ -O2 -std=c++17 -Wall -Wextra -Wshadow -Wpedantic -o png2ascii png2ascii.cpp
```

- 不使用 `reinterpret_cast` 或 C 风格强制转换（用 `static_cast` 或命名转换）
- 不使用裸指针管理所有权（用 `std::vector` 管理缓冲区）
- 所有外部输入（文件读取、参数解析）必须有边界检查

---

## 许可证

AGPLv3.0 License — 详见 [LICENSE](./LICENSE) 文件。