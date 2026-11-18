# SPIRV-Cross 纹理采样语义边界分析：隐式导数 / 显式 LOD / 梯度采样

> 分析对象：本仓库（commit `57e4efc`）中的 GLSL、HLSL、MSL 三个后端。
> 所有"实测"输出均由本机构建的 `spirv-cross`（Release，`external` 测试关闭）对 glslang
> 生成的 SPIR-V 实际运行得到；构建与用例脚本不在仓库内，位于 `/tmp/lod-cases`（可复现）。
> 报告严格区分**当前行为**（代码与实测事实）与**建议**（作者意见），后者集中在第 8 节。

---

## 1. SPIR-V 层面的三条语义边界

SPIR-V 把"采样时细节级别（LOD）从哪里来"拆成三组操作码，SPIRV-Cross 在
`spirv_glsl.cpp:8472-8545` 的 `to_texture_op()` 中统一解码，再按后端分派：

| SPIR-V 操作码 | LOD 来源 | 是否依赖隐式导数 | 关键 ImageOperands |
|---|---|---|---|
| `OpImageSampleImplicitLod` (87) 及 Proj/Dref 变体 (89, 91, 93) | 实现根据坐标的屏幕空间偏导**隐式计算** | 是 | `Bias`、`MinLod`（不能带 `Lod`/`Grad`） |
| `OpImageSampleExplicitLod` (88) 及变体 (90, 92, 94) | 调用者显式给出 | 否 | `Lod` 或 `Grad` 二选一（SPIR-V 规范约束） |
| `OpImageSparseSample*` (305-307…) | 同上两类 | 随对应变体 | 同上，额外返回 residency code |

配套的显式偏导指令 `OpDPdx/OpDPdy/OpFwidth`（含 Coarse/Fine 变体，`spirv_cross.cpp:124-132`）
与 `OpImageQueryLod` 同属"隐式导数族"。SPIR-V 规范规定：隐式 LOD 与导数指令只在
**片段着色器**（或声明了相应 derivative group capability 的阶段）中良定义；
在非片元阶段，前端必须把它改写成显式 LOD（glslang 的实际做法是改写为 `Lod 0`，见第 6 节实测）。

三个后端共享同一套操作数解析循环（`spirv_glsl.cpp:8608-8616`）：

```cpp
test(bias, ImageOperandsBiasMask);
test(lod,  ImageOperandsLodMask);
test(grad_x, ImageOperandsGradMask);
test(grad_y, ImageOperandsGradMask);
...
test(minlod, ImageOperandsMinLodMask);
```

也就是说，**语义分流的输入只有 `bias/lod/grad_x/grad_y/minlod` 五个 ID 是否非零**，
后端差异全部发生在把这些 ID 翻译成目标语言函数名与限定符的阶段。

---

## 2. 三种采样方式在 GLSL / HLSL / MSL 中的表达对照

### 2.1 函数名与限定符映射（普通 2D 纹理，实测 `case18_functions.frag`）

同一条包含三种采样的辅助函数，在三个后端的输出：

| SPIR-V 形态 | GLSL（桌面 450） | HLSL（SM 5.0） | MSL（默认 macOS） |
|---|---|---|---|
| ImplicitLod | `texture(s, uv)` | `s.Sample(samp, uv)` | `s.sample(samp, uv)` |
| ExplicitLod + `Lod` | `textureLod(s, uv, l)` | `s.SampleLevel(samp, uv, l)` | `s.sample(samp, uv, level(l))` |
| ExplicitLod + `Grad` | `textureGrad(s, uv, dx, dy)` | `s.SampleGrad(samp, uv, dx, dy)` | `s.sample(samp, uv, gradient2d(dx, dy))` |
| ImplicitLod + `Bias` | `texture(s, uv, b)` | `s.SampleBias(samp, uv, b)` | `s.sample(samp, uv, bias(b))` |
| ImplicitLod + `MinLod` | `textureClampARB(...)` + `GL_ARB_sparse_texture_clamp` | **直接抛异常** | `min_lod_clamp(m)`（MSL 2.2+） |
| Proj 变体 | `textureProj*` 内建 | 改写为坐标显式除法 `uv.xy / uv.z`（`spirv_hlsl.cpp:4119-4120`） | 同样改写为除法（`spirv_msl.cpp:13010-13016`） |

代码位置：

