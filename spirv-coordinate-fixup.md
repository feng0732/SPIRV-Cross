# SPIRV-Cross 跨图形 API 坐标修正（Coordinate Fixup）语义对比报告

> 分析对象：本仓库 SPIRV-Cross（GLSL / HLSL / MSL 三个后端）。
> 关注问题：裁剪空间（clip space）深度范围、Y 轴方向、片元坐标（FragCoord）以及手写深度输出在跨 API 翻译时，**修正责任落在谁身上、在入口点的什么位置生效、多阶段组合时在哪里失效**。
> 所有结论均标注源码位置（`文件:行号`），并在文末用两个可复现场景（含仓库自带参考输出）支撑。

---

## 1. 背景：同一份 SPIR-V，三套坐标约定

SPIR-V 按 Vulkan 约定描述坐标，但目标 API 的约定不同：

| 约定 | Vulkan / SPIR-V | OpenGL / GLSL | Direct3D / HLSL | Metal / MSL |
|---|---|---|---|---|
| 裁剪空间 Z（NDC 深度） | `[0, w]` | `[-w, w]` | `[0, w]` | `[0, w]` |
| 窗口/帧缓冲原点 | 左上 | 左下 | 左上 | 左上 |
| 片元坐标语义 | `FragCoord`，左上角原点 | `gl_FragCoord`，左下角原点 | `SV_Position`，左上角原点 | `[[position]]`，左上角原点 |
| `FragCoord.w` | `1/clip.w` | `1/clip.w` | D3D 中为 `clip.w`（需倒数还原） | 与 SPIR-V 一致 |

SPIRV-Cross 的策略不是"自动按后端纠正一切"，而是：

1. **入口点选项**（`CompilerGLSL::Options.vertex`，三个后端共用同一结构）只暴露两个显式开关，默认都为 `false`：
   - `fixup_clipspace`：裁剪空间 Z 范围换算；
   - `flip_vert_y`：翻转顶点输出的 Y。
   见 `spirv_glsl.hpp:173-189`，其中注释明确写出同一开关在不同后端方向相反：
   > GLSL: rewrite [0, w] depth (Vulkan/D3D style) to [-w, w] … MSL/HLSL: rewrite [-w, w] … to [0, w]。
2. **片元侧没有对应的 `flip_frag_y` 开关**（全仓库 grep 不到任何 fragment Y-flip 选项；`FragmentOptions` 只有精度设置，`spirv_glsl.hpp:192-198`）。片元坐标差异通过内建变量的隐式映射、HLSL 的自动倒数/半像素修正、MSL 的模拟选项分别处理。
3. C API 把这两个开关列为**通用选项**（任何后端都可设置），名字却带误导性：`SPVC_COMPILER_OPTION_FIXUP_DEPTH_CONVENTION` 与 `SPVC_COMPILER_OPTION_FLIP_VERTEX_Y`（`spirv_cross_c.h:628-629`，实现在 `spirv_cross_c.cpp:430-435`）。
4. CLI 入口 `--fixup-clipspace` / `--flip-vert-y`（`main.cpp:1715-1716`）经 `main.cpp:1469-1470` 写入同一份 options，再由 `set_common_options` 下发给实际编译器——HLSL/MSL 类都继承自 `CompilerGLSL`（`spirv_hlsl.hpp:107`、`spirv_msl.hpp:293`），所以一个布尔值会在不同 `emit_fixup()` 覆写里产生方向相反的代码。

README 也明确建议优先改投影矩阵而不是依赖这些开关（`README.md:437-449`，"Clip-space conventions"一节）。

---

## 2. 三个后端的 `emit_fixup()`：同一开关，三种生成语义

修正代码由入口点 Return 块统一触发。基类 `CompilerGLSL` 在 `spirv_glsl.cpp:19582-19589` 规定顺序：先跑 `fixup_hooks_out`（接口块拷贝等），再调用虚函数 `emit_fixup()`；HLSL 与 MSL 各自 `override`（`spirv_hlsl.hpp:261`、`spirv_msl.hpp:946`）。

