# SPIRV-Cross 分离纹理/采样器（Separate Image & Sampler）组合语义报告

> 范围：SPIR-V 输入中 `OpTypeImage`（`Sampled=1`，Vulkan GLSL 的 `texture2D` 等分离纹理）与
> `OpTypeSampler`（`sampler`）经 `OpSampledImage` 临时组合的资源，在降级到不原生支持分离资源的
> 目标语言时如何被“组合（combine）”。本文对比 **GLSL/ESSL 后端**（`spirv_glsl.cpp`，HLSL SM30
> 路径复用其机制）与 **MSL 后端**（`spirv_msl.cpp`，Metal 原生分离）两套语义，并以 HLSL 现代路径
> 作为旁证。GPU 驱动层（MoltenVK 等）不在范围内。
>
> 仓库版本：分支 `task/13`，HEAD `57e4efc`。所有引用均给出 `文件:行号`，可直接跳转。

---

## 1. 背景与核心数据结构

Vulkan/SPIR-V 的资源模型允许纹理与采样器分离声明、在着色器内用 `OpSampledImage` 按需配对：

```
%uTex     = OpVariable ... OpTypeImage          ; layout(binding=1) uniform texture2D uTex;
%uSampler = OpVariable ... OpTypeSampler         ; layout(binding=0) uniform sampler uSampler;
%19       = OpSampledImage %18 %13 %17           ; sampler2D(uTex, uSampler)
%24       = OpImageSampleImplicitLod ... %19 ... ; texture(sampler2D(...), uv)
```

而桌面/ES GLSL（非 Vulkan 语义）只有 `sampler2D` 这种**一体的组合采样器类型**，没有独立的
`sampler` 变量。SPIRV-Cross 因此需要在 IR 层把每一对实际出现过的 `(image, sampler)` 物化成一个
新的 `SampledImage` 变量。

### 1.1 三元组身份：`CombinedImageSampler`

`spirv_cross.hpp:145-153` 定义了组合关系的最小单元：

```cpp
struct CombinedImageSampler
{
    VariableID combined_id; // 新合成的 sampler2D 变量 ID（SPIRV-Cross 自己分配）
    VariableID image_id;    // 来源 texture2D 变量 ID（SPIR-V 原始 ID）
    VariableID sampler_id;  // 来源 sampler  变量 ID（SPIR-V 原始 ID）
};
```

- 组合的**身份键是 `(image_id, sampler_id)` 有序对**，而不是任何一个单点资源。
  注册处 `spirv_cross.cpp:3239-3300` 在 push 前先按这对 ID 去重：同一对在任意多处
  `OpSampledImage` 出现只生成一个组合变量（见 3.1 的复用反证）。
- `combined_id` 是通过 `ir.increase_bound_by(2)` 新造的 `OpTypeSampledImage` 指针 + 变量
  （`spirv_cross.cpp:3265-3280`），**原始 SPIR-V 中不存在该 ID**，因此它天然没有
  `DescriptorSet/Binding/Name` 装饰。
- 合成时只显式继承了 `RelaxedPrecision`（三者任一有标记即继承，`spirv_cross.cpp:3283-3289`），
  以及图像侧的数组维度（`spirv_cross.cpp:3291-3298`）。**绑定号默认不继承**。

### 1.2 反射侧的原始分类不受组合影响

`Compiler::get_shader_resources()` 按 SPIR-V **原始变量类型**分类（`spirv_cross.cpp:1210-1234`）：

| SPIR-V 类型条件 | 反射列表 | Vulkan GLSL 对应 |
|---|---|---|
| `Image` 且 `image.sampled == 2` | `storage_images` | `image2D`（读写图像） |
| `Image` 且 `image.sampled == 1` | `separate_images` | `texture2D` |
| `Sampler` | `separate_samplers` | `sampler` |
| `SampledImage` | `sampled_images` | 原生 `sampler2D` |

头文件明确说明（`spirv_cross.hpp:454-455`）：

> "If any image/sampler remapping was found, no separate image/samplers will appear in the
> decompiled output, **but will still appear in reflection**."

即**代码生成与反射是两条独立通道**：组合只改变输出着色器里的变量形态，不删除
`separate_images/separate_samplers` 反射记录；合成的 `combined_id` 反过来会作为新的
`sampled_images` 资源出现在组合后的反射里（`spirv_cross.hpp:451-452`）。

### 1.3 驱动 API 的入口

- C++：`Compiler::build_combined_image_samplers()`（`spirv_cross.hpp:463`，实现
  `spirv_cross.cpp:3336-3347`）、`get_combined_image_samplers()`（`spirv_cross.hpp:466`）、
  `build_dummy_sampler_for_combined_images()`（`spirv_cross.hpp:443`）。
