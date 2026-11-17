# 着色缓冲中矩阵与数组的跨后端布局：SPIRV-Cross 的 GLSL 与 HLSL 表达

> 分析对象：本仓库（SPIRV-Cross）源码。所有结论均来自源码与离线编译产物（`spirv-cross` 反编译 + `spirv-dis` 反汇编 + `--reflect` JSON），**不依赖任何 GPU 驱动**。文中“字节偏移/步长”都是 SPIR-V 装饰（`Offset` / `ArrayStride` / `MatrixStride`）所声明的逻辑布局，以及后端把它们翻译出来的可观察文本。

## 0. 共同的事实来源：SPIR-V 装饰，而不是源语言规则

两个后端都不重新“计算”布局，它们读取的是 SPIR-V 模块上已经固化的装饰。访问器全部定义在 `spirv_cross.cpp`：

- 成员起始偏移：`Compiler::type_struct_member_offset()` 直接返回成员元数据中的 `DecorationOffset`，没有装饰则抛异常（`spirv_cross.cpp:2058-2072`）。
- 数组步长：`type_struct_member_array_stride()` 读取数组**类型自身**上的 `DecorationArrayStride`（`spirv_cross.cpp:2074-2089`，注释明确“ArrayStride is part of the array type not OpMemberDecorate”）。
- 矩阵步长：`type_struct_member_matrix_stride()` 读取 `OpMemberDecorate` 上的 `DecorationMatrixStride`（`spirv_cross.cpp:2091-2106`，注释“MatrixStride is part of OpMemberDecorate”）。
- 矩阵占用大小按行/列主序分别用 `matrix_stride * vecsize` 或 `matrix_stride * columns` 计算（`spirv_cross.cpp:2353-2361`）。

因此，两个后端的差异只是“**用什么语言构造去表达同一组装饰**”。

### 0.1 布局识别引擎：`BufferPackingStandard`

枚举见 `spirv_cross.hpp:170-180`：`Std140 / Std430 / Std140EnhancedLayout / Std430EnhancedLayout / HLSLCbuffer / HLSLCbufferPackOffset / Scalar / ScalarEnhancedLayout`。

核心校验函数 `CompilerGLSL::buffer_is_packing_standard()`（`spirv_glsl.cpp:2003-2145`）逐成员模拟布局，并用 SPIR-V 的实际 `Offset`/`ArrayStride` 对照：

- 每成员对齐值由 `type_to_packed_alignment()` 给出，大小由 `type_to_packed_size()` 给出，数组步长由 `type_to_packed_array_stride()` 给出（`spirv_glsl.cpp:1746-1872`）；
- 模拟偏移与 SPIR-V 实际偏移不一致即判定该布局标准不成立（`spirv_glsl.cpp:2097-2105`）；
- 数组成员还要校验 `type_to_packed_array_stride()` 与实际 `ArrayStride` 相等（`spirv_glsl.cpp:2115-2125`）；
- 嵌套结构递归校验，子结构使用更严格的“子结构打包”变体（`spirv_glsl.cpp:2129-2137` 与 `packing_to_substruct_packing()`，`spirv_glsl.cpp:189-204`，其中 `HLSLCbufferPackOffset → HLSLCbuffer`，增强布局 → 普通布局）。

两个关键分类谓词：

- `packing_is_vec4_padded()`：std140 系与 **HLSL 两个 cbuffer 变体都返回 true**（`spirv_glsl.cpp:133-145`）。即代码把 HLSL cbuffer 与 std140 视为同一类“vec4 填充”模型；
- `packing_is_hlsl()`：仅两个 HLSL 变体为真（`spirv_glsl.cpp:148-159`）。HLSL 额外有“向量/结构不得跨越 16 字节 vec4 边界，否则对齐提升到 16”的规则，在 `spirv_glsl.cpp:2057-2076` 与 `1939-1946` 两处用 `begin_word != end_word` 判定。

---

## 1. GLSL 后端：用块级布局限定符整体表达，成员不写偏移

### 1.1 块声明：推断 std140 / std430 / scalar

`CompilerGLSL::layout_for_variable()` 对 uniform 块调用 `buffer_to_packing_standard()`（`spirv_glsl.cpp:2410-2417`），后者按优先级尝试匹配（`spirv_glsl.cpp:2437-2507`）：