### 2.1 GLSL 后端（`spirv_glsl.cpp:18332-18344`）

```cpp
void CompilerGLSL::emit_fixup()
{
    if (is_vertex_like_shader())
    {
        if (options.vertex.fixup_clipspace)
            statement("gl_Position.z = 2.0", suffix, " * gl_Position.z - gl_Position.w;");
        if (options.vertex.flip_vert_y)
            statement("gl_Position.y = -gl_Position.y;");
    }
}
```

- 方向：`z' = 2z - w`，把 Vulkan 的 `[0,w]` 映射到 GL 的 `[-w,w]`。
- 守卫**只检查** `is_vertex_like_shader()`（`spirv_cross.cpp:2635-2640`：Vertex / Geometry / TessControl / TessEval 四类），**不检查 Position 是否真的活跃**。
- 字面量后缀按 `backend.float_literal_suffix` 决定（GLSL 默认 false，输出裸 `2.0`）。
- 实测（本仓库预编译 `./spirv-cross`，对 glslang 编译的 `gl_Position.z *= 0.5` 顶点着色器）：

```glsl
// --fixup-clipspace --flip-vert-y
    gl_Position.z *= 0.5;
    gl_Position.z = 2.0 * gl_Position.z - gl_Position.w;
    gl_Position.y = -gl_Position.y;
```

### 2.2 HLSL 后端（`spirv_hlsl.cpp:3780-3797`）

```cpp
void CompilerHLSL::emit_fixup()
{
    if (is_vertex_like_shader() && active_output_builtins.get(BuiltInPosition))
    {
        if (hlsl_options.shader_model <= 30) { /* gl_HalfPixel 半像素修正 */ }
        if (options.vertex.flip_vert_y)
            statement("gl_Position.y = -gl_Position.y;");
        if (options.vertex.fixup_clipspace)
            statement("gl_Position.z = (gl_Position.z + gl_Position.w) * 0.5;");
    }
}
```

- 方向：`z' = (z + w) * 0.5`，把 GL 风格 `[-w,w]` 映射到 D3D 风格 `[0,w]`——与 GLSL 后端**互为逆变换**。
- 守卫多了 `active_output_builtins.get(BuiltInPosition)`：着色器完全不写 Position 时，整段修正不生成（实测见 §5.2）。
- 额外责任：SM30（默认 shader model 就是 30，`spirv_hlsl.hpp:112`）自动注入 `uniform float4 gl_HalfPixel;`（`spirv_hlsl.cpp:1738-1742`）并做 texel 偏移；SM50+ 不生成。
- HLSL 设置 `backend.float_literal_suffix = true`（`spirv_hlsl.cpp:7201`），所以输出 `0.5f` 风格。
- 实测同一 SPIR-V（SM50）：

```hlsl
    gl_Position.z *= 0.5f;
    gl_Position.y = -gl_Position.y;
    gl_Position.z = (gl_Position.z + gl_Position.w) * 0.5;
```

注意两个后端连**语句顺序都不同**：GLSL 先 Z 后 Y，HLSL 先 Y 后 Z。对当前各分量独立的变换没有数值差异，但属于可观察的生成语义差别。

### 2.3 MSL 后端（`spirv_msl.cpp:13883-13937`）

```cpp
void CompilerMSL::emit_fixup()
{
    if (stage_out_var_id && !capture_output_to_buffer)
    {
        ...
        if (is_vertex_like_shader() && !qual_pos_var_name.empty())
        {
            if (options.vertex.fixup_clipspace)
                statement(qual_pos_var_name, ".z = (", ..., ".z + ", ..., ".w) * 0.5;  // Adjust clip-space for Metal");
            if (msl_options.emulate_depth_clip_enable)   { /* 视口 Z 变换，查 spvDepthClipState 缓冲 */ }
            if (msl_options.emulate_reversed_depth_viewport) { /* z = w - z，按视口掩码条件执行 */ }
            if (options.vertex.flip_vert_y)
                statement(qual_pos_var_name, ".y = -(", ..., ".y);  // Invert Y-axis for Metal");
        }
        else if (get_execution_model() == ExecutionModelFragment && !qual_frag_depth_var_name.empty() &&
                 msl_options.emulate_depth_clip_enable)
        { /* 对 gl_FragDepth 做 clamp */ }
    }
}
```