- GLSL 函数名拼装：`spirv_glsl.cpp:8790-8817`（`texture` → 追加 `Grad`/`Lod`/`Offset`/`Clamp`/`ARB`）。
- HLSL 方法选择：`spirv_hlsl.cpp:4025-4032`，优先级链 `bias → grad → lod → 裸 Sample`；
  SM 2.x 遗留路径走 `tex2Dbias/tex2Dgrad/tex2Dlod`（`spirv_hlsl.cpp:4063-4070`），实测 ESSL/低版本输入时可见。
- MSL 限定符输出：`spirv_msl.cpp:13141-13273`，函数名本身只有 `sample/sample_compare/gather/read`
  （`spirv_msl.cpp:12751-12759`），LOD 语义全部通过尾部限定符 `level()/bias()/gradient2d()/gradient3d()/gradientcube()/min_lod_clamp()` 表达。

### 2.2 维度决定梯度限定符（MSL）

MSL 没有单一的 "grad" 关键字，梯度类型随纹理维度变化（`spirv_msl.cpp:13234-13262`）：

- `Dim1D/Dim2D` → `gradient2d`
- `Dim3D` → `gradient3d`
- `DimCube` → `gradientcube`；cube 数组模拟或开启 `agx_manual_cube_grad_fixup` 时改用
  注入的辅助函数 `spvGradientCube`（实测 `case17_cubegrad.frag`，参考
  `reference/shaders-msl/frag/sampler-cube-grad.agx-cube-grad.frag`）。

HLSL 与 GLSL 则不区分维度，统一是 `SampleGrad` / `textureGrad`。

### 2.3 遗留/低版本目标的降级表达

实测把同一 SPIR-V 定向到 ESSL 1.00（`--es --version 100`）时输出：

```glsl
#extension GL_EXT_shader_texture_lod : require
#extension GL_OES_standard_derivatives : require
r += texture2D(uSamp, vUV);
r += texture2DLodEXT(uSamp, vUV, 2.0);
r += texture2DGradEXT(uSamp, vUV, dFdx(vUV), dFdy(vUV));
```

对应 `legacy_tex_op()`（`spirv_glsl.cpp:7945` 起）：显式 LOD/梯度在遗留 GLSL 中需要
`GL_EXT_shader_texture_lod`，裸导数需要 `GL_OES_standard_derivatives`。
HLSL SM < 4.0 时退化为 `tex2D/tex2Dlod/tex2Dgrad` 自由函数（实测 `case1_branch` 默认 HLSL 输出）。

---

## 3. 非均匀控制流（non-uniform control flow）下的生成结果

### 3.1 控制依赖表达式的识别与失效机制（当前行为）

这是 SPIRV-Cross 处理"分支内导数"的核心机制，三个后端共用：

1. **标记**：`Compiler::block_is_control_dependent()`（`spirv_cross.cpp:106-196`）把包含
   隐式 LOD 采样、`OpImageQueryLod`、gather、所有 `OpDPdx*`/`OpDPdy*`/`OpFwidth*`、
   subgroup/quad 指令和控制屏障的块标记为 control-dependent。
2. **注册**：每个隐式 LOD 采样结果在发射时调用
   `register_control_dependent_expression(id)`（GLSL：`spirv_glsl.cpp:8434-8445`；
   HLSL：`spirv_hlsl.cpp:4259-4266`；MSL 经继承的 GLSL 路径，
   以及 swizzle 临时变量场景 `spirv_msl.cpp:13495-13506`；导数指令在
   `spirv_glsl.cpp:14811-14896`、`spirv_msl.cpp:9981-10000` 同样注册）。
3. **失效**：表达式被记入 `SPIRBlock::invalidate_expressions`（`spirv_common.hpp:1015`），
   在分支边 `branch()`（`spirv_glsl.cpp:18469-18472`）、布尔三元
   （`spirv_glsl.cpp:8196-8201`）、内存/控制屏障（`spirv_hlsl.cpp:6688-6693`、
   `spirv_msl.cpp:11453-11455`）等处经 `flush_control_dependent_expressions()`
   （`spirv_cross.cpp:594-600`）转入全局 `invalid_expressions`，强制该表达式
   **物化为具名临时变量而不是被内联前向（forward）到分支外**。

### 3.2 可观察的代码生成效果

