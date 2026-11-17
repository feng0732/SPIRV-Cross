# SPIRV-Cross 中着色缓冲的 GLSL / HLSL 布局表达分析

本文分析本仓库（commit `57e4efc`）的 GLSL 与 HLSL 后端如何把同一份 SPIR-V 布局装饰
（`Offset` / `ArrayStride` / `MatrixStride` / `RowMajor` / `ColMajor`）表达为各自的源语言声明，
覆盖标量数组、矩阵数组、嵌套结构体，以及行/列主序。所有结论都由**本机编译出的 `spirv-cross`
CLI + 手工构造/编译的 SPIR-V 实测**得出，不依赖 GPU 驱动。

---

## 1. 结论摘要

| 维度 | GLSL 后端 | HLSL 后端 |
|---|---|---|
| 表达手段 | `layout(std140/std430/scalar)` 块级限定符；必要时对**顶层块成员**追加 `layout(offset=N)`；`layout(row_major)` | cbuffer 路径：结构体声明 + `: packoffset(cN[.x/y/z/w])`；SSBO 路径：`ByteAddressBuffer` + 显式**字节**偏移 |
| 偏移来源 | 由所选打包标准**隐式**推导；只有不满足标准时才显式写 `offset`。变量级 `Offset` 装饰也会原样输出（`2405-2406`） | cbuffer：只有**起始偏移**可被 `packoffset` 固定；SSBO：直接使用装饰中的字节数 |
| 数组步长 | 由标准规则隐式决定（std140 最小 16，std430 依元素） | cbuffer：由 HLSL 规则隐式决定；SSBO：使用 SPIR-V `ArrayStride` |
| 矩阵步长 | 由标准规则隐式决定，**无法**表达任意 `MatrixStride` | cbuffer：由 HLSL 规则隐式决定（`row_major`/`column_major`），**无法**表达任意 `MatrixStride`；SSBO：使用 SPIR-V `MatrixStride` |
| 行列主序 | 成员级只输出 `row_major`（列主序为默认，`1584-1586` 的 `column_major` 分支被注释掉）；变量级两者都可能输出（`2209-2212`） | **翻转**：SPIR-V `ColMajor` → HLSL `row_major`，`RowMajor` → `column_major` |
| 嵌套结构体起始偏移 | 结构体本身不写 layout；把子成员的 `row_major` **合并**后写到顶层块成员上 | 结构体成员可整体指定 `packoffset`；主序写在**每个成员**上，不合并 |
| 不可表达时 | 抛异常（`Buffer block cannot be expressed as ...`），可退化为 `--flatten-ubo` | 抛异常（`cbuffer ID ... cannot be expressed ...`），SSBO 路径不会失败 |

核心差别一句话：**GLSL 用“命名布局标准”重建内存，HLSL 的 cbuffer 用“HLSL 自己的打包规则 + 起始偏移”，
只有 HLSL 的 ByteAddressBuffer/StructuredBuffer 路径才逐字节照抄 SPIR-V 的偏移/步长。**

---

## 2. 实验方法（可复现、无 GPU）

1. `make -j` 构建本仓库 CLI：`./spirv-cross`。
2. 用 `glslangValidator -V` 把 GLSL 源编译成 SPIR-V；用 `spirv-as`/`spirv-dis` 手工构造并反汇编模块；
   用 `spirv-val [--scalar-block-layout]` 判定模块合法性。
   这些工具只做 CPU 编译/校验，不接触驱动。
3. 用 `spirv-cross --reflect`（`spirv_reflect.cpp:424-450`）直接读出 SPIR-V 中的
   `offset` / `array_stride` / `matrix_stride` / `row_major`，作为“地面真值”。
4. 测 HLSL 真实布局时，把 HLSL 源交回 `glslangValidator -D -V` 生成 SPIR-V，再读其 `OpMemberDecorate`。
   这种做法把“HLSL 语义下的偏移”变成可读数据，仍然只依赖编译器前端。
5. 交叉验证：把 `spirv-cross` 的输出重新编译成 SPIR-V，比对前后 `Offset/ArrayStride/MatrixStride`。

> `--reflect` 的注释也印证了“结构体大小依赖上下文”：见 `spirv_reflect.cpp:358-363` 的 FIXME——
> 同一个结构体在 std140 与 std430 块中大小不同，所以反射只输出装饰值，不输出结构体大小。

---

## 3. 两个后端共同的输入：SPIR-V 布局装饰

这两处 helper 是所有偏移/步长的唯一来源：

- `Compiler::type_struct_member_offset`（`spirv_cross.cpp:2058`）→ `DecorationOffset`
- `Compiler::type_struct_member_array_stride`（`spirv_cross.cpp:2074`）→ 数组成员类型上的 `DecorationArrayStride`
- `Compiler::type_struct_member_matrix_stride`（`spirv_cross.cpp:2091`）→ 成员上的 `DecorationMatrixStride`
- 矩阵“声明大小”由主序决定（`spirv_cross.cpp:2353-2361`）：
  `RowMajor → stride * vecsize`，`ColMajor → stride * columns`。