- C ABI：`spvc_compiler_build_combined_image_samplers`、
  `spvc_compiler_get_combined_image_samplers`（`spirv_cross_c.h:1039-1041`，
  实现 `spirv_cross_c.cpp:2500-2523`，逐字拷贝三个 ID）。
- CLI：非 Vulkan 语义的 GLSL 默认自动开启（`main.cpp:1342`
  `combined_image_samplers = !args.vulkan_semantics;`），合成后统一改名
  （`main.cpp:1609-1613`）：

```cpp
compiler->set_name(remap.combined_id,
    join("SPIRV_Cross_Combined", get_name(remap.image_id), get_name(remap.sampler_id)));
```

---

## 2. 组合发生的位置：全局变量与函数参数两条路径

`OpSampledImage` 的处理在核心编译器的 `CombinedImageSamplerHandler`
（`spirv_cross.cpp:2947-3304`）中，分两种情形：

### 2.1 全局图像 + 全局采样器

直接进入全局表 `combined_image_samplers`（`spirv_cross.cpp:3239-3300`）。
代码生成时 `CompilerGLSL::to_combined_image_sampler()` 查全局映射表并返回
`to_expression(itr->combined_id)`（`spirv_glsl.cpp:8274-8286`）。

### 2.2 任一操作数是函数形参

当 image 或 sampler 来自函数参数时，不能只造全局变量，需要给被调函数**追加“影子形参”**：

- `register_combined_image_sampler()` 把参数位次记录到
  `SPIRFunction::combined_parameters`（`spirv_cross.cpp:2955-3021`，结构见
  `spirv_common.hpp:1079`），并新建一个 `StorageClassFunction` 的局部 SampledImage 变量。
- 代码生成时，原始的分离 image/sampler 实参被 `skip_argument()` 跳过
  （`spirv_glsl.cpp:12547-12556`：`Sampler` 或 `Image(sampled==1)` 类型一律不传），
  改由调用点把组合结果作为新实参追加（`spirv_glsl.cpp:13547-13552`），
  函数签名侧通过 `shadow_arguments` 补出对应形参（`spirv_glsl.cpp:18155-18169`）。
- 查不到映射会直接抛异常（`spirv_glsl.cpp:8266-8268` 与 `8283-8284`）：

> "Cannot find mapping for combined sampler, was build_combined_image_samplers() used
> before compile() was called?"

这意味着**组合必须在 `compile()` 之前完成**，是一次性的 IR 改写，不支持编译后增量查询。

---

## 3. GLSL/ESSL 后端：真正的“物化组合”语义

### 3.1 可观察的生成接口

**输入**（`shaders/vulkan/frag/separate-sampler-texture.vk.frag:4-8`）：1 个 `sampler uSampler`
(binding=0) + 4 个 `texture*`（binding=1..4），主函数与两个辅助函数里共出现 5 种
`(image, sampler)` 配对。

**Vulkan 语义输出**（`-V`，不组合），保留两个独立资源与原始 set/binding
（`reference/opt/shaders/vulkan/frag/separate-sampler-texture.vk.frag.vk:5-9`）：

```glsl
layout(set = 0, binding = 1) uniform mediump texture2D uTexture;
layout(set = 0, binding = 0) uniform mediump sampler uSampler;
...
FragColor = ... texture(sampler2D(uTexture, uSampler), ...) ...
```

**默认 GLSL 输出**（组合后，`reference/opt/shaders/vulkan/frag/separate-sampler-texture.vk.frag:5-8`）：

```glsl
uniform mediump sampler2D SPIRV_Cross_CombineduTextureuSampler;
uniform mediump sampler2DArray SPIRV_Cross_CombineduTextureArrayuSampler;
uniform mediump samplerCube SPIRV_Cross_CombineduTextureCubeuSampler;
uniform mediump sampler3D SPIRV_Cross_CombineduTexture3DuSampler;
```

可观察事实：

1. **分离的 `texture2D`/`sampler` 声明完全消失**。资源发射循环里
   `skip_separate_image_sampler = !combined_image_samplers.empty() || !options.vulkan_semantics`
   （`spirv_glsl.cpp:4199`），命中后 `Image(sampled==1)` 与 `Sampler` 变量被 `return` 跳过
   （`spirv_glsl.cpp:4206-4214`）。
2. 输出的组合变量**没有 `layout(set=, binding=)`**——因为 `combined_id` 是新 ID，没有装饰。
   宿主必须自行分配绑定，或调用 `inherit_combined_sampler_bindings`（见 3.3）。
3. 同一对 `(uTex, uSampler)` 在两条采样指令里复用，只生成一个变量。
   汇编码证据 `shaders/asm/frag/combined-sampler-reuse.asm.frag:47-52` 中一个 `%19 = OpSampledImage`
   被两次 `OpImageSample*` 使用，GLSL 输出只有一个
   `SPIRV_Cross_CombineduTexuSampler`（`reference/opt/shaders/asm/frag/combined-sampler-reuse.vk.asm.frag:5`）。