**效果 A——内联表达式被提升/物化到分支外。** 仓库参考测试
`shaders-msl/frag/control-dependent-in-branch.desktop.frag` 中，`texture()` 与全部
`dFdx/dFdy/fwidth` 的结果在 `if` 之前声明：

```metal
float4 t = uSampler.sample(uSamplerSmplr, in.vInput.xy);   // 分支外
float4 d0 = dfdx(in.vInput); ...
if (in.vInput.y > 10.0) { out.FragColor += t; ... }        // 分支内只做累加
```

HLSL 参考（`reference/shaders-hlsl/frag/control-dependent-in-branch.desktop.frag`）
与 GLSL 参考行为一致；其中 `OpImageQueryLod` 还被拆成
`_56_tmp = ...CalculateLevelOfDetail(...)` 临时变量（`spirv_hlsl.cpp:4239-4250`）。

**效果 B——已经存入变量的结果不重复移动。** 实测 `case2_hoist.frag` / `case7_forward.frag`
（采样结果先 `OpStore` 到局部变量、分支汇合后再读取），三个后端都把采样语句保留在
`if` 块内。这是正确的：变量本身就是持久临时，控制依赖机制只阻止"把整条采样表达式
前向进分支外的另一个表达式"，不改变语句的 CFG 位置。

**效果 C——带 quad 保证的分支可以合法保留在分支内。** 参考
`shaders-msl/frag/quad-derivatives.msl21.frag`（`GL_EXT_shader_quad_control` 的
`quad_derivatives` 模式），两个隐式 LOD 采样原样留在 `quad_all/quad_any` 守护的分支里，
输出为普通 `tex1.sample(...)`，不做提升——因为 quad 内控制流一致，导数良定义。

**效果 D——函数调用传播控制依赖性。** `OpFunctionCall` 会查询
`function_is_control_dependent(callee)`（`spirv_glsl.cpp:13501, 13595-13596`），
若被调函数内含导数类操作，调用结果同样注册为控制依赖；`block_is_control_dependent`
递归检查被调函数（`spirv_cross.cpp:115-121`）。实测 `case6_phi.frag` 中
辅助函数内的 `texture()` 经此路径处理。

### 3.3 非均匀资源索引（与控制流正交但常同时出现）

`DecorationNonUniform` 的处理是另一条独立通路，不要与上面的控制依赖混淆：

- GLSL（Vulkan 语义）：数组化资源访问外包 `nonuniformEXT(...)`，并要求
  `GL_EXT_nonuniform_qualifier`（`spirv_glsl.cpp:8873-8877`、`619-624`）。
- HLSL：限定符字符串为 `NonUniformResourceIndex`（`spirv_hlsl.cpp:7221`），
  实测 `reference/shaders-hlsl/frag/nonuniform-qualifier.nonuniformresource.sm51.frag`：
  `uSamplers[NonUniformResourceIndex(i + 10)].Sample(...)`。
- MSL：`backend.nonuniform_qualifier = ""`（`spirv_msl.cpp:1706`），**限定符被静默丢弃**，
  实测同一着色器 MSL 输出只是 `uSamplers[_25].sample(...)`，没有任何标注。
  理由是 Metal 资源索引本身按非均匀语义处理，但这属于后端假设而非源语义显式保留。

---

## 4. 数组纹理上的三种采样

### 4.1 坐标与层的拆分

数组纹理的 SPIR-V 坐标把 layer 压在最后一个分量，各后端在参数发射阶段拆开：

- MSL（`spirv_msl.cpp:13050-13072`）：`texture2d_array.sample(samp, uv.xy, uint(rint(uv.z)), ...)`，
  layer 经 `rint()` 取整为 uint；cube 数组为 `sample(samp, xyz, uint(w))`
  （实测 `case8_array.frag`，参考 `texture-cube-array.frag`）。
- HLSL：数组纹理方法直接接受多一个分量的坐标（`Texture2DArray.Sample(samp, float3)`），
  层索引不拆出。
- GLSL：保持 `texture(sampler2DArray, vec3)` 形式。

梯度的维度**不含层分量**：2D 数组纹理的 `textureGrad`/`SampleGrad`/`gradient2d` 仍然只带
`vec2`/`float2` 梯度，层方向没有导数概念。实测 `case8_array` 三个后端均如此。

### 4.2 阴影数组纹理：LOD 没有直接表达，触发梯度/级别回退