GLSL 后端的 `--flatten-ubo` 和 HLSL 后端的 ByteAddressBuffer 路径都是**直接消费这些字节数**；
GLSL 的常规路径与 HLSL 的 cbuffer 路径则必须把它们“翻译”成某个打包标准。

---

## 4. GLSL 后端：反推命名布局标准

### 4.1 布局标准的判定顺序

`layout_for_variable` 决定块级限定符（`spirv_glsl.cpp:2408-2417`），实际选择在
`buffer_to_packing_standard`（`spirv_glsl.cpp:2437-2506`）：

1. 允许 std430 的块（SSBO / push constant）：`std430`
2. 否则 `std140`
3. Vulkan 语义下可退到 `scalar`（需要 `GL_EXT_scalar_block_layout`）
4. 以上都失败时，用 `*EnhancedLayout` 版本再试；成功则设置
   `SPIRVCrossDecorationExplicitOffset`，并在成员上输出 `layout(offset = N)`（`spirv_glsl.cpp:2460/2475/2482/2498`）
5. 仍失败 → 抛异常（`spirv_glsl.cpp:2502-2506`）

UBO 的 std430 只在 `options.vulkan_semantics` 下被接受（`spirv_glsl.cpp:2486-2492`），
这解释了下面“std430 UBO 在普通 GLSL 下直接报错”的现象。

### 4.2 打包模型（对齐 / 步长 / 大小）

- 数组对齐：std140/std140-enhanced 时最小 16，其余取元素对齐（`spirv_glsl.cpp:1766-1777`）
- 结构体对齐：成员最大对齐；std140 再向上取 16（`spirv_glsl.cpp:1780-1795`）
- 标量/向量：GL 4.5 §7.6.2.2 Rule 1/2/3（`spirv_glsl.cpp:1810-1825`）；HLSL 打包时向量只按基础大小对齐（`1805-1808`）
- 列主序矩阵 Rule 5（`1829-1839`）：vec4 填充时对齐 16；std430 下 vec3 列也对齐 16
- 行主序矩阵 Rule 7（`1843-1852`）：vec4 填充时对齐 16；std430 下 columns==3 也对齐 16
- 数组步长 = 元素大小按数组对齐向上取整（`1860-1872`）
- 成员大小：`type_to_packed_size`（`1874-2001`）；其中 `1888-1919`（数组尾元素）与
  `1988-1996`（矩阵尾元素）是 HLSL 专用分支（见 §5、§6.1b）
- 结构体后续成员：非 HLSL 时按前一个结构体的对齐做 `pad_alignment`（`1949-1952`、`2085-2090`），
  这是 std140/std430 “结构体后第一个成员按结构体基对齐”规则的实现

### 4.3 校验只检查 Offset 与 ArrayStride

`buffer_is_packing_standard`（`spirv_glsl.cpp:2003-2142`）里：

```cpp
// 2095-2111：固定 offset 的标准要求 actual_offset == offset；灵活 offset 只要求对齐
// 2114-2125：数组步长必须与 type_to_packed_array_stride 完全一致
// 2127-2137：子结构必须递归满足（无法给子结构成员写 layout(offset)）
```

**这里没有任何 `MatrixStride` 的比较。** 其后果见 §7.2。

### 4.4 显式 offset 的边界

`layout_for_member`（`spirv_glsl.cpp:1547-1616`）只在 `SPIRVCrossDecorationExplicitOffset`
被设置且成员有 `Offset` 时输出 `offset = N`（`1606-1610`）。注释明确：
子结构体成员无法写 `layout(offset)`（`2466-2467`、`2127-2128`），所以子结构必须是“本来就合规”的；
enhanced layouts 只救得了顶层排布。

### 4.5 展平路径

`emit_buffer_block_flattened`（`spirv_glsl.cpp:2881-2904`）把 UBO 变成
`uniform vec4 NAME[ceil(struct_size/16)]`；访问由 `flattened_access_chain_offset`
（`11933-12109`）用 `Offset`/`ArrayStride`/`MatrixStride`/`RowMajor` 现算字节偏移；
动态索引要求步长能被 16 整除，否则抛异常（`11968-11975`、`12053-12060`、`12082-12090`）。

---

## 5. HLSL 后端：两条完全不同的表达路径

### 5.1 cbuffer 路径：类型 + packoffset

`emit_buffer_block`（`spirv_hlsl.cpp:2838-2967`）：

- 只有通过 `BufferPackingHLSLCbufferPackOffset` 校验才设置 `SPIRVCrossDecorationExplicitOffset`
  （`2904-2912`），否则抛
  `cbuffer ID ... cannot be expressed with either HLSL packing layout or packoffset.`
- `emit_struct_member`（`2713-2739`）把字节偏移转成 `packoffset`：
  ```cpp
  uint32_t offset = memb[index].offset - base_offset;
  if (offset & 3) SPIRV_CROSS_THROW("Cannot pack on tighter bounds than 4 bytes in HLSL.");  // 2730-2731
  static const char *packing_swizzle[] = { "", ".y", ".z", ".w" };
  packing_offset = join(" : packoffset(c", offset / 16, packing_swizzle[(offset & 15) >> 2], ")");  // 2733-2734
  ```
  → `packoffset` **只能固定起点**，粒度为 4 字节；16 位标量布局无法表达。