4. 原生 `sampler2D` 与合成组合变量可以共存：
   `tests-other/msl_constexpr_test.spv` 的默认 GLSL 输出同时含有原生
   `layout(binding=2) uniform sampler2D uSampler;` 与合成的
   `uniform sampler2D SPIRV_Cross_CombineduImage2uSampler2;`（本机用构建出的
   `./spirv-cross tests-other/msl_constexpr_test.spv` 实测可见）。

### 3.2 数组语义：图像数组被逐元素继承，采样器数组不支持

头文件契约（`spirv_cross.hpp:461-462`）：

> "Arrays of separate samplers are not supported, but arrays of separate images are supported.
> Array of images + sampler -> Array of combined image samplers."

实现上，合成变量的数组类型直接从图像变量拷贝（`spirv_cross.cpp:3291-3298`）。
输入 `texture2D uTexture[4]` + 标量 `sampler uSampler`
（`shaders/vulkan/frag/separate-sampler-texture-array.vk.frag:4-8`），输出
（`reference/opt/shaders/vulkan/frag/separate-sampler-texture-array.vk.frag:5-8`）：

```glsl
uniform mediump sampler2D SPIRV_Cross_CombineduTextureuSampler[4];
...
texture(SPIRV_Cross_CombineduTextureuSampler[2], ...)   // 数组下标表达式被原样保留
```

`to_combined_image_sampler()` 会把图像表达式上的 `[index]` 后缀切下来贴到组合变量后面
（`spirv_glsl.cpp:8223-8229`），因此动态/常量下标都能正确重定向。

**运行时（unsized）采样器数组是组合的硬边界**：`sampler uSamps[]` 与
`texture2D uSamplers[]` 用动态下标配对时
（`shaders/vulkan/frag/nonuniform-qualifier.vk.nocompat.frag:5-8,30`），静态物化需要
N×M 个笛卡尔积组合变量，无法表达。该用例文件名带 `.nocompat.`，测试框架对它**跳过普通
GLSL 生成与校验**（`test_shaders.py:757-767`），只保留 `-V` Vulkan 输出。同一份 IR 在
MSL/HLSL 则可原生表达（见第 4、5 节）。

### 3.3 绑定号与反射信息在组合前后的关系

| 信息 | 组合前（image / sampler） | 组合后（combined_id）默认 | 开启 `--combined-samplers-inherit-bindings` |
|---|---|---|---|
| `Name` | `uTexture` / `uSampler` | CLI 改名为 `SPIRV_Cross_Combined<image><sampler>`（`main.cpp:1611`） | 同名 |
| `DescriptorSet` | 各自的 set | **丢失**（新 ID 无装饰） | 复制 **image 的 set** |
| `Binding` | 各自的 binding | **丢失** | 复制 **image 的 binding** |
| `RelaxedPrecision` | 各自标记 | 三者 OR 继承（`spirv_cross.cpp:3283-3289`） | 同左 |
| 数组维数 | image 上的数组 | 从 image 拷贝（`spirv_cross.cpp:3296`） | 同左 |
| 比较/阴影属性 | sampler 是否用于 Dref | 经 `analyze_image_and_sampler_usage()` 传播到 combined_id（`spirv_cross.cpp:4819-4822`） | 同左 |

继承逻辑只有四行、且**只看 image 一侧**（`spirv_cross_util.cpp:59-76`）：

```cpp
if (compiler.has_decoration(s.image_id, DecorationDescriptorSet))
    compiler.set_decoration(s.combined_id, DecorationDescriptorSet,
        compiler.get_decoration(s.image_id, DecorationDescriptorSet));
// Binding 同理，取自 image_id
```

本机实测对照（`tests-other/msl_constexpr_test.spv`，image 在 set=2/binding=2，sampler 在
set=2/binding=4）：

```
$ ./spirv-cross msl_constexpr_test.spv
uniform sampler2D SPIRV_Cross_CombineduImage2uSampler2;                 # 无绑定

$ ./spirv-cross msl_constexpr_test.spv --combined-samplers-inherit-bindings
layout(binding = 2) uniform sampler2D SPIRV_Cross_CombineduImage2uSampler2;  # 继承 image 的 2
```

注意 sampler 的 binding=4 被丢弃——这是合理的，因为 GLSL 侧只剩一个资源槽位。
**当多个不同 image 共用同一 sampler、或同一 image 配对多个 sampler 时，组合变量数等于实际
配对数（笛卡尔积中“被用到”的那些格子），每个各占一个独立绑定槽**，绑定继承只保证与 image
对齐，宿主仍需保证槽位不冲突。