这是三条语义边界上分歧最大的地方。GLSL 没有 `sampler2DArrayShadow` /
`samplerCubeShadow` 的 `textureLod` 内建，MSL 的 `sample_compare` 在 macOS 上对
`gradient*` 限定符支持有限，HLSL 比较采样器只有 `SampleCmpLevelZero` 这一个带 LOD 的方法。
各后端的当前策略：

**GLSL（`spirv_glsl.cpp:8768-8785`，`workaround_lod_array_shadow_as_grad`）：**

- Vulkan 语义或已启用 `GL_EXT_texture_shadow_lod`：直接发 `textureLod`，并
  `require_extension_internal("GL_EXT_texture_shadow_lod")`。
- 普通桌面 GLSL 且 Lod 操作数是常量 null（0）：把 `textureLod(..., 0)` 改写为
  `textureGrad(..., vec2(0), vec2(0))`。实测 `case8_array` 普通 GLSL 输出中
  `textureLod` 一行被替换成第二个 `textureGrad`。
- 普通 GLSL 且 Lod 非常量：只能要求 `GL_EXT_texture_shadow_lod`（无扩展则无法表达）。
- 仓库 asm 证据：`shaders/asm/frag/hlsl-sample-cmp-level-zero.asm.frag`
  （`OpImageSampleDrefExplicitLod Lod 0`）→
  `reference/shaders/asm/frag/hlsl-sample-cmp-level-zero.asm.frag` 输出
  `textureGrad(..., vec2(0.0), vec2(0.0))`。

**HLSL（`spirv_hlsl.cpp:3986-3990`）：** 任何带 `lod` 或 `grad` 的深度比较采样
统一发 `SampleCmpLevelZero`，注释原文："Assume we want a fixed level, and the only
thing we can get in HLSL is SampleCmpLevelZero." 实测 `case9_shadow.frag` 中
**非零** LOD（3.0）与**非零**梯度（0.7/0.9）两行都被压成
`SampleCmpLevelZero`，LOD/梯度信息静默丢失（见第 7 节可观察状态 #2）。

**MSL（`spirv_msl.cpp:13082-13218`）：**

- macOS + 编译期零梯度：提升为 `level(0)`（13108-13117）。实测默认 MSL 输出
  `sample_compare(..., level(0))`。
- iOS：零梯度也保留 `gradient2d(float2(0), float2(0))`（实测 `--msl-ios`）。
- macOS + 非零梯度且 MSL < 2.3：**抛异常**
  `"Using non-constant 0.0 gradient() qualifier for sample_compare. This is not supported on macOS prior to MSL 2.3."`
  （13118-13122）。实测默认版本对 `case19_msl_throw.frag` 直接 throw；
  `--msl-version 20300` 后正常输出 `gradient2d(...)`。
- 可选 `sample_dref_lod_array_as_grad`（`spirv_msl.hpp:500-507`）：把任意 LOD
  反算成梯度 `exp2(lod - 0.5) / extent`（13157-13214），用于规避部分 Metal 设备
  `level()` 偏置 bug；在旧版本上同样受 2.3 门槛约束（实测开启后默认版本 throw、
  2.3 输出反算公式，参考
  `depth-array-texture-lod.lod-as-grad.1d-as-2d.msl23.frag`）。

### 4.3 Cube 数组与模拟路径

开启 `emulate_cube_array` 时，cube 数组映射到 2D 数组，层经辅助函数
`spvCubemapTo2DArrayFace` 计算（`spirv_msl.cpp:13021-13034`），梯度限定符随之
降为 `gradient2d`；参考 `texture-cube-array.ios.emulate-cube-array.frag`。

---

## 5. 偏导不可用时的生成结果

"偏导不可用"有三种情形，当前行为各不相同：

### 5.1 阶段不支持隐式导数（vertex / compute 等）

SPIR-V 合法模块在这些阶段不会出现 `OpImageSampleImplicitLod`；glslang 前端把
GLSL 的 `texture()` 直接降级为 `textureLod(..., 0)`。实测：

- `case11_bad_vert.vert`（源码写 `texture()`）→ SPIR-V 反编译回 GLSL 是
  `textureLod(uSamp, vUV, 0.0)`；compute 版 `case13_bad_comp.comp` 同理，
  MSL 输出 `sample(..., level(0.0))`。