- Z 方向与 HLSL 相同（`(z+w)*0.5`），但操作的是返回结构体成员（`out.gl_Position`，名字由 `qual_pos_var_name` 记录，赋值点在 `spirv_msl.cpp:2993-2994` 与 `3645-3646`）。
- 独有两层 MSL 专属深度模拟，且**执行顺序固定**为：clipspace 换算 → 视口 Z 缩放/平移 → 反向深度 `z = w - z` → Y 翻转。
- 最外层守卫 `stage_out_var_id && !capture_output_to_buffer` 是关键边界（见 §5）。
- 片元分支只在**手写了 `gl_FragDepth`**（`qual_frag_depth_var_name` 非空）且开启 `emulate_depth_clip_enable` 时生效，在用户写入之后追加 `clamp`。
- `float_literal_suffix = false`（`spirv_msl.cpp:1684`），输出无 `f` 后缀。

### 2.4 片元坐标：没有开关，责任由内建映射承担

| 后端 | FragCoord 翻译 | 自动修正 |
|---|---|---|
| GLSL | 直接保留 `gl_FragCoord`（`spirv_glsl.cpp:10646-10647`） | 无；Y 方向差异留给应用（负视口高度或投影矩阵），编译器不提供片元 Y 翻转 |
| HLSL | `gl_FragCoord : SV_Position`（`spirv_hlsl.cpp:778`） | ① 现代 SM：入口包装里 `gl_FragCoord.w = 1.0 / gl_FragCoord.w;`（`spirv_hlsl.cpp:3436-3437`），把 D3D 的 `clip.w` 还原成 SPIR-V 约定的 `1/clip.w`；② SM30 legacy：改为 `+ float4(0.5, 0.5, 0, 0)` 半像素偏移（`spirv_hlsl.cpp:3430-3433`） |
| MSL | `float4 gl_FragCoord [[position]]`（映射表 `spirv_msl.cpp:18755`） | 无 Y 翻转；采样率着色时在 fixup-in 钩子补 `get_sample_position` 偏移（`spirv_msl.cpp:15953-15960`）；`interpolateAtOffset` 自动加 `0.4375` 像素中心补偿（`spirv_msl.cpp:12265-12271`） |

HLSL 的 `w` 倒数有现成参考测试佐证：`reference/shaders-hlsl/frag/texel-fetch-offset.frag:25-26` 与 `input-attachment.frag:29-30` 均含 `gl_FragCoord.w = 1.0 / gl_FragCoord.w;`。MSL 的 0.4375 补偿见 `reference/shaders-msl/asm/frag/pull-model-interpolation.asm.msl23.frag` 等三个参考文件。

---

## 3. 入口点选项的生效位置：fixup 钩子机制

三个后端共用同一套钩子基础设施（定义在基类入口函数对象上）：

- `fixup_hooks_in`：在入口函数体**之前**执行（`spirv_glsl.cpp:18314-18315`，先于 `emit_block_chain`）。用于 stage-in 接口块装载、MSL 的 TessCoord 翻转等输入侧修正。
- `fixup_hooks_out`：在每个 Return 块、`emit_fixup()` **之前**执行（`spirv_glsl.cpp:19584-19588`）。用于把局部变量拷回 stage-out 结构体。
- `emit_fixup()`：Return 块内最后一步，因此所有坐标修正都位于**用户代码之后**。

两个重要推论：

1. **提前 return 也安全**：SPIR-V 的每个 `OpReturn` 都编译成一个 Return 块，修正语句会在每条返回路径上各插一份。实测含 `if (u > 0.5) return;` 的顶点着色器，GLSL 与 MSL 都在 if 内和函数尾部分别生成了一份 fixup。
2. **几何/细分着色器里修正只出现一次且在函数末尾**，不会跟着 `EmitVertex()` 走。实测几何着色器（循环内三次 `EmitVertex`）生成的 HLSL 为：