1. 先试 `std430`；
2. 再试 `std140`；
3. Vulkan 语义下试 `scalar`，命中则要求 `GL_EXT_scalar_block_layout` 扩展；
4. 再退回“增强布局”变体（std430/std140/scalar + 成员级 `layout(offset=...)`），命中时调用 `set_extended_decoration(..., SPIRVCrossDecorationExplicitOffset)`，把块标记为需要显式偏移；
5. 全部不匹配则抛异常，建议 flatten 块。

也就是说，**常规 GLSL 输出里没有逐成员偏移/步长**——步长与填充完全由块上的 `std140`/`std430`/`scalar` 关键字隐式承担。实测（本报告自建用例，离线编译）：

```glsl
layout(set = 0, binding = 1, std430) buffer SSBO430 { float s[4]; mat4 cm; ... } b430;
layout(set = 0, binding = 0, std140) uniform UBO140 { float s[4]; mat4 cm; ... } u140;
layout(set = 0, binding = 0, scalar) buffer S { vec3 a; float b; mat3 m; float arr[3]; } s;
```

### 1.2 成员上的显式偏移与行主序

- 只有被标记 `SPIRVCrossDecorationExplicitOffset` 的块，成员才输出 `layout(offset = N)`（`spirv_glsl.cpp:1606-1612`）。仓库自带用例 `shaders/desktop-only/comp/enhanced-layouts.comp` 的 GLSL 参考输出（`reference/shaders/desktop-only/comp/enhanced-layouts.comp:11-33`）即可观察到 `layout(offset = 4) int a;` 等成员级偏移。
- 行/列主序：`layout_for_member()` 对 `DecorationRowMajor` 输出 `row_major`，而列主序是 GLSL 默认值，**刻意不输出任何文本**（`spirv_glsl.cpp:1582-1586`，注释“We don't emit any global layouts, so column_major is default.”）。
- GLSL 不允许在裸 `struct` 声明内写 `layout(row_major)`，所以嵌套结构上的行主序要“上提”到块级成员：`combined_decoration_for_member()` 递归合并子结构所有成员的装饰（`spirv_cross.cpp:5219-5246`），输出形如 `layout(row_major) Inner rowchild;`。源码注释完整解释了这一限制与上提策略（`spirv_glsl.cpp:1566-1579`）。
- 旧平台/OpenGL 语义下行主序矩阵不是原生支持时（仅 MSL 显式把 `backend.native_row_major_matrix` 置 false，见 `spirv_msl.cpp:1700`；GLSL/HLSL 默认 true，`spirv_glsl.hpp:662`），访问点用 `transpose()` 改写（`convert_row_major_matrix()`，`spirv_glsl.cpp:16955-17011`）。另外默认开启的 `enable_row_major_load_workaround`（`spirv_glsl.hpp:153-157`）会在整块含行主序矩阵时，把整矩阵加载包进 `spvWorkaroundRowMajor(...)` 包装函数（`spirv_glsl.cpp:20827-20881`），参考输出见 `reference/shaders/vert/read-from-row-major-array.vert:11-12,43`。

### 1.3 标量数组：步长差异由布局关键字隐含

同一份 SPIR-V（glslang 对 std140 UBO 与 std430 SSBO 生成不同装饰），`--reflect` 给出的可观察步长：

| 成员 | std140 UBO | std430 SSBO | scalar 块 |
|---|---|---|---|
| `float s[4]` 的 `ArrayStride` | **16** | **4** | **4** |
| `vec3` 后随标量的偏移 | 标量被推到下一 16B 边界 | 紧密排列（标量紧跟 vec3 之后） | 同 std430，且 vec3 自身对齐仅 4 |

scalar 实测反射：`vec3 a @0`、`float b @12`、`mat3 m @16, matrix_stride=12`、`float arr[3] @52, array_stride=4`。对齐规则的代码依据：scalar 布局下任何类型的对齐都退回到基础标量大小（`spirv_glsl.cpp:1801-1803`），矩阵大小为 `vecsize * columns * base`（`spirv_glsl.cpp:1963-1965`）；std140 下数组与结构对齐被提升到 16（`spirv_glsl.cpp:1769-1770, 1792-1793`）。