反射 JSON（`--reflect`，由 `spirv_reflect.cpp:549-551` 输出）始终报告原始的
`separate_images` / `separate_samplers` 及其 set/binding/数组，例如
`reference/shaders-reflection/frag/separate-sampler-texture-array.vk.frag.json:27-84`
完整列出 4 个 `separate_images`（binding 1..4，各带 `"array":[4]`）与 1 个
`separate_sampler`（binding 0）。宿主的推荐工作流是：读反射拿原始集合 → 调
`get_combined_image_samplers()` 拿三元组 → 用三元组里的 image_id 回反射表查 set/binding →
自行（或借助 inherit 工具）给 combined_id 赋绑定。

### 3.4 Dummy sampler：无采样器指令的边界

`OpImageFetch/OpImageQuerySize*/OpImageQueryLevels/OpImageQuerySamples` 只吃 `Image` 不需要
sampler，但 GLSL 的 `texelFetch(sampler2D, ...)` 语法仍要求组合类型。
`build_dummy_sampler_for_combined_images()` 扫描这些操作码
（`spirv_cross.cpp:3062-3077`），需要时合成名为 `SPIRV_Cross_DummySampler` 的虚拟 sampler
（`spirv_cross.cpp:3306-3334`），随后与 image 配对成
`SPIRV_Cross_Combined<image>SPIRV_Cross_DummySampler`。GLSL 侧若没先建 dummy 会抛
"Cannot find dummy sampler ID"（`spirv_glsl.cpp:8849-8851`）。

这是**反例一**的素材，见第 6 节。

---

## 4. MSL 后端：不组合，只拆分成 Metal 的原生参数空间

Metal 着色语言本身就是分离模型：`texture2d<T> [[texture(n)]]` 与 `sampler [[sampler(n)]]`
是两类独立参数，各自有独立的索引空间。因此 **MSL 后端根本不使用
`combined_image_samplers` 表**——全仓 grep 可证 `spirv_msl.cpp`/`spirv_msl.hpp` 对
`combined_image_samplers` 的引用数为 0；即使宿主调用 `build_combined_image_samplers()`，
MSL 代码路径也不读取它。

### 4.1 `OpSampledImage` 在 MSL 里只是表达式节点，不产生变量

`CompilerMSL::emit_sampled_image_op()`（`spirv_msl.cpp:13377-13381`）：

```cpp
void CompilerMSL::emit_sampled_image_op(...) {
    set<SPIRCombinedImageSampler>(result_id, result_type, image_id, samp_id);
}
```

它只在 IR 表达式里记一下“这俩配过对”，后续 `to_sampler_expression()`
（`spirv_msl.cpp:13737-13760`）在采样点把 `.sampler` 一侧取出来，最终生成
`tex.sample(samp, uv)` 形式的**双参数调用**，而不是新 uniform。

### 4.2 一个 SPIR-V `SampledImage` 变量拆成两个入口参数

入口参数发射 `entry_point_args_discrete_descriptors()` 对每个
`SPIRType::SampledImage` 变量压入两条 Resource（`spirv_msl.cpp:15603-15620`）：

```cpp
if (type.basetype == SPIRType::SampledImage) {
    resources.push_back({..., to_name(var_id), SPIRType::Image,
                         get_metal_resource_index(var, SPIRType::Image, i), ...});
    if (type.image.dim != DimBuffer && !constexpr_sampler)
        resources.push_back({..., to_sampler_expression(var_id), SPIRType::Sampler,
                             get_metal_resource_index(var, SPIRType::Sampler), ...});
}
```

同一份 `combined-sampler-reuse` SPIR-V（分离的 uTex binding=1、uSampler binding=0），
MSL 输出（`reference/opt/shaders-msl/asm/frag/combined-sampler-reuse.asm.frag:16`）：

```metal
fragment main0_out main0(main0_in in [[stage_in]],
    texture2d<float> uTex [[texture(0)]],
    sampler uSampler [[sampler(0)]])
{
    out.FragColor = uTex.sample(uSampler, in.vUV);
```

命名规则：纹理保留原名；采样器名 = 纹理名 + 后缀（默认 `Smplr`，定义于
`spirv_msl.hpp:1339`，可用 `set_combined_sampler_suffix` 改，
`spirv_msl.cpp:21067-21073`；CLI `--msl-combined-sampler-suffix`，`main.cpp:1335-1336`）。原生分离变量（本就是
`Image`+`Sampler`）同样各自独立命名，见
`reference/opt/shaders-msl/frag/sample-depth-separate-image-sampler.frag`：两个纹理占
`texture(0/1)`、两个采样器独立占 `sampler(0/1)`，其中 comparison sampler 配合
`spvDepthCast` 模板生成 `sample_compare`。

### 4.3 绑定号：两套独立索引空间，自动分配或显式 remap

`get_metal_resource_index()`（`spirv_msl.cpp:16475-16589`）的规则：