```hlsl
void geom_main(...)
{
    for (int i = 0; i < 3; i++) {
        gl_Position = gl_in[i].gl_Position;
        geometry_stream.Append(stage_output);   // 图元已发出
    }
    gl_Position.z = (gl_Position.z + gl_Position.w) * 0.5;  // 此时翻转/换算影响不到已 Append 的顶点
}
```

即：对几何着色器（以及 TESC 的逐控制点输出），尾部 fixup 改写的只是函数结束时残留的 `gl_Position` 变量，**真正流向下一阶段的顶点值不受影响**。这是多阶段接口上最容易踩的语义陷阱（详见 §5.3）。

---

## 4. 手写深度输出与坐标修正同时出现时的结果

用户手写 `gl_Position.z` / `gl_FragDepth` 并不会抑制或融合修正——修正一律是**用户语句之后追加的二次改写**，因此最终值是"用户表达式再套一层后端变换"。

### 4.1 顶点侧：手写 `gl_Position` + `fixup_clipspace`

以仓库测试着色器 `shaders-msl-no-opt/vert/depth-clip-enable.emulate-depth-clip-enable.fixup-clipspace.vert` 为例（全文只有 `gl_Position = vec4(0.0, 0.0, 0.25, 1.0);`），MSL 输出（参考文件同名）：

```metal
    out.gl_Position = float4(0.0, 0.0, 0.25, 1.0);
    out.gl_Position.z = (out.gl_Position.z + out.gl_Position.w) * 0.5;       // Adjust clip-space for Metal
    if (spvDepthClipState.emulateViewportZ != 0u)
    {
        float2 spvViewportDepthRange = spvDepthClipState.viewportDepthRanges[0];
        out.gl_Position.z = out.gl_Position.z * (spvViewportDepthRange.y - spvViewportDepthRange.x)
                          + out.gl_Position.w * spvViewportDepthRange.x;    // Emulate viewport Z transform
    }
```

手写的 `z = 0.25` 先保留，再被三层变换依次改写。若同时开 `--msl-emulate-reversed-depth-viewport`，参考文件 `...emulate-reversed-depth-viewport.fixup-clipspace.vert` 显示最后还会追加条件反转：

```metal
    if ((spvEmulatedReversedDepthViewportMask & 1u) != 0u)
        out.gl_Position.z = out.gl_Position.w - out.gl_Position.z;    // Emulate reversed-depth viewport
```

### 4.2 片元侧：手写 `gl_FragDepth` + 深度裁剪模拟

`shaders-msl-no-opt/frag/depth-clip-enable.emulate-depth-clip-enable.msl2.frag` 手写 `gl_FragDepth = 1.25;`，参考输出：

```metal
    out.gl_FragDepth = 1.25;
    if (spvDepthClipState.emulateDepthClamp != 0u)
    {
        float2 spvViewportDepthRange = spvDepthClipState.viewportDepthRanges[spvDepthClipViewportIndex];
        out.gl_FragDepth = clamp(out.gl_FragDepth, spvViewportDepthRange.x, spvViewportDepthRange.y);
    }
```

要点：

- 修正读的是**用户写完之后的 `out.gl_FragDepth`**，所以 clamp 边界对用户值生效；
- 该缓冲参数只在确实需要时才出现：`needs_depth_clip_state_buffer()`（`spirv_msl.hpp:642-649`）要求 `emulate_depth_clip_enable` 且有 `stage_out_var_id` 且非 capture，并且片元阶段必须存在手写 FragDepth。实测不带 FragDepth 的片元着色器即使开了选项也不生成缓冲和 clamp；
- 顶点侧与片元侧的模拟状态共用同一个 `spvDepthClipState` 缓冲（固定 slot 17，`spirv_msl.hpp:323`），反向深度掩码固定 slot 18（`spirv_msl.hpp:322`），这是应用侧必须配套填充的 ABI 边界。