GLSL 文本本身不出现这些数字——这是与 HLSL 后端最根本的表达差异。

---

## 2. HLSL 后端：三条不同的表达通道

HLSL 后端（`spirv_hlsl.cpp`）按缓冲种类选择完全不同的输出策略。

### 2.1 通道 A：UBO → `cbuffer` + 每成员 `packoffset(cN[.swizzle])`

入口 `CompilerHLSL::emit_buffer_block()`（`spirv_hlsl.cpp:2838-2968`）。对非数组块：

1. 先调用 `buffer_is_packing_standard(type, BufferPackingHLSLCbufferPackOffset, ...)` 验证 SPIR-V 布局能否用 HLSL 规则表达，失败直接抛异常（`spirv_hlsl.cpp:2904-2912`）；
2. 通过后打上 `SPIRVCrossDecorationExplicitOffset` 标记；
3. 逐成员输出 `packoffset`。字节偏移到 HLSL 寄存器/分量的换算在 `emit_struct_member()`（`spirv_hlsl.cpp:2726-2735`）：

```cpp
uint32_t offset = memb[index].offset - base_offset;
if (offset & 3)
    SPIRV_CROSS_THROW("Cannot pack on tighter bounds than 4 bytes in HLSL.");
static const char *packing_swizzle[] = { "", ".y", ".z", ".w" };
packing_offset = join(" : packoffset(c", offset / 16,
                      packing_swizzle[(offset & 15) >> 2], ")");
```

即：**SPIR-V 字节偏移 ÷16 得寄存器号 `cN`，余数 ÷4 得 `.x/.y/.z/.w` 分量**；小于 4 字节的粒度无法表达。嵌套结构成员同样经此路径，因为顶层块在 `emit_buffer_block` 中被“展平”为带前缀的成员（`spirv_hlsl.cpp:2884-2886` 注释说明与 GLSL 一样不能在子结构上用 offset）。

实测 std140 混合类型块（SPIR-V 偏移：0, 8, 16, 32, 44, 48, 80）的输出：

```hlsl
cbuffer U : register(b0)
{
    float  u_a : packoffset(c0);      // byte 0
    float2 u_b : packoffset(c0.z);    // byte 8  -> c0 的第 3 个分量
    float  u_c : packoffset(c1);      // byte 16
    float3 u_d : packoffset(c2);      // byte 32
    float  u_e : packoffset(c2.w);    // byte 44 -> 与 vec3 共用同一 vec4
    row_major float2x2 u_m2 : packoffset(c3);  // byte 48
    float  u_f : packoffset(c5);      // byte 80
};
```

仓库参考输出中同样可见分量级 packoffset，如 `reference/shaders-hlsl/comp/rwbuffer-matrix.comp:5-8`（`packoffset(c0)` 与 `packoffset(c0.y)`）。

**表达失败是可观察的行为差异**：把含 std140 标量数组（`ArrayStride=16`）的嵌套结构 UBO 交给 HLSL 后端时，`buffer_is_packing_standard` 校验失败并抛出
`cbuffer ID ... cannot be expressed with either HLSL packing layout or packoffset.`
（本报告实测；抛出处 `spirv_hlsl.cpp:2909-2911`）。原因是 HLSL cbuffer 内标量数组按 4 字节紧密排列，无法表达 std140 的 16 字节步长。

### 2.2 通道 B：块数组 → `ConstantBuffer<T>`（SM 5.1+），无 packoffset

当块变量自身是数组时，`packoffset` 不可用，改为校验更严格的 `BufferPackingHLSLCbuffer`（无 flexible offset，`spirv_glsl.cpp:161-174`），通过后输出 `ConstantBuffer<T> name[N] : register(...)`（`spirv_hlsl.cpp:2945-2966`，含 SM 版本检查与失败异常）。参考实例：`reference/shaders-hlsl-no-opt/frag/constant-buffer-array.invalid.sm51.frag:9` 的 `ConstantBuffer<CBO_1> cbo[2][4] : register(b4, space0);`。此路径下布局只能依赖 HLSL 自身的隐式 cbuffer 打包规则，SPIR-V 偏移不再逐成员写出。

### 2.3 通道 C：SSBO / UAV → `RWByteAddressBuffer`，偏移变成代码里的整数运算