1. 优先查宿主通过 `add_msl_resource_binding()` 注册的 `MSLResourceBinding`
   （结构 `spirv_msl.hpp:106-116`，一次可带 `msl_buffer/msl_texture/msl_sampler` 三个号）；
2. 否则取已缓存的扩展装饰；`enable_decoration_binding` 时可回退到 SPIR-V 的 binding 号；
3. 再否则按类型从三个**互相独立**的计数器自增：
   `next_metal_resource_index_texture / _sampler / _buffer`
   （`spirv_msl.cpp:16570-16584`）。

SampledImage 的 sampler 分量走 **secondary** 装饰位
（`spirv_msl.cpp:16487-16491`），即一个变量可同时携带
`SPIRVCrossDecorationResourceIndexPrimary`（纹理号）与
`...Secondary`（采样器号），枚举见 `spirv_common.hpp:1834-1840`，反射读取 API 为
`get_automatic_msl_resource_binding(_secondary)`（`spirv_msl.cpp:220-228`）。
多平面（YCbCr）图像再占用 Tertiary/Quaternary。

测试框架的显式 remap 注释把模型说得很直白（`test_shaders.py:394-402`）：

> "Set 0, binding 0 gets texture index 0 and sampler index 1, like a combined image sampler
> in MoltenVK." —— 参数序列 `--msl-resource-binding 0 0 0 0 1` 即
> `(desc_set, binding, buffer_idx, texture_idx, sampler_idx)`。

### 4.4 别名（aliasing）：同一 set/binding 上的多个变量

SPIR-V 允许不同类型的多个变量声明在同一个 `(set, binding)` 上（描述符互斥使用）。
MSL 离散描述符模式下只允许很有限的别名（buffer 间用指针 cast），且**带数组的别名直接
抛异常**（`spirv_msl.cpp:15574-15579`）：

> "Attempting to alias same binding with a descriptor array which is not implemented through
> argument buffers. This is unsupported."

完整的可变别名（mutable aliasing）需要参数缓冲（argument buffer）且 Metal 3+，否则抛
（`spirv_msl.cpp:20827-20830`、`20885-20891`）：

> "Full mutable aliasing of argument buffer descriptors only works on Metal 3+."

参数缓冲下，同 index 的资源被标记 `SPIRVCrossDecorationOverlappingBinding`
（`spirv_msl.cpp:20689-20702`），主声明只发一次，其余以注释 + `reinterpret_cast` 别名出现。
实证 `shaders-msl/frag/overlapping-combined-image-samplers.msl3.argument.resource-binding.frag`
（三个 `sampler*` 全在 set=0/binding=0）的输出
（`reference/opt/shaders-msl/frag/overlapping-combined-image-samplers...frag:32-58`）：

```metal
struct spvDescriptorSetBuffer0 {
    texture2d<float> s0_2d [[id(0)]];
    // Overlapping binding: texture2d<float> s0_2d_shadow [[id(0)]];
    // Overlapping binding: texturecube<float> s0_cube [[id(0)]];
    sampler s0_2dSmplr [[id(1)]];
    // Overlapping binding: sampler s0_2d_shadowSmplr [[id(1)]]; ...
};
...
constant auto &s0_2d_shadow = reinterpret_cast<constant texture2d<float> &>(spvDescriptorSet0.s0_2d);
```

即 MSL 侧“组合前后”根本不存在身份合并：三个 SPIR-V 变量在参数缓冲里坍缩为一个纹理
`id(0)` + 一个采样器 `id(1)`，靠 cast 恢复不同视图类型。这与 GLSL 侧“每个用到的配对物化为
独立 uniform”是完全不同的身份模型。

### 4.5 constexpr sampler：采样器完全不作为资源出现

通过 `MSLConstexprSampler` 选项 remap 后，sampler 变成立即数局部变量
（`spirv_msl.cpp:15585-15594, 15622-15624` 显式跳过资源登记），不占 `sampler(n)` 槽、
不进 `entry_point_bindings`。测试见 `tests-other/msl_constexpr_test.cpp:91-108`。

---

## 5. 两后端语义差异对照（含 HLSL 现代路径旁证）