### 4.3 同一份手写 SPIR-V 在三后端的数值含义不同

同一个 `gl_Position.z = 0.25, w = 1.0`：

| 后端/选项 | 生成代码 | 最终 z |
|---|---|---|
| GLSL `--fixup-clipspace` | `z = 2.0*z - w` | `-0.5` |
| HLSL `--fixup-clipspace` | `z = (z+w)*0.5` | `0.625` |
| MSL `--fixup-clipspace` | `z = (z+w)*0.5` | `0.625` |
| 任意后端，不开开关 | 原样 `0.25` | `0.25` |

也就是说选项名称相同、输入相同，输出数值按后端解释方向不同；不开开关时三个后端都原样透传，由应用自己保证投影矩阵与目标 API 匹配。

---

## 5. 多阶段接口组合时暴露的边界

### 5.1 MSL 的 capture-output 路径：整段 `emit_fixup()` 失效

`emit_fixup()` 的最外层条件是 `stage_out_var_id && !capture_output_to_buffer`（`spirv_msl.cpp:13885`）。两种情况会强制 `capture_output_to_buffer = true`（`spirv_msl.cpp:1899-1905`）：

1. 显式 `--msl-capture-output`（顶点输出改为写 buffer，常用于 GPU-driven 管线）；
2. **Tessellation Control 着色器，或 `--msl-vertex-for-tessellation` 的顶点着色器**——它们在 Metal 里以 compute kernel 形式运行。

实测对 TESC 与 `--msl-vertex-for-tessellation` 的 VS 同时给 `--fixup-clipspace --flip-vert-y`，输出中 grep 不到任何 `Adjust clip-space` / `Invert Y-axis` 语句；而 GLSL 后端对同一份 TESC 仍照常生成 `gl_Position.z = 2.0 * ...` 与 `gl_Position.y = -...`。

含义：**fixup 的责任在这些路径上被悄悄移交给应用/驱动**（MoltenVK 在组装真正的渲染管线时处理），编译器不再兜底。同样，反向深度缓冲参数的注入条件也带 `!capture_output_to_buffer`（`spirv_msl.cpp:15226-15228`）。

### 5.2 "Position 未活跃"在三后端表现不一致

用一个只写 `location=0` 颜色、完全不写 `gl_Position` 的顶点着色器，开两个 fixup 开关：

- **GLSL**：仍生成 `gl_Position.z = 2.0*...` 和 `gl_Position.y = -...`（守卫不看活跃集合，`spirv_glsl.cpp:18334`）。引用了未赋值的内建变量，依赖 GLSL 语义把它当全局变量处理。
- **HLSL**：整段不生成（`active_output_builtins.get(BuiltInPosition)` 为 false，`spirv_hlsl.cpp:3782`），输出结构体里也没有 SV_Position。
- **MSL**：反过来会**隐式补一个 Position 成员**。`spirv_msl.cpp:1101-1173` 的逻辑：Vertex/TESE 阶段只要还有别的 stage output、又没 Position，就合成一个 `[[position]]` 变量（参考 `shaders-msl/vert/implicit-position-1.vert` 及其参考输出），于是 `qual_pos_var_name` 非空，fixup 照样对这个零初始化的成员执行。

`--mask-stage-output-builtin Position` 进一步放大差异（实测）：

- MSL：Position 被掩码后从输出结构体消失，`qual_pos_var_name` 为空 → fixup 静默消失，只剩函数内对局部 `gl_Position` 的无效写入；
- GLSL：掩码不影响 fixup 生成；
- HLSL：输出结构体仍保留 `SV_Position` 且 fixup 照常（掩码未清空其 active 标记路径与 MSL 不同）。

### 5.3 Mesh 着色器：两套完全不同的翻转机制，且 Z 修正被丢弃

`is_vertex_like_shader()`（`spirv_cross.cpp:2635-2640`）**不包含 `ExecutionModelMeshEXT`**，所以三个后端的 `emit_fixup()` 对 mesh 着色器整体不生效。Y 翻转由各后端单独实现，Z 的 clipspace 换算则没有任何替代：