- 顶层结构体的成员被**内联**进 cbuffer（`2925-2942` 逐个 `emit_struct_member`），因此它们的布局
  必须符合 HLSL 规则；嵌套结构体则以一个整体成员出现，只能整体指定起始 `packoffset`。
- 数组的 UBO（SM5.1）走 `ConstantBuffer<T>`，**不支持 packoffset**，必须完全符合 HLSL 打包规则
  （`2954-2961`），校验用 `BufferPackingHLSLCbuffer`（非灵活 offset）。
- push constant 在无 root constants 时等价于 cbuffer（`2970-2975`）；有 root constants 时按区间切片再加
  `packoffset`（`2987-3030`）。

### 5.2 SSBO 路径：ByteAddressBuffer / StructuredBuffer

`emit_buffer_block` 的 UAV 分支（`2848-2878`）输出 `RWByteAddressBuffer`（或
`--hlsl-preserve-structured-buffers` 下的 `RWStructuredBuffer<T>`，检测见 `is_user_type_structured`，
`7391-7404`）。读写用 `read_access_chain*` / `write_access_chain*`
（`4874-5070`、`5211-5441`），逐级累加 `type_struct_member_offset`，矩阵按
`MatrixStride` 与主序展开成标量/向量 `Load*` 指令：

```cpp
// 行主序矩阵：取元素逐个寻址（5063-5064）
load_expr += join(base, ".Load", template_expr, "(", chain.dynamic_index,
                  chain.static_index + c * (type.width / 8) + r * chain.matrix_stride, ")");
// 列主序矩阵：按列读取（5032-5035）
chain.static_index + c * chain.matrix_stride
```

因此 SSBO 路径是**逐字节忠实**的：偏移与步长都来自装饰，不受 HLSL 打包规则影响。

### 5.3 行列主序：翻转

`layout_for_member`（`spirv_hlsl.cpp:2696-2711`）：

```cpp
// Flip the convention. HLSL is a bit odd in that the memory layout is column major ...
if (flags.get(DecorationColMajor))      return "row_major ";
else if (flags.get(DecorationRowMajor)) return "column_major ";
```

类型名保持 `float<cols>x<rows>`（`spirv_hlsl.cpp:501-521`），语义通过“矩阵始终以转置状态保存、
乘法顺序对调”实现（`OpMatrixTimesVector` 等，`6055-6074`）。注意这里**不做 GLSL 那样的子成员合并**，
注释 `2700-2701` 明确说明这一点。

---

## 6. 实测案例：可观察的偏移与声明

下表所有数值均为实测（`spirv-dis` 读取装饰；`spirv-cross` 输出；`glslang` 回编验证）。

### 6.1 标量数组

源（`a140.frag`）：`layout(std140) uniform UBO { float s[4]; float tail; }`

| 项 | 值 |
|---|---|
| SPIR-V | `ArrayStride 16`，`s@0`，`tail@64` |
| GLSL 输出 | `layout(binding=0, std140) uniform UBO { float s[4]; float tail; }` |
| HLSL 输出 | `float ubo_s[4] : packoffset(c0);` `float ubo_tail : packoffset(c4);` |

同样的紧凑数组放在 **SSBO**（`layout(std430) buffer`，`ArrayStride 4`、`tail@16`）：

| 后端/模式 | 结果 |
|---|---|
| GLSL（SSBO） | `layout(binding=0, std430) buffer B { float s[4]; float tail; }`（步长 4 隐式） |
| GLSL（把同样内容放进 UBO，普通 GLSL） | **抛异常**：UBO 不允许 std430，且 std140 的步长 16 与原布局不符（`spirv_glsl.cpp:2486-2492`、`2502-2506`） |
| GLSL（UBO + `-V`） | `layout(scalar)` + `GL_EXT_scalar_block_layout` |
| HLSL cbuffer（同一 UBO） | **抛异常** `cbuffer ID 12 ... member index 0 (name: s) cannot be expressed ...` |
| HLSL ByteAddressBuffer（同一 SSBO） | 正常，`Load(0)`/`Load(12)`/`Store(16)`，步长 4 直接来自 `ArrayStride`，无数组声明 |

结论：**std140 下两者对 `float[]` 一致（步长 16）；std430 的紧凑 `float[]` 只有 GLSL 的 SSBO /
Vulkan-scalar 能表达，HLSL cbuffer 一律拒绝，ByteAddressBuffer 则按字节正确处理。**

注意区分“标量”与“标量数组”：std140 中单个 `float` 对齐只有 4，所以 `{float a; float b;}` 在
std140 下仍是 0/4，GLSL 会选中 `std140` 而不报错（实测 `ubo430.spv`）；只有**数组**的步长在
std140 下被抬到 16，才会造成上面的差异。

### 6.1b 向量数组：`packoffset` 会覆盖 HLSL 的尾元素打包