| 维度 | GLSL/ESSL（非 Vulkan） | MSL | HLSL SM40+（旁证，`spirv_hlsl.cpp`） |
|---|---|---|---|
| 目标语言是否原生分离 | 否，只有 `sampler2D` 一体类型 | 是，`texture2d`/`sampler` 原生分离 | 是，`Texture2D`/`SamplerState` 分离 |
| 组合手段 | 新建真实 `SampledImage` **变量**，每对一个 | 不组合，只建 IR 表达式节点 | 默认不组合（`spirv_hlsl.cpp:3053-3056`）；SM30 或显式开启时走 GLSL 同款物化 |
| 原始 image/sampler 声明 | 被抑制不输出（`spirv_glsl.cpp:4206-4214`） | 全部保留为入口参数 | 保留；SampledImage 另自动补一个 `_sampler` 后缀的 SamplerState（`spirv_hlsl.cpp:4458-4467`） |
| 绑定模型 | 单一 uniform 槽，默认无 set/binding，可从 image 继承 | texture/sampler/buffer **三个独立索引空间**，primary/secondary 双装饰 | `t`/`s`/`b`/`u` 四类 register + space，自动按 SPIR-V binding 落号（`spirv_hlsl.cpp:4308-4316, 4368-4373`） |
| 命名 | `SPIRV_Cross_Combined<Img><Samp>` | 纹理原名；采样器 `<纹理名>Smplr` | 纹理原名；自动采样器 `_<名>_sampler`（`spirv_hlsl.cpp:3036-3048`） |
| 同一份 `combined-sampler-reuse` 输出 | 1 个合成 `sampler2D` uniform | 1 个 `texture2d` + 1 个 `sampler` 参数 | `Texture2D uTex : register(t1)` + `SamplerState uSampler : register(s0)`（`reference/opt/shaders-hlsl/asm/frag/combined-sampler-reuse.asm.frag:1-2`） |
| 阴影/比较 | Dref 分析传播到合成变量，选 `sampler2DShadow` | `spvDepthCast` + `sample_compare`，sampler 类型不变 | 自动选 `SamplerComparisonState`（`spirv_hlsl.cpp:4461-4463, 4472-4473`） |
| 反射 | 原始分离资源仍在反射；三元组另表可查 | `get_shader_resources()` 原样返回；自动索引用扩展装饰查 | 同核心反射；register 号体现在生成文本 |
| 函数参数 | 影子形参改写签名（2.2 节） | 形参直接拆成 `(texture2d t, sampler s)`（`spirv_msl.cpp:12549-12564`，实证 `reference/opt/shaders-msl/frag/separate-image-sampler-argument.frag:14`） | 现代路径自动补 sampler 实参（`spirv_hlsl.cpp:3071-3074` 附近） |
| 运行时数组动态配对 | **不支持**（nocompat，跳过生成） | 原生 `array<texture2d,N>` + `array<sampler,M>`（实证 `reference/shaders-msl/frag/nonuniform-qualifier.msl2.frag:27`） | 原生 `Texture2D ... [] : register(t...)` 与 `SamplerState ... [] : register(s...)`（`reference/shaders-hlsl/frag/nonuniform-qualifier.nonuniformresource.sm51.frag:8-11`） |
| 别名 | 无对应概念；每对独立 uniform | 离散模式受限/抛错，参数缓冲 + Metal3 才完整支持 | 靠 register(space) 复用，无物化冲突 |

HLSL 的 SM30 路径是 GLSL 物化机制的复用实证：`main.cpp:1514-1518` 在
`shader_model <= 30` 时强制 `combined_image_samplers = true` 并建 dummy sampler，
`spirv_hlsl.cpp:1745` 与 `4492`（"Separate image and samplers not supported in legacy
HLSL."）表明 SM30 与老式 GLSL 处于同一能力象限。输出对照见
`reference/opt/shaders-hlsl/frag/separate-combined-fake-overload.sm30.frag:1-2`（合成
`sampler2D SPIRV_Cross_CombineduTuS`），而同一着色器的现代 GLSL 物化结果在
`reference/opt/shaders/vulkan/frag/separate-combined-fake-overload.vk.frag:3-4`。

---

## 6. 无法安全组合的边界与两个反例

### 6.1 边界清单（附源码位置）

1. **运行时采样器数组 + 动态下标配对**：静态物化会产生无界笛卡尔积，GLSL 路径标记
   `.nocompat.` 跳过（`test_shaders.py:757-767`；头文件声明
   `spirv_cross.hpp:461`）。MSL/HLSL 因原生分离不受影响。
2. **无 sampler 的 fetch/query 指令**：必须注入 dummy sampler；若未先调用
   `build_dummy_sampler_for_combined_images()` 即编译，GLSL 抛
   "Cannot find dummy sampler ID"（`spirv_glsl.cpp:8849-8851`）。dummy 变量没有真实
   描述符支撑，合成出的 uniform 绑定语义是悬空的。
3. **组合必须先于 compile**：顺序错误时 `to_combined_image_sampler` 抛
   "was build_combined_image_samplers() used before compile()"（`spirv_glsl.cpp:8266-8268, 8283-8284`）。
4. **MSL 离散描述符上的数组别名**：直接 throw（`spirv_msl.cpp:15578`）；
   完整可变别名要求参数缓冲 + Metal 3（`spirv_msl.cpp:20830, 20891`）。
5. **legacy HLSL（SM≤30）遇到无法物化的分离资源**：throw
   （`spirv_hlsl.cpp:4492`）。