- **HLSL**：编译期注入 `spvFlipVertY(float4)` / `spvFlipVertY(float)` 两个辅助函数（`spirv_hlsl.cpp:2413-2424`），并在**每条写 Position 的 store 指令处**包一层（`spirv_hlsl.cpp:5460-5471`，靠 `SPIRExpression::access_meshlet_position_y` 标记，标记在访问链构造时打上，`spirv_glsl.cpp:11305-11310` 与 `11365-11370`）。仓库参考 `reference/shaders-hlsl-no-opt/mesh/mesh-shader-basic-triangle.spv14.vk.nocompat.nofxc.flip-vert-y.mesh` 显示整向量写、`.y` 分量写、动态下标写分别被包装成 `spvFlipVertY(...)` 或 `-x`；而 `--fixup-clipspace` 在同一 mesh 输出中**没有任何 Z 语句**。
- **MSL**：在 `emit_mesh_outputs()`（经 `fixup_hooks_out` 注册，`spirv_msl.cpp:15855`；实现 `21146` 起）逐顶点 `set_vertex` 之前插 `spvV.gl_Position.y = -(spvV.gl_Position.y);`（`spirv_msl.cpp:21244-21247`）。实测同样确认 `--fixup-clipspace` 不产生任何 Z 修正。
- **GLSL**：mesh/task 模型强制要求 Vulkan 语义，否则直接抛异常 `"Mesh shaders require Vulkan semantics."`（`spirv_glsl.cpp:547-553`）；加 `-V` 后两个开关都不产生代码。

结论：mesh 阶段想要深度范围换算，只能在着色器或投影矩阵里自己做，`fixup_clipspace` 在该阶段是**静默无效**而非报错。

### 5.4 细分接口：TESC 与 TESE 的责任分界

- TESC：MSL 降级为 compute + capture，无 fixup（§5.1）；GLSL 尾部 fixup 写的是 `gl_out[gl_InvocationID]` 的别名 `gl_Position`，在多控制点场景只作用于当前调用结束时刻。
- TESE：属于 vertex-like，MSL 正常生成 fixup（实测 `[[patch(...)] ] vertex` 函数尾部带 `Adjust clip-space`）。
- 细分域原点差异由独立选项 `--msl-domain-lower-left`（`tess_domain_origin_lower_left`）在**输入钩子**处理：`spirv_msl.cpp:16022-16028` 对非三角形域生成 `gl_TessCoord.y = 1.0 - gl_TessCoord.y;`；三角形域故意不翻转，注释说明交给 MoltenVK 反转绕序（`spirv_msl.cpp:16024-16025`）。参考 `reference/shaders-msl/tese/quad.domain.tese:69-71`（测试后缀 `.domain.` → CLI 映射在 `test_shaders.py:269-270`）。
- HLSL 后端根本不支持细分执行模型，直接 `SPIRV_CROSS_THROW("Unsupported execution model.")`（`spirv_hlsl.cpp:3107,3125`；`shaders-hlsl/` 下没有任何 tesc/tese 用例）。

### 5.5 几何着色器尾部修正不影响已发射顶点

如 §3 所述，`EmitVertex()`/`Append` 发生在 fixup 之前，三个后端对 GS 的 fixup 都只在函数尾执行一次。跨阶段传递的 Position 在发射时已定型，期望"靠 `--flip-vert-y` 翻转 GS 输出拓扑"的用法不会得到正确结果——应在每个 `EmitVertex()` 之前自行翻转。

### 5.6 测试覆盖本身的边界

`test_shaders.py` 的文件名后缀驱动里：

- `.fixup-clipspace.` 只接在 **MSL** 参数上（`test_shaders.py:386-387`）；
- `.flip-vert-y.` 只接在 **HLSL** 参数上（`test_shaders.py:610-611`）；
- GLSL 的 `cross_compile()` 路径从不追加这两个开关，仓库里**不存在 GLSL 侧 fixup 的参考测试**。GLSL 后端的 `2.0*z-w` 路径只能靠 CLI/API 直接验证（本报告 §2.1 的实测即为此而做）。