存储缓冲（`StorageClassStorageBuffer` 或带 `BufferBlock`）默认输出 `RWByteAddressBuffer`/`ByteAddressBuffer`（只读时），仅在 `preserve_structured_buffers` 且带 GOOGLE user-type 装饰时才输出 `StructuredBuffer<T>`（`spirv_hlsl.cpp:2848-2878` 与 `7391-7404`）。

ByteAddressBuffer 路径下，**SPIR-V 的 Offset/ArrayStride/MatrixStride 被原样翻译成 Load/Store 的字节地址表达式**，没有任何布局限定符：

- 结构成员：`subchain.static_index = chain.static_index + type_struct_member_offset(type, i)`（读 `spirv_hlsl.cpp:4882`，写 `spirv_hlsl.cpp:5223-5224`）；
- 数组：动态下标乘 `ArrayStride`，读路径 `ident * chain.array_stride + chain.dynamic_index`（`spirv_hlsl.cpp:4866`，写路径 `spirv_hlsl.cpp:5197`），嵌套数组每层从类型装饰重新取步长（`spirv_hlsl.cpp:4869, 5205`）；
- 矩阵：进入结构时记录 `matrix_stride = type_struct_member_matrix_stride(...)` 与 `row_major_matrix = has_member_decoration(..., DecorationRowMajor)`（`spirv_hlsl.cpp:4891-4898, 5232-5239`）。
  - 列主序矩阵按列做 `Load4(static + c * matrix_stride)`（`spirv_hlsl.cpp:4994-5039`）；
  - 行主序矩阵逐标量取数，地址为 `static + c*(width/8) + r*matrix_stride`——注意两个方向分别用**标量宽度**与**矩阵步长**（`spirv_hlsl.cpp:5059-5064`，写路径对称实现于 `5435` 附近）；
  - 从行主序矩阵取单列时按 `r * matrix_stride` 跨步加载（`spirv_hlsl.cpp:4983-4989`）。

仓库自带的 `reference/shaders-hlsl/comp/rwbuffer-matrix.comp` 是完整证据库：同一矩阵在列主序/行主序两种存储下生成不同地址公式，例如行主序 mat4 的动态标量访问 `_68_index0 * 4 + _68_index1 * 16 + 64`（行步长 4 = 一个 float，列步长 16 = MatrixStride），而列主序是 `_68_index0 * 16 + _68_index1 * 4 + 0`（该文件 72、91 行）。矩阵起始偏移（64、128…）正是 SPIR-V 的成员 `Offset`。

因为 SSBO 走的是裸字节地址，std140 与 std430 的步长差异在 HLSL 端不再导致编译失败——2.1 节中被 cbuffer 拒绝的同型布局改成 SSBO 后可正常输出，地址表达式直接携带 4 或 16 的步长（本报告实测 std430 嵌套结构输出 `u.Load(44)`、`u.Load(12)` 等紧密偏移）。

### 2.4 HLSL 的行/列主序：语义翻转

`CompilerHLSL::layout_for_member()`（`spirv_hlsl.cpp:2696-2711`）把 SPIR-V 装饰**反相**输出：

```cpp
// HLSL is a bit odd in that the memory layout is column major ... but the
// language API is "row-major". ... multiply everything in inverse order,
// and reverse the memory layout.
if (flags.get(DecorationColMajor)) return "row_major ";
else if (flags.get(DecorationRowMajor)) return "column_major ";
```

且与 GLSL 不同，HLSL 允许在任意 struct 内写该限定符，不做子结构装饰合并（同函数注释，`spirv_hlsl.cpp:2700-2701`）。可观察对照：

- GLSL 源 `layout(column_major) Foo foo;` → HLSL 结构体内 `row_major float4x4 v;`（`shaders-hlsl/frag/row-major-layout-in-struct.frag` 对 `reference/shaders-hlsl/frag/row-major-layout-in-struct.frag:7-11`）；
- GLSL 源 `layout(std140, row_major) uniform Block { mat2x3 var[3][4]; }` → HLSL `column_major float2x3 _104_var[3][4] : packoffset(c0);`（`reference/shaders-hlsl/vert/read-from-row-major-array.vert:3`）；
- 普通列主序 `mat4 uMVP;` → `row_major float4x4 ... : packoffset(c0);`，且乘法顺序被反转为 `mul(aVertex, _16_uMVP)`（`reference/shaders-hlsl/vert/basic.vert:3,26`）。