6. **函数重载哈希失真风险**：组合后 image/sampler 形参被影子参数替换，后端在重载判定时
   必须显式忽略这些类型，否则伪重载会冲突（`spirv_glsl.cpp:18050-18061`）。
7. **绑定继承只覆盖 image 侧**：sampler 的 set/binding 在 GLSL 组合后无槽位可承载
   （`spirv_cross_util.cpp:64-74`），依赖采样器独立绑定的管线状态无法在输出 GLSL 中表达。

### 6.2 反例一：Dummy sampler 制造出没有真实资源支撑的“组合”

**输入**：`shaders/asm/frag/image-fetch-no-sampler.asm.vk.invalid.frag`（HLSL 源生成的
SPIR-V），其中 `OpImageFetch` 直接作用于分离 image，没有任何 `OpSampledImage`。
文件名 `.invalid.` 表示该 SPIR-V 对纯 GLSL 降级本身就是无效场景
（`test_shaders.py:925-926`：`shader_is_invalid_spirv = '.invalid.' in shader`）。

**组合后的 GLSL 输出**（`reference/opt/shaders/asm/frag/image-fetch-no-sampler.asm.vk.invalid.frag:3-12`）：

```glsl
uniform sampler2D SPIRV_Cross_CombinedparamSPIRV_Cross_DummySampler;
uniform sampler2D SPIRV_Cross_CombinedSampledImageSPIRV_Cross_DummySampler;
...
vec4 sample_fetch(ivec3 UV, sampler2D SPIRV_Cross_CombinedtexSPIRV_Cross_DummySampler)
{
    return texelFetch(SPIRV_Cross_CombinedtexSPIRV_Cross_DummySampler, UV.xy, UV.z);
}
```

为什么不安全：

- `SPIRV_Cross_DummySampler` 是编译器凭空造的变量（`spirv_cross.cpp:3328`），**原始反射里
  不存在对应描述符**，宿主没有任何真实 sampler 可以绑到这些组合 uniform 上；它们只是为了让
  `texelFetch` 通过 GLSL 语法检查而存在的形式占位。
- 同一份 IR 的 Vulkan 语义输出则走 `GL_EXT_samplerless_texture_functions`，完全不需要组合
  （`reference/opt/shaders/asm/frag/image-fetch-no-sampler.asm.vk.invalid.frag.vk:2,9-12`
  的 `texelFetch(tex, ...)` 直接接受 `texture2D`）。这证明问题不是源数据缺失，而是目标语言
  能力缺口只能靠伪资源填补——生成接口表面上与正常组合 uniform 无异，宿主无法从签名上区分
  真伪，只能靠 `.invalid.` 约定与对 dummy 名字的特判规避。

### 6.3 反例二：动态索引的运行时纹理/采样器数组无法静态物化

**输入**：`shaders/vulkan/frag/nonuniform-qualifier.vk.nocompat.frag:5-8,30`：

```glsl
layout(binding = 0) uniform texture2D uSamplers[];
layout(binding = 1) uniform sampler   uSamps[];
...
FragColor = texture(nonuniformEXT(sampler2D(uSamplers[i + 10], uSamps[i + 40])), vUV);
```

两个操作数都是 **unsized 运行时数组、下标都是运行时表达式**（且带 `nonuniformEXT`）。

为什么不安全：

- GLSL 物化要求为每个静态可辨识的 `(image_id, sampler_id)` 对造一个编译期变量
  （去重逻辑 `spirv_cross.cpp:3239-3244`）。这里配对身份要到运行时才确定，可能的组合数为
  两数组长度的乘积且编译期未知，无法枚举；头文件亦明确 "Arrays of separate samplers are
  not supported"（`spirv_cross.hpp:461`）。
- 测试框架的可观察行为是：普通 GLSL 参考文件**根本不生成**
  （`test_shaders.py:758-767` 对 nocompat 分支 `remove_file(glsl_path)`），只产出 `-V`
  Vulkan 版本；即工具链以“拒绝该目标”而非“生成错误代码”的方式处理边界。
- 同一 IR 在原生分离后端上完全合法，形成对照：
  - MSL（`reference/shaders-msl/frag/nonuniform-qualifier.msl2.frag:27,45`）：
    `array<texture2d<float>, 8> uSamplers [[texture(0)]]` 与
    `array<sampler, 7> uSamps [[sampler(0)]]` 保持两个独立数组，调用点
    `uSamplers[_25].sample(uSamps[_37], in.vUV)` 运行时配对；
  - HLSL SM51（`reference/shaders-hlsl/frag/nonuniform-qualifier.nonuniformresource.sm51.frag:8-9`）：
    `Texture2D ... uSamplers[] : register(t0, space0)` 与
    `SamplerState uSamps[] : register(s1, space3)`。