源（`vec3arr140.frag`）：`layout(std140) uniform UBO { vec3 arr[2]; float tail; }`
→ `ArrayStride 16`、`arr@0`、`tail@32`（std140 规则）。

纯 HLSL 写同样的类型时，编译器会把最后一个元素的 12 字节“尾部”让给后续标量，实测
（`tailpack.hlsl`）得到 `arr@0`、`tail@28`。而 `spirv-cross` 输出：

```hlsl
cbuffer UBO : register(b0)
{
    float3 ubo_arr[2] : packoffset(c0);
    float ubo_tail : packoffset(c2);   // 强制 32，覆盖 HLSL 的 28
};
```

回编后 `OpMemberDecorate %UBO 1 Offset 32` —— 与原始一致。这里能直接看到“声明差异”：
同样的 `float3 arr[2]`，有无 `packoffset(c2)` 决定了 `tail` 落在 28 还是 32。

矩阵数组有同样的现象，而且验证了模型的尾元素处理（`spirv_glsl.cpp:1888-1919`、`1988-1996`）：
纯 HLSL 写 `row_major float2x3 a[2]; float t;` 实测得到 `a@0`（步长 32）、`t@60`（最后一个元素的
尾部 4 字节被复用）；`spirv-cross` 对该模块输出
`row_major float2x3 _18_a[2] : packoffset(c0); float _18_t : packoffset(c3.w);`，
即 `c3.w` = 字节 60 —— `packoffset` 精确复现了 HLSL 的尾元素打包。

### 6.2 矩阵数组

**(a) std140 列主序 `mat2x3[2]`**（`matarr140.spv`）：`MatrixStride 16`、`ArrayStride 32`、`tail@64`

```glsl
// spirv-cross GLSL
layout(binding = 0, std140) uniform UBO { mat2x3 cm[2]; float tail; } ubo;
```
```hlsl
// spirv-cross HLSL
cbuffer UBO : register(b0)
{
    row_major float2x3 ubo_cm[2] : packoffset(c0);
    float ubo_tail : packoffset(c4);
};
```
`row_major float2x3` 在 HLSL 中 = 2 行 × 16 = 32 B/元素，与 `ArrayStride 32` 一致（实测
`glslang`：`_arr_mat2v3float... ArrayStride 32`）。若换成 `column_major float2x3`，步长会变成
3 × 16 = 48（实测 `hlslarr.spv` 中 `column_major float2x3 b[2]` 的 `ArrayStride 48`），
所以“翻转主序”不是装饰性写法，而是步长正确性的前提。

**(b) SPIR-V 行主序 `mat2x3[2]`**（`rmarr.spv`）：`RowMajor`、`MatrixStride 16`、`ArrayStride 48`、`tail@96`

```glsl
layout(binding = 0, std140) uniform UBO { layout(row_major) mat2x3 rm[2]; float tail; } ubo;
```
```hlsl
cbuffer UBO : register(b0)
{
    column_major float2x3 ubo_rm[2] : packoffset(c0);
    float ubo_tail : packoffset(c6);
};
```
实测 HLSL 侧 `column_major float2x3` 的数组步长 = 3 列 × 16 = 48，与 SPIR-V 一致。
仓库自带的参照输出正是这一对：`reference/shaders/vert/read-from-row-major-array.vert:5`
是 `layout(row_major) mat2x3 var[3][4]`，`reference/shaders-hlsl/vert/read-from-row-major-array.vert:3`
是 `column_major float2x3 _104_var[3][4] : packoffset(c0);`。

**(c) std430 `mat4 m[2]`**：`MatrixStride 16`、`ArrayStride 64`、`m@0`、`tail@128`；GLSL `std430`，
HLSL ByteAddressBuffer 逐元素寻址。读取 `m[1][2][3]` 生成 `b.Load(108)`，正好是
`0 + 1*64（数组步长）+ 2*16（矩阵步长）+ 3*4（分量）`，三个装饰值全部被直接消费。

**(d) 紧凑矩阵步长 `MatrixStride 8`（std430/scalar）**

| 场景 | GLSL | HLSL cbuffer | HLSL ByteAddressBuffer |
|---|---|---|---|
| 单个列主序 `mat2x3`，`MatrixStride 8`，`tail@24`（UBO） | 普通 GLSL：`std140` + `layout(offset=24)`（步长被改写为 16）；`-V`：`scalar` | **接受**，`row_major float2x3 : packoffset(c0)` + `float : packoffset(c1.z)`（越界语义错误，见 §7.2） | — |
| 单个行主序 `mat2x3`，`MatrixStride 8`，`tail@24`（UBO） | 普通 GLSL：`std140` + `layout(row_major, offset=24)`（步长被改写）；`-V`：`scalar` | **接受**，`column_major float2x3 : packoffset(c0)` + `float : packoffset(c1.z)`（同样越界） | — |
| 行主序 `mat2x3`，`MatrixStride 8`，在 **std430 SSBO** 中 | `std430`（std430 对行主序 mat2x3 的对齐正是 8，步长合法） | — | 按 `MatrixStride 8` 正确寻址 |
| 行主序 `mat2x3` 数组（`ArrayStride 24`，`MatrixStride 8`）在 UBO | `-V`：`scalar`；普通 GLSL：抛异常 | **抛异常**（数组步长校验不通过） | — |