- 显式 LOD 与梯度在非片元阶段完全合法且原样保留：实测 `case10_vert.vert` →
  GLSL `textureLod/textureGrad`、HLSL SM50 `SampleLevel/SampleGrad`、
  MSL `sample(..., level(0.0))` 与 `sample(..., gradient2d(...))`。

### 5.2 目标语言/版本不支持某限定符（编译期报错而非静默错译）

- MSL `OpImageQueryLod` 要求 MSL 2.2+，否则 throw
  （`spirv_msl.cpp:10376-10377`）；1D 纹理直接不支持 QueryLod（10393-10395）。
- MSL `MinLod` → `min_lod_clamp` 要求 2.2+（`spirv_msl.cpp:13268-13269`），
  参考 `shaders-msl/asm/frag/min-lod.msl22.asm.frag`。
- HLSL 遇到 `MinLod` 操作数无条件 throw
  `"MinLod texture operand not supported in HLSL."`（`spirv_hlsl.cpp:3951-3952`），
  没有模拟实现。
- ESSL 上 sparse feedback 直接 throw（`spirv_glsl.cpp:8374-8375`）；
  MSL 整个 sparse 路径未实现（`spirv_msl.cpp:11288-11289`
  `"Sparse feedback not yet supported in MSL."`）。
- ESSL 不支持 Fine/Coarse 导数控制，throw `GL_ARB_derivative_control is unavailable in OpenGL ES`
  （`spirv_glsl.cpp:14827-14830` 等）。

### 5.3 硬件类型不支持限定符：1D 纹理的静默丢弃（MSL）

Metal 1D 纹理的 `sample()` 不接受 `level/bias/gradient` 限定符。MSL 后端用
`imgtype.image.dim != Dim1D || texture_1D_as_2D` 守卫全部三个发射点
（`spirv_msl.cpp:13143, 13150, 13229`，注释 "Metal does not support LOD for 1D textures."）。
实测 `case15_1d.frag` 原生 1D 输出四行**完全相同**的 `uS1.sample(samp, vU)`——
`textureLod`、`textureGrad`、bias 全部被静默丢弃；而加 `--msl-texture-1d-as-2d`
后限定符恢复（`level(2.0)`、`gradient2d(...)`、`bias(...)`）。
这是本主题下唯一"无报错、无警告的语义降级"。

### 5.4 隐式导数族的控制依赖兜底

即使目标语言支持导数，SPIRV-Cross 也不信任分支内导数的可移植性：第 3 节的失效机制
保证隐式 LOD/导数表达式不会被内联到汇合点之后。若源 SPIR-V 本身在非均匀分支里
做隐式采样且没有 quad 限定，翻译结果在语义上仍依赖目标编译器按 helper invocation
规则处理——SPIRV-Cross 不插入 `quad_broadcast` 之类的展平代码，只做表达式物化。

---

## 6. 三个后端的行为差异速查

| 情形 | GLSL | HLSL | MSL |
|---|---|---|---|
| 隐式 LOD | `texture` | `Sample` | `sample` |
| 显式 LOD | `textureLod` | `SampleLevel` | `level()` |
| 梯度 | `textureGrad` | `SampleGrad` | `gradient2d/3d/cube` |
| Bias | `texture(...,b)` | `SampleBias` | `bias()` |
| MinLod | `textureClampARB` + 扩展 | **throw** | `min_lod_clamp()`（2.2+） |
| 阴影数组 LOD 0（无扩展） | 改写零梯度 `textureGrad` | `SampleCmpLevelZero` | macOS→`level(0)`；iOS→零 `gradient2d` |
| 阴影数组**非零** LOD/梯度 | 要求 `GL_EXT_texture_shadow_lod` | **静默压成 LevelZero** | 2.3+ 保留；旧 macOS **throw** |
| LOD→梯度反算 | 不支持（仅零常量） | 不支持 | 选项 `sample_dref_lod_array_as_grad` |
| 1D 的 LOD/grad/bias | 正常内建 | 正常方法 | 原生 1D **静默丢弃**；1D-as-2D 保留 |
| Sparse 反馈 | `sparseTexture*ARB` + 扩展，ESSL throw | 支持 | 整体未实现，throw |
| NonUniform 资源索引 | `nonuniformEXT()` | `NonUniformResourceIndex()` | 静默省略 |
| Proj | 专用内建 | 显式除法 | 显式除法 |
| 分支内隐式采样 | 表达式物化/提升 | 同左 | 同左；quad_derivatives 下可留分支内 |