结论：组合安全性的充要条件是**配对关系在编译期可静态枚举且每个合成变量都有可绑定的真实
描述符**。GLSL 后端靠“静态物化 + 可选绑定继承”满足前者的常规情形，靠 dummy sampler 掩盖
fetch 类语法缺口，对动态数组则直接放弃；MSL/HLSL 因为目标语言原生分离，全程不做身份合并，
从而绕开了全部三类边界。

---

## 7. 引用索引

### 源码

| 主题 | 位置 |
|---|---|
| `CombinedImageSampler` 三元组 | `spirv_cross.hpp:145-153` |
| 组合 API 契约（含数组限制、反射说明） | `spirv_cross.hpp:429-469` |
| 全局配对去重与合成变量 | `spirv_cross.cpp:3239-3300` |
| 函数参数组合 / 影子形参 | `spirv_cross.cpp:2947-3021`；`spirv_glsl.cpp:12547-12556, 13547-13552, 18155-18169` |
| `build_combined_image_samplers` | `spirv_cross.cpp:3336-3347` |
| Dummy sampler 扫描与创建 | `spirv_cross.cpp:3030-3077, 3306-3334`；`spirv_glsl.cpp:8849-8853` |
| 绑定继承（仅 image 侧） | `spirv_cross_util.cpp:59-76` |
| GLSL 抑制分离资源 / 组合表达式 | `spirv_glsl.cpp:4199-4214, 8221-8287, 8329-8340` |
| GLSL 重载哈希忽略组合类型 | `spirv_glsl.cpp:18050-18061` |
| MSL 不组合、仅表达式节点 | `spirv_msl.cpp:13377-13381, 13737-13760` |
| MSL SampledImage 拆双参数 | `spirv_msl.cpp:15603-15620, 12549-12564` |
| MSL 三套索引空间 / primary-secondary | `spirv_msl.cpp:16475-16589`；`spirv_common.hpp:1834-1840`；`spirv_msl.cpp:220-228` |
| MSL 别名限制与异常 | `spirv_msl.cpp:15556-15582, 20689-20702, 20827-20835, 20885-20891` |
| HLSL 现代分离 / SM30 物化 / legacy 异常 | `spirv_hlsl.cpp:1745-1760, 3036-3062, 4443-4498`；`main.cpp:1514-1518` |
| 反射分类与 JSON 输出 | `spirv_cross.cpp:1210-1234`；`spirv_reflect.cpp:544-555` |
| C ABI | `spirv_cross_c.h:1039-1041`；`spirv_cross_c.cpp:2500-2523` |
| CLI 自动化（开启、改名、继承） | `main.cpp:1342-1345, 1541-1544, 1602-1614` |

### 测试与参考输出

| 证据 | 路径 |
|---|---|
| 基础分离→GLSL 组合（含 Vulkan 对照） | `shaders/vulkan/frag/separate-sampler-texture.vk.frag`；`reference/opt/shaders/vulkan/frag/separate-sampler-texture.vk.frag{,.vk}` |
| 图像数组继承 | `shaders/vulkan/frag/separate-sampler-texture-array.vk.frag`；`reference/opt/shaders/vulkan/frag/separate-sampler-texture-array.vk.frag` |
| 反射 JSON（原始分离资源保留） | `reference/shaders-reflection/frag/separate-sampler-texture-array.vk.frag.json` |
| 配对复用（三后端同输入对照） | `shaders/asm/frag/combined-sampler-reuse.asm.frag`；`reference/opt/shaders/{asm,shaders-msl/asm,shaders-hlsl/asm}/frag/combined-sampler-reuse.*` |
| MSL 函数参数拆分 | `shaders-msl/frag/separate-image-sampler-argument.frag`；`reference/opt/shaders-msl/frag/separate-image-sampler-argument.frag` |
| MSL 阴影/比较分离 | `reference/opt/shaders-msl/frag/sample-depth-separate-image-sampler.frag` |
| MSL 同绑定别名 | `shaders-msl/frag/overlapping-combined-image-samplers.msl3.argument.resource-binding.frag` 及同名 reference |
| HLSL 自动 SamplerState / 比较状态 | `reference/opt/shaders-hlsl/frag/combined-texture-sampler-parameter.frag`、`combined-texture-sampler-shadow.frag` |
| SM30 物化复用 | `shaders-hlsl/frag/separate-combined-fake-overload.sm30.frag` 及 reference |
| 反例一：dummy sampler | `shaders/asm/frag/image-fetch-no-sampler.asm.vk.invalid.frag`；`reference/opt/shaders/asm/frag/image-fetch-no-sampler.asm.vk.invalid.frag{,.vk}` |
| 反例二：运行时数组 nocompat | `shaders/vulkan/frag/nonuniform-qualifier.vk.nocompat.frag`；`test_shaders.py:757-767`；MSL/HLSL 对照 reference 见 6.3 |
| 测试框架 remap 约定 | `test_shaders.py:394-402`；nocompat 判定 `test_shaders.py:925-926` |