说明“UBO 才有问题”：std430 对列主序 `mat2x3` 要求 16 对齐（vec3 规则），所以 `MatrixStride 8`
在 std430 UBO 里本就不合法；而行主序 `mat2x3` 在 std430 下对齐是 `columns*4 = 8`，因此同样的
`MatrixStride 8` 放进 SSBO 后，GLSL 能给出 `layout(row_major) mat2x3` 并**往返保持** 8
（`scalar4.spv` → 回编后仍为 `MatrixStride 8`）。一旦被迫退回 `std140`（UBO 的默认选择），
步长就会被静默改写。

为什么数组会拒绝而单矩阵不会：数组校验会比较 `type_to_packed_array_stride`
（`spirv_glsl.cpp:2114-2125`），能发现 24 与 HLSL 的 32/48 不符；而单个矩阵既没有数组步长、
也不比较 `MatrixStride`，于是漏过。ByteAddressBuffer 的忠实性可以直接观察
（手工构造的两个模块，唯一差别是 `MatrixStride`）：

```
MatrixStride 8  读取 m[1][2] → _4.Load(16)     // 1*8  + 2*4
MatrixStride 16 读取 m[1][2] → _4.Load(24)     // 1*16 + 2*4
```

### 6.3 嵌套结构体

**(a) std430 SSBO（`nest430.spv`）**：`P{float a@0; float b@4}`、`P parr[2]`（`ArrayStride 8`）、
块成员 `x@0, p@4, parr@12, tail@28`

```glsl
layout(binding = 0, std430) buffer B { float x; P p; P parr[2]; float tail; } b;
```
```hlsl
// ByteAddressBuffer 路径没有结构体声明，只有字节偏移：
b.Store(0, asuint(((asfloat(b.Load(4)) + asfloat(b.Load(8))) + asfloat(b.Load(24))) + asfloat(b.Load(28))));
```
`p.a=4`、`p.b=8`、`parr[1].b=12+8+4=24`、`tail=28` —— 与 SPIR-V 完全一致。

**(b) std140 UBO（`nestubo.spv`）**：`P{vec4 a@0; vec4 b@16}`、`P parr[2]`（`ArrayStride 32`）、
块成员 `x@0, p@16, parr@48, tail@112`

```hlsl
cbuffer UBO : register(b0)
{
    float ubo_x : packoffset(c0);
    P ubo_p : packoffset(c1);
    P ubo_parr[2] : packoffset(c3);
    float ubo_tail : packoffset(c7);
};
```
`16→c1`、`48→c3`、`112→c7`，全部一致。仓库自带测试同样体现该一致性：
`reference/shaders/comp/struct-packing.comp` 与
`reference/shaders-hlsl/comp/struct-packing.comp`（后者把 `Content` 的成员偏移 0/16/32/64/80/96/128/144/152
逐字节还原为 `ssbo_140.Load*` 的常数项）。

**(c) 只有 std140 能表达的嵌套结构（`nested_bad.spv`）**：
`Bad{float a@0; vec3 b@16; float c@28}`，`tail@32`

| 后端 | 结果 |
|---|---|
| GLSL | `layout(binding=0, std140) uniform UBO { Bad bad; float tail; }` ✓ |
| HLSL | **抛异常** `cbuffer ID 14 (name: UBO), member index 0 (name: bad) cannot be expressed ...` |

原因：HLSL 对嵌套结构体只能整体给一个 `packoffset`，而 HLSL 自身的规则会把 `vec3` 放在偏移 4
（不跨 vec4 的策略给出 0/4/16），与 SPIR-V 的 0/16/28 不符；子结构内部无法再用 `packoffset` 修正。

**(d) HLSL 原生、GLSL 无法表达的嵌套结构（`hb_float3.spv`）**：
`float a@0; float3 b@4; float c@16`（HLSL 语义下合法，`spirv-val` 通过）

| 后端 | 结果 |
|---|---|
| GLSL（普通） | **抛异常** |
| GLSL（`-V`） | `layout(scalar)` + `GL_EXT_scalar_block_layout` |
| GLSL（`--flatten-ubo`） | `uniform vec4 CB[2];` + `CB[0].x, CB[0].y, CB[1].x` |
| HLSL | 原样还原为 `float _14_a : packoffset(c0); float3 _14_b : packoffset(c0.y); float _14_c : packoffset(c1);` |

### 6.4 主序：GLSL 合并 vs HLSL 逐成员

同一份由 HLSL 生成的 SPIR-V（`mixshift.spv`），结构体内**混合主序**：

```
OpMemberDecorate %Mixed 0 RowMajor  MatrixStride 16 Offset 0    // cm
OpMemberDecorate %Mixed 1 ColMajor  MatrixStride 16 Offset 48   // rm
OpMemberDecorate %Mixed 2 Offset 80                             // tail(vec4)
OpMemberDecorate %CB    0 Offset 0 ; %CB 1 Offset 96
```