---

## 7. 两个可观察的调用 / 资源状态

以下两项均可由命令行直接复现，作为跨后端语义边界的端到端证据。

### 可观察状态 #1：资源声明状态——一个 combined sampler 拆成独立 texture + sampler，且阴影类型经 reinterpret 转换

输入 `case8_array.spv`（GLSL  combined `sampler2DArray` / `samplerCubeArray` /
`sampler2DArrayShadow`），MSL 反射出的函数签名（实测）：

```metal
fragment main0_out main0(...,
    texture2d_array<float> uArr       [[texture(0)]],
    texturecube_array<float> uCubeArr [[texture(1)]],
    texture2d_array<float> uArrShadow [[texture(2)]],
    sampler uArrSmplr       [[sampler(0)]],
    sampler uCubeArrSmplr   [[sampler(1)]],
    sampler uArrShadowSmplr [[sampler(2)]])
```

HLSL 对应为 `Texture2DArray : register(tN)` + 独立 `SamplerState/SamplerComparisonState : register(sN)`
（实测，代码 `spirv_hlsl.cpp:4082-4090`）。两点可观察事实：

1. **资源数量翻倍**：3 个 combined 采样器在 MSL/HLSL 中变成 6 个绑定槽；
   `--reflect` 的 JSON 仍按源类型报告 `sampler2DArrayShadow` 等（实测反射输出），
   即反射视图与目标代码视图不同。
2. **阴影类型不靠声明而靠调用点转换**：MSL 里 `uArrShadow` 声明仍是普通
   `texture2d_array<float>`，每次比较采样调用才套 `spvDepthCast(uArrShadow)`
   （`spirv_msl.cpp:12742-12746`，模板定义见各参考输出头部）。
   这使得同一资源可同时用于 dref 与非 dref 调用，是 MSL 类型系统下的刻意设计。

### 可观察状态 #2：同一对 ExplicitLod 操作数，三个后端生成不等价的调用

输入 `case9_shadow.frag`（`sampler2DArrayShadow` 上分别以 LOD=3.0 和
非零梯度 0.7/0.9 采样），实测输出：

- GLSL：`textureLod(..., 3.0)` 与 `textureGrad(..., vec2(0.7), vec2(0.9))` —— 两个操作数完整保留；
- HLSL SM50：两行都是 `uArrShadow.SampleCmpLevelZero(samp, xyz, w)` —— LOD 与梯度被丢弃；
- MSL 2.3：`sample_compare(..., level(3.0))` 与 `sample_compare(..., gradient2d(0.7, 0.9))` —— 完整保留；
  MSL 默认（<2.3）macOS：编译期 throw，不产生输出。

即同一份 SPIR-V 在 GLSL/MSL 上保留三级 LOD 语义，在 HLSL 上恒为 0 级比较。
这是第 4.2 节代码路径的直接外部表现，也是移植 HLSL 后端时最需要警惕的可观察差异。

---

## 8. 当前行为与建议（意见部分，与上述事实分离）

### 8.1 当前行为总结（事实）

1. 三后端共用操作数解码与控制依赖失效框架，后端差异集中在函数名/限定符翻译层。
2. 控制流处理策略是"物化表达式、不重排 CFG、不展平分支"；quad 限定场景例外。
3. 语义缺口有两种失败模式：**显式 throw**（MinLod-on-HLSL、旧 MSL 的 compare 梯度、
   sparse-on-MSL、QueryLod 版本门槛）与**静默降级**（HLSL 阴影 LOD→LevelZero、
   MSL 原生 1D 丢限定符、MSL 丢 NonUniform 限定符、GLSL 零 LOD→零梯度改写）。
4. 平台差异通过编译选项暴露（`sample_dref_lod_array_as_grad`、
   `agx_manual_cube_grad_fixup`、`texture_1d_as_2d`、MSL 版本号、iOS/macOS 开关），
   默认值偏保守。

### 8.2 建议

1. **为静默降级增加诊断通道（建议，非现状）。** HLSL 的 `SampleCmpLevelZero`
   压平（`spirv_hlsl.cpp:3986-3990`）与 MSL 1D 限定符丢弃（13143/13150/13229）
   目前无任何 warning；建议至少在编译器选项中提供严格模式，对这两类输出
   `SPIRV_CROSS_THROW` 或可查询的 warning 列表，避免渲染结果静默变化。