内存布局本身不变（仍由 SPIR-V 装饰保证），翻转的只是 HLSL 语言侧的命名与乘法次序。

---

## 3. 三类专题的并排对照

### 3.1 标量数组

| 维度 | GLSL 后端 | HLSL 后端 |
|---|---|---|
| 步长表达 | 不出现数字；`std140`→16B、`std430`/`scalar`→4B 由块关键字隐含 | cbuffer：无步长语法，靠 HLSL 隐式打包，故 std140 的 16B 标量数组**无法表达并抛异常**；ByteAddressBuffer：步长显式出现在 `idx * ArrayStride` 运算中（`spirv_hlsl.cpp:4866`） |
| 起始偏移 | 仅增强布局时输出 `layout(offset=N)`（`spirv_glsl.cpp:1608-1610`） | `packoffset(cN[.swizzle])`，换算公式 `spirv_hlsl.cpp:2734`；SSBO 中为 Load/Store 的整数字节参数 |
| 填充 | 隐式（std140 规则 4，数组/结构对齐 16：`spirv_glsl.cpp:1769-1770`） | cbuffer 靠“不可跨 vec4”规则模拟（`spirv_glsl.cpp:2057-2076`）；SSBO 不填充，只按装饰寻址 |
| 最小粒度 | 字节（显式 offset） | 4 字节，`offset & 3` 非零即抛异常（`spirv_hlsl.cpp:2730-2731`） |

### 3.2 矩阵与矩阵数组

- **矩阵步长**：GLSL 端完全隐含在 `std*`/`scalar` 与行/列主序限定符中；HLSL cbuffer 端隐含在 `row_major`/`column_major` 限定符与 packoffset 中；HLSL ByteAddressBuffer 端 `MatrixStride` 直接成为地址乘数（列主序 `c * matrix_stride`，行主序 `r * matrix_stride`，`spirv_hlsl.cpp:5034-5035, 5063-5064`）。
- **矩阵数组步长**：SPIR-V 类型装饰 `ArrayStride`（实测 std140 下 `mat4[2]` 为 64、`mat3[3]` 为 48）。GLSL 只写 `mat4 m4[2];`；HLSL cbuffer 写 `row_major float4x4 u_m4[2] : packoffset(c0);` 与 `column_major float3x3 u_rm3[3] : packoffset(c8);`——其中 `c8` 即字节 128，与 SPIR-V `Offset 128` 严格对应（本报告实测）。`type_to_packed_size()` 还为 HLSL 实现了“矩阵/向量数组最后一个元素不占满 vec4、允许后续向量打包进剩余分量”的特殊规则（`spirv_glsl.cpp:1892-1917, 1988-1996`）。
- **行主序的访问改写**：GLSL 在不支持原生行主序的目标上用 `transpose()` 并交换索引次序（`spirv_glsl.cpp:16955-16994`，访问链中交换 `matrix_stride` 与 `width/8` 的角色，`spirv_glsl.cpp:12049-12082`）；HLSL 则在 ByteAddressBuffer 里用双下标 `c*(width/8) + r*matrix_stride` 逐标量拼装（`spirv_hlsl.cpp:5059-5070`），在 cbuffer 里靠翻转限定符 + `mul` 反序。

### 3.3 嵌套结构