| 后端 | 声明 | 回编后的 SPIR-V 偏移 |
|---|---|---|
| HLSL | `struct Mixed { column_major float2x3 cm; row_major float2x3 rm; float4 tail; };`<br>`Mixed _16_m : packoffset(c0); float4 _16_after : packoffset(c6);` | **完全保持**：0 / 48 / 80 / `after@96` |
| GLSL | `struct Mixed { mat2x3 cm; mat2x3 rm; float4 tail; };`<br>`layout(row_major) Mixed m;` | **漂移**：0 / 48 / **96** / `after@112` |

原因是 GLSL 把子成员的 `RowMajor` 递归合并到顶层结构体成员上
（`combined_decoration_for_member`，`spirv_cross.cpp:5219-5246`；使用点 `spirv_glsl.cpp:1563-1583`，
注释 `1563-1579` 解释“GLSL 的裸 struct 不支持 layout，只能提到顶层”）。合并后**所有**矩阵都变成
`row_major`，于是 `mat2x3` 的步长从 32（列主序）变成 48（行主序），后续成员整体后移 16 字节。
HLSL 因为“每个成员单独写 `column_major`/`row_major`”（`spirv_hlsl.cpp:2700-2711`）而不受影响。

其他可引用的一致性例子：

- SPIR-V 列主序 `mat4` → HLSL `row_major float4x4`
  （`reference/shaders-hlsl/vert/basic.vert:3`）。
- SPIR-V `ColMajor` 的结构体成员在 HLSL 侧变成 `row_major`
  （`reference/shaders-hlsl/frag/row-major-layout-in-struct.frag:9-10`；
  源 `shaders-hlsl/frag/row-major-layout-in-struct.frag` 声明的是 `layout(column_major) Foo foo;`，
  SPIR-V 中两个矩阵成员都是 `ColMajor`）。
- GLSL 的成员级代码不输出 `column_major`（`spirv_glsl.cpp:1584-1586` 的分支被注释掉），因为列主序
  是 GLSL 的默认值；但**变量级**装饰会照原样输出（`2209-2212`）。手工给块变量加 `ColMajor` 装饰，
  实测 GLSL 得到 `layout(column_major, binding = 0, std140) uniform UBO`，而 HLSL 仍按自己的翻转规则
  输出 `row_major float4x4`（`blockdec.spv`）。glslang 通常把 GLSL 源的块级 `row_major`
  下沉成成员装饰（见 `blk_rowmajor.spv`），所以这条路径主要出现在手工/第三方 SPIR-V 中。

---

## 7. 结构性差异与失效模式

### 7.1 偏移与步长的“表达力”边界

| 布局特征 | GLSL std140 | GLSL std430/scalar | HLSL cbuffer | HLSL ByteAddress |
|---|---|---|---|---|
| 标量数组步长 16 | ✔ 隐式 | ✔ | ✔ | ✔ 显式 |
| 标量数组步长 4 | ✘ | ✔（std430 / scalar） | **✘ 抛异常** | ✔ |
| 矩阵步长 = 标准值 | ✔ | ✔ | ✔（规则一致时） | ✔ |
| 矩阵步长 ≠ 标准值（单个成员） | **✘ 静默改写**（§7.2） | ✔（scalar/std430 合法时） | **✘ 静默接受但语义错误**（§7.2） | ✔ |
| 嵌套结构体任意紧凑 | ✘（子结构必须合规） | ✔（scalar） | 部分（依赖 HLSL 规则） | ✔ |
| `packoffset` 的粒度（4 字节） | n/a（GLSL 无此语法，用 `offset`） | n/a | **✘ 非 4 对齐的偏移直接抛异常**（`spirv_hlsl.cpp:2730-2731`） | n/a（按字节访问，16 位需 SM6.2，`4924`） |

### 7.2 校验缺口：两侧都不检查 `MatrixStride`

GLSL 的 `buffer_is_packing_standard`（`spirv_glsl.cpp:2003-2142`）只比较 `Offset`
（`2095-2111`）与 `ArrayStride`（`2114-2125`），**完全不读 `DecorationMatrixStride`**。
HLSL cbuffer 路径同样如此：它调用同一个函数，只是把打包标准换成
`BufferPackingHLSLCbuffer(±PackOffset)`（`spirv_hlsl.cpp:2905`、`2956`、`2987`）。

可观察后果（`u_cm2x3_s8_t24.spv`：`ColMajor, MatrixStride 8, Offset 0`，`tail@24`）：

```
GLSL 输出:   layout(binding=0, std140) uniform ... { layout(offset = 0) mat2x3 _m0;
                                                      layout(offset = 24) float _m1; }
回编后 SPIR-V: MatrixStride 16   （原为 8 —— 静默改变，偏移数字却完全相同）
HLSL 输出:   row_major float2x3 _4_m0 : packoffset(c0);
             float _4_m1 : packoffset(c1.z);          // tail 落在矩阵 32B 占用的内部
```