2. **HLSL 阴影 LOD 的文档化限制应上移到接口文档。** 代码注释只解释了
   LevelZero 的来由，未说明非零 LOD/梯度会失真；建议在 `spirv_hlsl.hpp`
   选项注释与 README 移植指南中明确该限制，并建议上游在需要正确 LOD 时
   改用手动梯度计算或着色器内分支。
3. **控制依赖机制可考虑输出期提示。** 当前机制正确但不可见；对调试
   "为什么采样被提前"的用户，建议在 debug dump（如 `--dump-*` 类设施）
   中暴露 `invalidate_expressions` 集合，而不是只靠阅读源码
   （`spirv_glsl.cpp:12600-12607`）。
4. **MSL 1D 用户应默认评估 `texture_1D_as_2D`。** 鉴于原生 1D 路径会丢
   LOD/grad/bias，建议在项目规范中对任何使用显式 LOD 的 1D 采样强制开启
   该选项；这是配置建议，不需要改 SPIRV-Cross 代码。
5. **NonUniform 在 MSL 的省略应在反射层保留痕迹。** 建议反射 JSON 对
   带 `DecorationNonUniform` 的资源加标记，便于下游验证 Metal 侧假设，
   而不是在代码生成时无痕丢弃（现状 `spirv_msl.cpp:1706`）。
6. **测试补强。** 仓库已有 `control-dependent-in-branch`、`quad-derivatives`、
   `sampler-compare-cascade-gradient`（含 msl23/ios 变体）、
   `depth-array-texture-lod.lod-as-grad`、`min-lod.msl22`、
   `sampler-cube-grad.agx-cube-grad` 等参考材料，但缺少：
   (a) HLSL 非零阴影 LOD 被压平的专门参考测试；(b) 原生 1D 限定符丢弃的
   负面参考（现有 `sampler-1d-lod.frag` 只覆盖 1D-as-2D 与 fetch 路径）。
   建议补这两类锁行为测试，防止未来重构改变静默降级策略而无感知。

---

## 9. 源码与测试材料索引

### 9.1 关键源码位置

| 主题 | 文件:行 |
|---|---|
| 采样操作数统一解码（bias/lod/grad/minlod） | `spirv_glsl.cpp:8593-8616` |
| GLSL 函数名拼装（Grad/Lod/Clamp/sparse/ARB） | `spirv_glsl.cpp:8755-8818` |
| GLSL 阴影数组 LOD→零梯度 workaround | `spirv_glsl.cpp:8768-8785`，参数侧 `8915-8923` |
| GLSL 隐式 LOD 注册控制依赖 | `spirv_glsl.cpp:8434-8445` |
| GLSL 导数指令与控制依赖 | `spirv_glsl.cpp:14811-14896` |
| GLSL 遗留（ESSL）纹理内建与扩展 | `spirv_glsl.cpp:7945` 起 `legacy_tex_op` |
| GLSL sparse 扩展与 ESSL 拒绝 | `spirv_glsl.cpp:8370-8390`，`15149-15168` |
| HLSL 方法选择（Sample/SampleBias/Grad/Level/Cmp*） | `spirv_hlsl.cpp:3976-4033` |
| HLSL 阴影 LOD/grad → SampleCmpLevelZero | `spirv_hlsl.cpp:3986-3992` |
| HLSL MinLod 拒绝 | `spirv_hlsl.cpp:3951-3952` |
| HLSL QueryLod 临时变量合成 | `spirv_hlsl.cpp:4239-4250` |
| HLSL 非均匀限定符 | `spirv_hlsl.cpp:7221` |
| MSL sample/sample_compare/gather/read 命名 | `spirv_msl.cpp:12735-12762` |
| MSL level/bias/gradient/min_lod_clamp 限定符 | `spirv_msl.cpp:13141-13273` |
| MSL compare 梯度 macOS/iOS/2.3 策略与 throw | `spirv_msl.cpp:13102-13138` |
| MSL LOD→梯度反算 workaround | `spirv_msl.cpp:13157-13214`；选项 `spirv_msl.hpp:500-507` |
| MSL 1D 限定符守卫 | `spirv_msl.cpp:13143, 13150, 13229` |
| MSL QueryLod（2.2+，calculate_*_lod） | `spirv_msl.cpp:10374-10419` |
| MSL sparse 未实现 | `spirv_msl.cpp:11288-11289` |
| MSL AGX cube 梯度修正 | `spirv_msl.cpp:13194-13198, 13248-13253`；选项 `spirv_msl.hpp:526` |
| 控制依赖块分析（操作码清单） | `spirv_cross.cpp:106-196` |
| 控制依赖刷新 | `spirv_cross.cpp:594-600`；`spirv_glsl.cpp:18469-18472` |
| 表达式注册/失效数据结构 | `spirv_glsl.cpp:12600-12607`；`spirv_common.hpp:1015` |