- **起始偏移**：SPIR-V 每层结构都带自己的 `Offset`，`get_declared_struct_size()` 按最高偏移成员推算结构大小（`spirv_cross.cpp:2108-2129`）。
- GLSL：嵌套结构原样声明，内部成员不带偏移；行主序等无法在裸结构内表达的限定符上提到块级（`spirv_glsl.cpp:1566-1579` + `spirv_cross.cpp:5219-5246`）。结构后的成员对齐遵循“按前一结构基对齐补齐”的 GL 规则（`spirv_glsl.cpp:2085-2090, 1949-1952`）。
- HLSL cbuffer：顶层块展平，嵌套成员加前缀后逐个打 packoffset，因此嵌套起始偏移也是 `cN` 文本（实测 `Outer u140_o : packoffset(c37);` = 字节 592，与反射的 `o @592` 一致）；子结构递归按 `HLSLCbuffer`（无 packoffset 变体）校验（`spirv_glsl.cpp:2129-2137`）。
- HLSL ByteAddressBuffer：`read_access_chain_struct()`/`write_access_chain_struct()` 每层累加 `type_struct_member_offset`，并重置/重新继承矩阵与数组步长（`spirv_hlsl.cpp:4874-4902, 5211-5244`）。实测嵌套行主序矩阵数组访问生成 `u140_rowchild.rm_mats[1][... ]` 对应的裸地址 `Store(400, ...)`、`Store(416, ...)`（步长 16），起始 400 = 结构偏移 288 + 内部成员偏移（参考本报告生成的 `out.hlsl:58-66`）。
- **结构尾部填充**：HLSL 校验中若成员会跨越 vec4 边界，对齐强制提升到 16（`spirv_glsl.cpp:1939-1946`）；GLSL std140 同样提升但 std430/scalar 不提升（`packing_is_vec4_padded` 分支，`spirv_glsl.cpp:1792-1793`）。

---

## 4. 可观察差异速查（同一份 SPIR-V，两个后端）

1. **偏移可见性**：GLSL 常规输出零偏移数字（仅块级 `std140/std430/scalar`）；HLSL cbuffer 每成员必有 `packoffset(cN[.swizzle])`，SSBO 则把偏移内联进 `Load/Store` 整数参数。
2. **行/列主序字面量**：GLSL 只输出 `row_major`（列主序省略）；HLSL 两者都可能输出且**含义与 SPIR-V 装饰相反**（`spirv_hlsl.cpp:2705-2708`），同时乘法被改写为反序 `mul`。
3. **限定符位置**：GLSL 禁止裸 struct 内布局限定符，嵌套结构的行主序上提到块成员；HLSL 允许 struct 内任意位置使用 `row_major`/`column_major`。
4. **不可表达布局的行为**：GLSL 可借助增强布局回退到成员级 `layout(offset=...)`；HLSL cbuffer 无法回退时直接抛异常（典型：std140 标量数组 16B 步长、子结构不符合 HLSL 打包），而同类 SSBO 因走 ByteAddressBuffer 裸寻址始终可表达。
5. **数组步长**：GLSL 隐含；HLSL 在 SSBO 路径显式成为 `index * stride` 表达式，在 cbuffer 路径不可表达、只能依赖隐式规则（`ConstantBuffer<T>` 路径，SM 5.1+）。
6. **矩阵标量寻址**：HLSL 行主序矩阵的两个方向步长不同（`width/8` 与 `MatrixStride`），在生成代码中肉眼可辨；GLSL 不产生地址运算，只在必要时产生 `transpose()`/索引交换。

## 5. 复现方法（全程离线，无 GPU）

```bash
# 1. 编译本仓库（任意独立目录）
g++ -std=c++11 -O2 -DNDEBUG -I<repo> -c <repo>/spirv_*.cpp <repo>/main.cpp
ar rcs libspirv-cross.a spirv_*.o && g++ -o spirv-cross main.o libspirv-cross.a

# 2. GLSL -> SPIR-V（glslang），再反汇编查看装饰
glslangValidator -V -o x.spv x.comp
spirv-dis x.spv | grep -E "MemberDecorate|ArrayStride|MatrixStride"

# 3. 两后端输出与反射
spirv-cross x.spv -V                          # Vulkan GLSL
spirv-cross x.spv --hlsl                      # HLSL
spirv-cross x.spv --reflect                   # 含 offset/array_stride/matrix_stride 的 JSON
```

仓库内现成证据：`reference/shaders-hlsl/comp/rwbuffer-matrix.comp`（SSBO 矩阵字节寻址）、`reference/shaders-hlsl/vert/read-from-row-major-array.vert`（矩阵数组 + 主序翻转 + packoffset）、`reference/shaders-hlsl/frag/row-major-layout-in-struct.frag`（嵌套结构主序）、`reference/shaders/desktop-only/comp/enhanced-layouts.comp`（GLSL 成员级 offset）、`reference/shaders-hlsl-no-opt/frag/constant-buffer-array.invalid.sm51.frag`（`ConstantBuffer<T>` 数组块）。