HLSL 侧同样固定为 16：把 `row_major float2x3`（或 `column_major`）写进 cbuffer 交给
`glslang` 前端，得到的总是 `MatrixStride 16`（`hlslmat2.spv`、`live_cm.spv`），
可见“紧凑步长”在 cbuffer 里根本不存在可表达的语法。

即：普通 GLSL 与 HLSL cbuffer 都无法表达紧凑矩阵步长，却都不报错，而是**产出与原布局不等价的代码**。
同一模块在 `-V`（Vulkan）下被判定为 `scalar`，或在 `RWByteAddressBuffer` 路径下按 `MatrixStride 8`
正确寻址，因此这是“普通 GLSL / cbuffer”特有的缺口。

### 7.3 HLSL 结构体大小未按“对齐到 16”取整

HLSL 打包模型（`spirv_glsl.cpp:1874-2001`）在 `packing_is_hlsl` 时只做“不跨 vec4”检查
（`1939-1946`），结构体大小被算成**成员大小之和**，没有 HLSL 实际要求的“结构体占用的字节数向上
取整到 16”。实测 HLSL 语义（`glslang` 回编）：

- `struct P { float a; float b; }` → 下一个成员位于 16（模型给出：8）
- `struct Inner { float a; float2 b; }` → 下一个成员位于 16（模型给出：12）
- `struct Outer { Inner i; float f; }` → `f@16`、结构体大小 32（模型给出：`f@12`、大小 16）

两个可观察后果：

1. **严格的 `ConstantBuffer<T>` 会误拒**：`cbprobe.spv`
   `struct P { float a; float b; }; cbuffer { float x; P p; float t1; }` 且
   `x@0, p@16, t1@32`（HLSL 真实布局）。`--shader-model 51` 输出：
   ```
   HLSL ConstantBuffer<T> ID 4 (name: UBO), member index 2 (name: t1)
   cannot be expressed with normal HLSL packing rules.
   ```
   逐个偏移试探（`p@16` 固定，只改 `t1`）得到：

   | `t1` 偏移 | 8 | 12 | 16 | 24 | 32 |
   |---|---|---|---|---|---|
   | 模型判定 | 拒绝 | 拒绝 | 拒绝 | **接受** | 拒绝（真实布局） |

   即模型认为 `P` 只有 8 字节，`t1` 应在 24；HLSL 实际把 `P` 占用的 16 字节计入，`t1` 在 32。
   同一现象在 HLSL 原生输入 `oracle3u.spv`（`float x; P p; float t1;`）上重现。

2. **packoffset 路径会接受并产出重叠声明**：`outer4.spv`（scalar 合法）
   `Inner{float a@0; vec2 b@4}`、`Outer{Inner@0; float f@12}`、`UBO{Outer@0; float tail@16}`。
   HLSL 输出：
   ```hlsl
   struct Inner { float a; float2 b; };
   struct Outer { Inner _in; float f; };
   cbuffer UBO : register(b0) { Outer _6_o : packoffset(c0); float _6_tail : packoffset(c1); };
   ```
   但真实 HLSL 语义下 `Outer.f` 位于 16、`Outer` 大小为 32，因此 `_6_tail`（字节 16）与 `_6_o.f` 重叠。
   回编验证：`Outer 1 Offset 16` 与 `UBO 1 Offset 16` 重叠，`spirv-val` 报
   `member 1 at offset 16 overlaps previous member ending at offset 31`。
   同一模块的 GLSL `-V`（scalar）输出则**精确保持** 0/4/12/16。

补充：`BufferPackingHLSLCbufferPackOffset` 属于“灵活 offset”变体
（`packing_has_flexible_offset` 对其返回 true，`spirv_glsl.cpp:161-174`），只需偏移满足**模型算出
的对齐**（`2101-2111`），不要求 `actual_offset == offset`；因此即使算错的尺寸没有立刻导致拒绝，
错误仍会被“静默接受”。真正精确的路径是 `ConstantBuffer<T>` 使用的非灵活 `BufferPackingHLSLCbuffer`
（`spirv_hlsl.cpp:2954-2961`）——它把上面的尺寸误差放大成误拒。

### 7.4 push constant

| 输入 | GLSL（普通） | GLSL（`-V`） | HLSL |
|---|---|---|---|
| `layout(push_constant) { float a[4]; float tail; }`（`ArrayStride 4`） | `uniform PC pc;`（无 layout，见 `spirv_glsl.cpp:2526-2552`） | `layout(push_constant, std430)` | **抛异常**（`float a[4]` 步长 4 不满足 cbuffer） |
| `layout(push_constant) { vec4 v; float tail; }`（std140） | `uniform PC pc;` | `layout(push_constant, std430)` | `cbuffer PC { float4 pc_v : packoffset(c0); float pc_tail : packoffset(c1); };`（无 `register`） |

GLSL 非 Vulkan 把 push constant 当普通 uniform 结构体输出（`spirv_glsl.cpp:2528-2550`），
此时没有任何布局限定符，偏移由驱动/编译器默认规则决定；HLSL 则始终用 cbuffer 规则。

---

## 8. 汇总对照表（同一 SPIR-V，两种表达）