### 9.2 已有测试材料（仓库内）

| 测试源（shaders*/） | 参考输出（reference/） | 覆盖点 |
|---|---|---|
| `shaders-msl/frag/control-dependent-in-branch.desktop.frag` | 同名 | 分支内隐式采样/导数物化为分支外临时 |
| `shaders-msl/frag/quad-derivatives.msl21.frag` | 同名 | quad_derivatives 下采样留分支内 |
| `shaders-{msl,hlsl}/frag/control-dependent-select.spv16.frag` | 同名 | 三元选择与控制依赖 |
| `shaders/asm/frag/hlsl-sample-cmp-level-zero.asm.frag` | `reference/shaders/asm/frag/` 同名 | Dref ExplicitLod 0 → GLSL 零梯度 |
| `shaders-msl/frag/sampler-compare-cascade-gradient.{,msl23,ios}.frag` | 同名三份 | macOS level(0) / 2.3 梯度 / iOS 差异 |
| `shaders-msl/frag/depth-array-texture-lod.lod-as-grad.1d-as-2d.msl23.frag` | 同名 | LOD→exp2/extent 梯度反算 |
| `shaders-msl/frag/sampler-cube-grad.agx-cube-grad.frag` | 同名 | spvGradientCube 修正 |
| `shaders-msl/asm/frag/min-lod.msl22.asm.frag` | 同名 | MinLod → min_lod_clamp |
| `shaders-msl/asm/frag/implicit-read-dep-phi.asm.frag` | 同名 | 携带采样结果的 Phi 物化为临时 |
| `shaders/frag/texture-shadow-lod.vk.frag`、`texture-shadow-lod-bias.frag` | 同名 | GL_EXT_texture_shadow_lod 路径 |
| `shaders-hlsl/frag/tex-sampling.frag` | 同名 | HLSL 全采样方法对照表 |
| `shaders-hlsl/frag/sample-cmp-level-zero.frag`、`query-lod.desktop.frag` | 同名 | HLSL 比较采样与 CalculateLevelOfDetail |
| `shaders-msl/frag/nonuniform-qualifier.msl2.frag` | 同名；HLSL 版 `*.sm51.frag` | NonUniform 三后端差异 |
| `shaders-msl/frag/sampler-1d-lod{,.1d-as-2d}.frag` | 同名 | 1D / 1D-as-2D 采样 |
| `shaders-msl/frag/texture-cube-array{,.ios.emulate-cube-array}.frag` | 同名 | cube 数组与模拟 |
| `shaders-msl-no-opt/frag/explicit-lod-unpack-arguments.frag` | 同名 | 未优化 IR 下 level() 参数解包 |

测试驱动为 `test_shaders.py` / `test_shaders.sh`；文件名中的多后缀（`.msl23`、`.ios`、
`.agx-cube-grad`、`.lod-as-grad`、`.1d-as-2d` 等）编码该用例专用的后端选项，
后缀解析逻辑在 `test_shaders.py` 内。

### 9.3 报告实测用例（仓库外，可复现）

全部位于 `/tmp/lod-cases/`，glslang（自建，`/tmp/spv-tools/glslang`）编译
SPIR-V，spirv-cross（自建，`/tmp/spvc-build`）翻译：
`case1_branch`（分支三态）、`case2/3/4/5/6/7`（控制依赖与提升）、
`case8_array`（数组纹理/资源状态 #1）、`case9_shadow`（调用状态 #2）、
`case10-13`（非片元阶段）、`case14`（bias）、`case15_1d`（1D 静默丢弃）、
`case16_proj`（投影）、`case17_cubegrad`（cube 梯度）、`case18_functions`
（函数内三态）、`case19_msl_throw`（macOS 版本门槛 throw）。