---

## 6. 建议（单独标识）

> **建议 1：投影矩阵优先，编译器开关兜底。** README 已明示这一立场（`README.md:439-441`）。`fixup_clipspace`/`flip_vert_y` 是无状态的尾部改写，无法感知几何着色器的 `EmitVertex` 时机、mesh 输出的逐顶点 store 或 capture buffer 路径；在这些接口上开关要么失效要么作用于错误对象。矩阵修正在所有阶段语义一致。
>
> **建议 2：同一管线对所有 stage 使用一致的选项组合，并按后端校验输出。** 三后端共用 options 结构但变换方向相反（GLSL `2z-w` vs HLSL/MSL `(z+w)/2`）、守卫不同（Position 活跃性检查、隐式 Position 注入、capture 跳过）。尤其 TESC/vertex-for-tess/mesh 阶段，应在 CI 里对生成代码做 grep 级断言（如确认无 `Adjust clip-space` 字样时应用层已接管）。
>
> **建议 3：不要假设存在片元 Y 翻转开关。** 全后端没有 `flip_frag_y`；需要 Y 翻转时在顶点侧做（并知晓 §5.5 的 GS 限制），或由应用使用负视口高度/MSL 的 framebufferFlippedDrawable 等设施处理。HLSL 侧的 `FragCoord.w` 倒数与 SM30 半像素偏移是编译器自动责任，不要在 shader 里重复处理。
>
> **建议 4：MSL 深度模拟选项带来显式 ABI 契约。** 开启 `emulate_depth_clip_enable` / `emulate_reversed_depth_viewport` 后，生成代码依赖 slot 17 的 `spvDepthClipState` 与 slot 18 的掩码缓冲，且仅在非 capture、Position/FragDepth 实际存在时注入。管线集成代码必须按 `needs_depth_clip_state_buffer()`（`spirv_msl.hpp:642`）的同一条件提供缓冲，否则修正语句根本不会生成，造成静默行为差异。
>
> **建议 5：补齐 GLSL 后端的回归测试。** 当前测试矩阵只覆盖 MSL 的 clipspace 与 HLSL 的 Y-flip，GLSL 的反向变换无参考测试；涉及该路径的改动（如 `spirv_glsl.cpp:18332`）缺乏快照保护。

---

## 7. 两个具体场景（源码 + 已有测试材料支撑）

### 场景 A：手写深度输出与三层 MSL 深度修正叠加（顶点阶段）

- **源码开关链**：`main.cpp:1715`（CLI）→ `main.cpp:1469-1470`（写入共用 options）→ `CompilerMSL::emit_fixup()`（`spirv_msl.cpp:13883-13925`），顺序为 clipspace 换算 → 视口 Z 变换 → 反向深度 → Y 翻转；缓冲注入条件在 `spirv_msl.cpp:15209-15233`，结构声明在 `spirv_msl.cpp:8558-8560`。
- **测试材料**：
  - 输入 `shaders-msl-no-opt/vert/depth-clip-enable.emulate-depth-clip-enable.emulate-reversed-depth-viewport.fixup-clipspace.vert`（手写 `gl_Position = vec4(0,0,0.25,1)`）；
  - 期望输出 `reference/shaders-msl-no-opt/vert/...fixup-clipspace.vert`（同目录另有去掉 reversed-depth 的两选项版本与无 fixup 基线，可对比选项增量）；
  - 后缀到 CLI 的映射：`test_shaders.py:382-388`。
- **复现与验证**：用 glslang 15.1（`--target-env vulkan1.1`）编译该 `.vert`，再以仓库预编译 `spirv-cross --msl --msl-emulate-depth-clip-enable --msl-emulate-reversed-depth-viewport --fixup-clipspace` 翻译，输出与参考文件 **逐字节一致（已执行 diff 验证通过）**。
- **结论**：手写值 `z=0.25` 不会被识别为"已是目标约定"；它先经 `(z+w)*0.5 → 0.625`，再按视口 range 仿射，最后按掩码可能 `w-z` 反转。应用若在 CPU 侧已做过一次 D3D→Metal 换算，再开此选项会双重换算。