以下 `ubo.spv` 同时包含标量数组、矩阵、行/列主序与嵌套结构体：

```
SPIR-V 装饰（--reflect / spirv-dis）
Inner   : v2@0, f@8, v3@16
UBO     : scalars@0 (ArrayStride 16), mats@64 (Stride 64),
          m23@192 (ColMajor, MatrixStride 16), rm23@224 (RowMajor, MatrixStride 16),
          inner@272, inners@304 (ArrayStride 32), tail@400
```

```glsl
// GLSL（std140，全部偏移/步长隐式，结构体无 layout）
layout(binding = 0, std140) uniform UBO
{
    float scalars[4];
    mat4 mats[2];
    mat2x3 m23;
    layout(row_major) mat2x3 rm23;
    Inner inner;
    Inner inners[3];
    vec2 tail;
} ubo;
```

```hlsl
// HLSL（cbuffer，偏移固定为 packoffset，主序逐成员翻转，嵌套结构整体一个 packoffset）
cbuffer UBO : register(b0)
{
    float ubo_scalars[4] : packoffset(c0);
    row_major float4x4 ubo_mats[2] : packoffset(c4);
    row_major float2x3 ubo_m23 : packoffset(c12);
    column_major float2x3 ubo_rm23 : packoffset(c14);
    Inner ubo_inner : packoffset(c17);
    Inner ubo_inners[3] : packoffset(c19);
    float2 ubo_tail : packoffset(c25);
};
```

`packoffset` 的 cN 与字节偏移严格是 `N = offset/16`（余数用 `.y/.z/.w`）；
矩阵与数组的**步长**在两边都不写出来，而是由所选标准/HLSL 规则隐式给出。

---

## 9. 复现命令

```bash
make -j                                   # 构建 ./spirv-cross
glslangValidator -V x.frag -o x.spv       # GLSL → SPIR-V
spirv-as hand.spvasm -o hand.spv --target-env vulkan1.1
spirv-val --target-env vulkan1.1 [--scalar-block-layout] hand.spv
spirv-dis x.spv | grep -E 'OpMemberDecorate|ArrayStride'   # 地面真值
./spirv-cross x.spv --reflect             # 驱动无关的反射（offset/stride/row_major）
./spirv-cross x.spv                       # GLSL
./spirv-cross x.spv -V                    # Vulkan GLSL（scalar 回退）
./spirv-cross x.spv --flatten-ubo         # 显式字节偏移
./spirv-cross x.spv --hlsl --shader-model 50          # HLSL（cbuffer + ByteAddressBuffer）
./spirv-cross x.spv --hlsl --shader-model 51          # 数组 UBO → ConstantBuffer<T>
glslangValidator -D -V -S frag -e main out.hlsl -o out.spv   # 回编 HLSL，测真实偏移
```

关键验证点：把 `spirv-cross` 的输出再编译回 SPIR-V，对比 `OpMemberDecorate ... Offset`
与 `OpDecorate ... ArrayStride/MatrixStride` 是否与原始一致（§6.4、§7.2、§7.3 中的“漂移”即由此得到）。

---

## 10. 结论

1. **偏移**：GLSL 通过选择 `std140/std430/scalar` 让偏移**隐式**成立，必要时才对顶层成员写
   `layout(offset=N)`（`spirv_glsl.cpp:1606-1610`、`2460-2498`）。HLSL cbuffer 用
   `packoffset(cN.zw)` **显式固定每个成员的起始字节**（`spirv_hlsl.cpp:2726-2734`）；HLSL 的
   ByteAddressBuffer 则完全不声明类型，直接用装饰中的字节偏移寻址（`spirv_hlsl.cpp:4874-5070`）。
2. **步长**：GLSL 的数组步长由标准规则推导（`1860-1872`）；HLSL cbuffer 的步长由 HLSL 规则决定
   （与 GLSL 的标准规则在常见情形下恰好一致，见 §6.2）；只有 ByteAddressBuffer 路径直接使用
   `ArrayStride`/`MatrixStride`。两侧都**不校验** `MatrixStride`，导致紧凑矩阵步长在普通 GLSL 与
   cbuffer 中被静默改写（§7.2）。
3. **行列主序**：GLSL 的成员级只写 `row_major`（列主序为默认），并把子结构的主序**合并提升**
   到顶层成员，因此混合主序结构体会被“统一”从而改变偏移（§6.4）；HLSL **翻转**主序并逐成员
   书写（`spirv_hlsl.cpp:2696-2711`），能精确保留混合主序。
4. **嵌套成员的起始偏移与填充**：std140/std430 的“结构体对齐 16 / 后续成员按结构体对齐”规则
   （`1791-1793`、`1949-1952`、`2085-2090`）与 HLSL 的“结构体大小按 16 取整、向量不跨 vec4”
   规则（`1939-1946`）在多数布局下等价；但在“小结构体紧跟成员”的紧凑布局上，spirv-cross 的 HLSL
   模型漏掉了结构体大小的对齐取整，既会误拒合法的 `ConstantBuffer<T>` 布局，也会接受并生成
   不可实现的 `packoffset` 声明（§7.3）。