### 场景 B：Mesh 着色器的 Y 翻转按写入点包装，而 Z 修正静默丢失（HLSL）

- **源码开关链**：辅助函数生成 `spirv_hlsl.cpp:2413-2424`；store 改写 `spirv_hlsl.cpp:5460-5471`；访问链标记 `spirv_glsl.cpp:11305-11310,11365-11370`；`emit_fixup()` 因 `is_vertex_like_shader()` 不含 MeshEXT（`spirv_cross.cpp:2635-2640`）而整体跳过。
- **测试材料**：
  - 输入 `shaders-hlsl-no-opt/mesh/mesh-shader-basic-triangle.spv14.vk.nocompat.nofxc.flip-vert-y.mesh`（含整向量写、`.y` 分量写、动态下标写 Position 三种形态）；
  - 期望输出 `reference/shaders-hlsl-no-opt/mesh/...flip-vert-y.mesh`，其中第 45-53 行是两个 `spvFlipVertY` 辅助函数，第 58-65 行展示三种写入点的不同包装；
  - 后缀映射：`test_shaders.py:610-611`。
- **复现与验证**：glslang 以 `--target-env spirv1.4` 编译后，`spirv-cross --hlsl --shader-model 66 --flip-vert-y` 输出与参考文件 **逐字节一致（已执行 diff 验证通过）**；对同一 SPIR-V 再加 `--fixup-clipspace`，输出中不出现任何 Z 改写语句。
- **结论**：mesh 阶段的坐标修正是"逐 store 内联"而非"入口尾统一改写"，与普通顶点阶段的修正在生成位置、覆盖面（Y 有、Z 无）上都不同；多阶段管线中若 mesh 后接的光栅化需要 `[0,w]` 深度，必须在 mesh shader 内部或上游数据里保证，开关不会报错也不会生效。

---

## 附：关键源码索引

| 主题 | 位置 |
|---|---|
| 两个共用顶点选项及方向注释 | `spirv_glsl.hpp:173-189` |
| GLSL `emit_fixup`（`2z-w`） | `spirv_glsl.cpp:18332-18344` |
| HLSL `emit_fixup`（`(z+w)/2`、HalfPixel） | `spirv_hlsl.cpp:3780-3797` |
| MSL `emit_fixup`（clipspace/视口/反向深度/Y/片元 clamp） | `spirv_msl.cpp:13883-13937` |
| Return 块钩子+fixup 触发顺序 | `spirv_glsl.cpp:19582-19589`；入口 prologue `18314-18315` |
| vertex-like 阶段定义（不含 mesh） | `spirv_cross.cpp:2635-2640` |
| MSL capture 强制路径（TESC/for-tess） | `spirv_msl.cpp:1899-1905`；外层守卫 `13885` |
| MSL 隐式 Position 注入 | `spirv_msl.cpp:1101-1173` |
| MSL 深度缓冲需求判定与 slot | `spirv_msl.hpp:642-649`、`322-323` |
| HLSL FragCoord 倒数 / SM30 半像素 | `spirv_hlsl.cpp:3430-3437` |
| HLSL mesh Y 翻转辅助函数与 store 包装 | `spirv_hlsl.cpp:2413-2424`、`5460-5471` |
| MSL mesh 逐顶点 Y 翻转 | `spirv_msl.cpp:21244-21247`（注册点 `15855`） |
| MSL TessCoord 域原点翻转 | `spirv_msl.cpp:16013-16030` |
| MSL interpolateAtOffset 0.4375 补偿 | `spirv_msl.cpp:12265-12271` |
| C API 选项枚举与落地 | `spirv_cross_c.h:628-629`、`spirv_cross_c.cpp:430-435` |
| CLI 帮助与参数绑定 | `main.cpp:1024-1028`、`1715-1716` |
| 测试后缀映射（MSL/HLSL，GLSL 缺失） | `test_shaders.py:386-387`、`610-611` |
