# SPIRV-Cross 坐标修正责任分析：裁剪空间、深度范围、Y 方向与片元坐标

> 基于仓库 `task/38` 分支（HEAD `57e4efc`）源码阅读与实测。
> 实测工具：仓库内 `make spirv-cross` 构建的 CLI，配合 glslangValidator 15.1.0 将 GLSL 编译为 SPIR-V。
> 涉及后端：**GLSL**（`spirv_glsl.cpp`）、**HLSL**（`spirv_hlsl.cpp`）、**MSL**（`spirv_msl.cpp`）。

---

## 1. 问题背景：三个 API 的坐标系约定不同

SPIR-V 以 Vulkan 语义为基准，但目标图形 API 在四个维度上约定不一致：

| 维度 | Vulkan / SPIR-V | OpenGL（GLSL 目标） | Direct3D（HLSL 目标） | Metal（MSL 目标） |
|---|---|---|---|---|
| 裁剪空间 Z 范围（NDC 深度） | `[0, w]` | `[-w, w]` | `[0, w]` | `[0, w]` |
| 裁剪空间 Y 方向 | Y 向下 | Y 向上 | Y 向上 | Y 向上 |
| 片元坐标原点（`FragCoord` / `SV_Position` / `[[position]]`） | 左上，Y 向下 | 左下，Y 向上 | 左上，Y 向下 | 左上，Y 向下 |
| 片元深度范围 | `[0,1]` | `[-1,1]`（窗口变换后） | `[0,1]` | `[0,1]` |

因此从一个 SPIR-V 模块出发，**修正责任由"源语义 → 目标语义"的差异决定**，而不是由 SPIR-V 本身决定。SPIRV-Cross 把这件事拆成两个独立的、默认关闭的入口选项（`spirv_glsl.hpp:173-190`）：

```cpp
struct VertexOptions {
    // GLSL:  [0,w]  -> [-w,w]
    // MSL/HLSL: [-w,w] -> [0,w]
    bool fixup_clipspace = false;
    // 反转 gl_Position.y 或等价物
    bool flip_vert_y = false;
    ...
};
```

注意注释本身就揭示了**同一选项在不同后端的数学方向相反**：`fixup_clipspace` 不是"修正到某一个固定约定"，而是"修正到该后端原生约定"。CLI 帮助文本（`main.cpp:1024-1028`）与 `README.md:436-448` 也重申了这一点，并建议优先修改投影矩阵而非启用这两个选项。

C API 中它们是通用选项 `SPVC_COMPILER_OPTION_FIXUP_DEPTH_CONVENTION` 与 `SPVC_COMPILER_OPTION_FLIP_VERTEX_Y`（`spirv_cross_c.h:628-629`，实现在 `spirv_cross_c.cpp:430-434`），三个后端共享同一组布尔位。

---

## 2. 两个开关在三个后端的生成语义对比

### 2.1 修正代码的发射位置与门控条件

三个后端都在入口点函数的 `Return` 块处理中调用虚函数 `emit_fixup()`（GLSL 的调用点：`spirv_glsl.cpp:19582-19589`，先跑 `fixup_hooks_out` 再跑 `emit_fixup`），但门控条件不同：

| 后端 | 源码位置 | 门控条件 |
|---|---|---|
| GLSL | `spirv_glsl.cpp:18332-18345` | `is_vertex_like_shader()` —— 仅检查执行模型，**不检查 Position 是否真的被声明/写出** |
| HLSL | `spirv_hlsl.cpp:3780-3798` | `is_vertex_like_shader() && active_output_builtins.get(BuiltInPosition)` |
| MSL | `spirv_msl.cpp:13883-13938` | `stage_out_var_id && !capture_output_to_buffer`，且 `is_vertex_like_shader() && !qual_pos_var_name.empty()` |

`is_vertex_like_shader()` 的定义在 `spirv_cross.cpp:2635-2640`，只包含 `Vertex / Geometry / TessellationControl / TessellationEvaluation`，**显式排除了 MeshEXT**——这是后面网格着色器边界的根因。

### 2.2 生成语句对比（实测）

用一个仅写 `gl_Position = inPos;` 的顶点着色器实测（SPIR-V 由 glslangValidator 生成）：

**GLSL 后端**（`spirv_glsl.cpp:18336-18343`，顺序：先 Z 后 Y）：

```glsl
// --fixup-clipspace            [0,w] -> [-w,w]
gl_Position.z = 2.0 * gl_Position.z - gl_Position.w;
// --flip-vert-y
gl_Position.y = -gl_Position.y;
```

**HLSL 后端**（`spirv_hlsl.cpp:3793-3796`，顺序：先 Y 后 Z，与 GLSL/MSL 相反）：

```hlsl
gl_Position.y = -gl_Position.y;
gl_Position.z = (gl_Position.z + gl_Position.w) * 0.5;   // [-w,w] -> [0,w]
```

**MSL 后端**（`spirv_msl.cpp:13892-13924`，顺序：先 Z 后 Y，带注释）：

```metal
out.gl_Position.z = (out.gl_Position.z + out.gl_Position.w) * 0.5;       // Adjust clip-space for Metal
out.gl_Position.y = -(out.gl_Position.y);    // Invert Y-axis for Metal
```

三者数学上互为逆变换（GLSL 的 `2z-w` 与 HLSL/MSL 的 `(z+w)/2` 复合后恒等），且 Y 翻转与 Z 重映射操作的分量互不干扰，所以语句顺序差异不产生数值分歧——但它体现了三份独立实现而非共享代码路径。

### 2.3 HLSL 独有的第三类修正：SM30 半像素偏移

HLSL 的 `emit_fixup()` 还包含一个与两个开关无关、由 shader model 触发的修正（`spirv_hlsl.cpp:3785-3791`）：

```hlsl
if (hlsl_options.shader_model <= 30) {
    gl_Position.x = gl_Position.x - gl_HalfPixel.x * gl_Position.w;
    gl_Position.y = gl_Position.y + gl_HalfPixel.y * gl_Position.w;
}
```

CLI 默认 shader model 即 30（`spirv_hlsl.hpp:112`，`main.cpp:1496-1505` 仅在显式 `--shader-model` 时覆盖），并额外声明 `uniform float4 gl_HalfPixel;`（`spirv_hlsl.cpp:1741`），由应用侧填充。这是 D3D9 像素中心取整约定的历史包袱；升到 SM50 后语义消失，`SV_Position` 直接透传。仓库参考材料 `reference/shaders-hlsl/vert/legacy-int-attribute.sm30.vert:24-25` 固定了这一输出。

### 2.4 片元坐标：责任边界在"输入侧"，且后端大多零修正

片元阶段的坐标责任与顶点侧完全不同——后端倾向于直接映射到原生内建，把约定差异留给已有的顶点侧 Y 翻转/视口设置去吸收：

- **HLSL**：`gl_FragCoord` 直接映射为 `SV_Position`（`spirv_hlsl.cpp:3424-3435`）。SM30 加 `+0.5` 半像素偏移；SM40+ 仅做 `gl_FragCoord.w = 1.0 / gl_FragCoord.w`（D3D 的 w 语义是 1/w）。**Y 与 Z 不做任何翻转**，因为 D3D 片元坐标与 Vulkan 同为左上原点、`[0,1]` 深度。
- **MSL**：映射为 `float4 gl_FragCoord [[position]]`（`spirv_msl.cpp:18755-18756`），Metal 片元 `[[position]]` 同样是左上原点，零修正。唯一的输入侧修正是 sample-rate 着色时 `gl_FragCoord.xy += get_sample_position(gl_SampleID) - 0.5;`（`spirv_msl.cpp:15953-15960`）。
- **GLSL**：直接保持 `gl_FragCoord` 名称（`spirv_glsl.cpp:10646-10647`），但目标若是桌面 GL，Y 原点差异**不会被后端自动修正**——这正是 `flip_vert_y` 选项存在的原因：在顶点侧翻转 Y 等价于负视口高度，使整条管线的坐标系一致。

实测验证（同一 SPIR-V，开关前后逐字节 diff）：

```text
IDENTICAL: flip_vert_y inert on GLSL frag
IDENTICAL: fixup_clipspace inert on GLSL frag
IDENTICAL: flags inert on HLSL frag
```

即两个开关在片元阶段完全惰性；`emit_fixup()` 的 GLSL/MSL 分支甚至不包含 fragment 路径（MSL 的 fragment 分支只处理深度裁剪模拟，见 §4）。

### 2.5 另一处独立的 Y 修正：MSL 曲面细分域原点

MSL 还有一个与 `flip_vert_y` 无关的 Y 方向选项 `tess_domain_origin_lower_left`（`spirv_msl.hpp:339`，CLI `--msl-domain-lower-left`）。Vulkan/GL 的 quad 曲面域原点在左下，Metal 在左上；启用后在 TESE 入口对输入 `TessCoord` 做 `gl_TessCoord.y = 1.0 - gl_TessCoord.y;`（`spirv_msl.cpp:16024-16030`，三角形域例外，由 MoltenVK 反转绕序代替）。这说明 **Y 方向的修正责任分散在多个互不相关的选项里**，没有统一开关。参考材料：`reference/shaders-msl/tese/quad.domain.tese:69-70`。

---

## 3. 场景一（建议）：手写深度输出与坐标修正同时出现

> **建议标识：场景一 —— 顶点侧 Z 修正与片元侧手写 `gl_FragDepth` 之间存在未被后端闭合的语义缝隙。**

### 3.1 场景构造

顶点着色器写 `gl_Position`，片元着色器同时手写深度：

```glsl
// frag
color = vec4(gl_FragCoord.xyz, 1.0);
gl_FragDepth = gl_FragCoord.z * 0.5 + 0.25;
```

### 3.2 源码事实：修正只作用于 Position，FragDepth 原样透传

三个后端对手写 `gl_FragDepth` 都只做内建名/语义映射，不做任何 Z 范围重写：

- GLSL：保持 `gl_FragDepth`（`spirv_glsl.cpp:10652-10653`），保守深度限定原样输出 `layout(depth_greater|depth_less)`（`spirv_glsl.cpp:1401-1403`）。
- HLSL：映射为 `SV_Depth`，SM50+ 按执行模式细化为 `SV_DepthGreaterEqual / SV_DepthLessEqual`，SM30 退化为 `DEPTH`（`spirv_hlsl.cpp:588-600`）。
- MSL：映射为输出结构体成员 `[[depth(any|less|greater)]]`（`spirv_msl.cpp:18772-18778`）。

实测同一片元 SPIR-V 的输出（节选）：

```glsl
/* GLSL */  gl_FragDepth = (gl_FragCoord.z * 0.5) + 0.25;
```
```hlsl
/* HLSL */  gl_FragDepth : SV_Depth;
           gl_FragDepth = (gl_FragCoord.z * 0.5f) + 0.25f;
```
```metal
/* MSL */   float gl_FragDepth [[depth(any)]];
           out.gl_FragDepth = (gl_FragCoord.z * 0.5) + 0.25;
```

### 3.3 后果分析

1. **GLSL 目标 + `fixup_clipspace` 的缝隙最明显。** 顶点侧 `gl_Position.z` 被改写成 `2z-w`（`[0,w]→[-w,w]`），但片元侧手写的 `gl_FragDepth` 表达式按 Vulkan 的 `[0,1]` 语义书写且未被触碰。OpenGL 窗口映射期望 `[-1,1]` 输入，于是手写深度与光栅化插值深度落在两套刻度上。README（`README.md:438-439`）的建议——"改投影矩阵更好"——部分原因正在于此：矩阵方案让 Position 与 FragCoord/FragDepth 的刻度天然一致，而事后修正 Position 无法回溯到片元表达式。
2. **HLSL/MSL 目标数值上自洽，但责任仍在作者。** 这两个后端的原生深度就是 `[0,1]`，`fixup_clipspace` 处理的是"SPIR-V 本身按 GL `[-w,w]` 书写"的反向情形；片元 `SV_Depth`/`[[depth]]` 与 `gl_FragCoord.z` 同刻度，透传正确——前提是作者没有混用两种约定的 SPIR-V。
3. **保守深度限定会被保留但不会改变上述结论。** 实测 `layout(depth_less)` 在 HLSL SM50 得到 `SV_DepthLessEqual`、在 MSL 得到 `[[depth(less)]]`、SM30 仅得 `DEPTH`（硬件能力降级，限定信息丢失）。

### 3.4 仓库已有测试材料

- `shaders-msl/frag/depth-out-no-early-frag-tests.frag` + `reference/shaders-msl/frag/depth-out-no-early-frag-tests.frag`：手写 `gl_FragDepth = 0.6999...` 原样进入 `[[depth(less)]]`，无任何 Z 改写。
- `shaders-no-opt/frag/conservative-depth.frag`：`layout(depth_greater)` 限定的透传。
- `shaders-hlsl-no-opt/asm/frag/only-initializer-frag-depth.asm.frag`：HLSL 侧 FragDepth 初始化器场景。

### 3.5 MSL 的两个例外：后端会"碰"手写深度的情形

MSL 在两种模拟模式下会在用户写入**之后**追加语句（`spirv_msl.cpp:13926-13936`），这是仅有的后端主动修改手写 FragDepth 的路径：

```metal
out.gl_FragDepth = (gl_FragCoord.z * 0.5) + 0.25;          // 用户原始写入
if (spvDepthClipState.emulateDepthClamp != 0u) {
    float2 spvViewportDepthRange = spvDepthClipState.viewportDepthRanges[0];
    out.gl_FragDepth = clamp(out.gl_FragDepth, spvViewportDepthRange.x, spvViewportDepthRange.y);
}
```

对应仓库材料 `shaders-msl-no-opt/frag/depth-clip-enable.emulate-depth-clip-enable.msl2.frag` 与同名 reference（第 23-28 行）。另外，当 `enable_frag_depth_builtin=false`（CLI `--msl-disable-frag-depth-builtin`，`spirv_msl.cpp:14495-14498`）时，`[[depth]]` 成员直接消失，实测手写 `gl_FragDepth` 退化为一个不返回的局部 `float`——写入被静默丢弃，这是一个需要显式知晓的边界。

---

## 4. 场景二（建议）：多阶段接口组合时暴露的边界

> **建议标识：场景二 —— 修正按"单阶段入口点"独立发射，跨阶段组合时出现重复修正、时序错位与被门控静默跳过三类边界。**

### 4.1 边界 A：几何/曲面阶段中，修正落在所有 `EmitVertex()` 之后

`emit_fixup()` 在入口点 Return 块发射。对几何着色器这意味着修正在流输出完成**之后**才执行。实测一个发射三个顶点的 geom（HLSL SM50 + `--flip-vert-y`）：

```hlsl
gl_Position = 1.0f.xxxx; geometry_stream.Append(...);   // 未翻转的值已进流
gl_Position = 2.0f.xxxx; geometry_stream.Append(...);
gl_Position = 3.0f.xxxx; geometry_stream.Append(...);
gl_Position.y = -gl_Position.y;                         // 只修改不再被读取的本地变量
gl_Position.z = (gl_Position.z + gl_Position.w) * 0.5;
```

GLSL 后端同样把 `gl_Position.y = -gl_Position.y;` 放在 `EndPrimitive()` 之后（实测输出确认）。即**几何阶段开启这两个开关对实际输出图元无效**，只改写一个生命周期结束的变量。这是 `emit_fixup` 统一挂在 Return 块（`spirv_glsl.cpp:19587-19588`）与 GS 流式输出模型之间的结构性错配。

### 4.2 边界 B：同一管线多阶段各自独立修正 → 双重变换风险

`is_vertex_like_shader()` 同时包含 vert/tesc/tese/geom，每个阶段独立编译、独立发射修正。实测 TESE（MSL，双开关）：

```metal
out.gl_Position = patchIn.gl_in[0].gl_Position;
out.gl_Position.z = (out.gl_Position.z + out.gl_Position.w) * 0.5;  // Adjust clip-space
out.gl_Position.y = -(out.gl_Position.y);                           // Invert Y-axis
```

而 TESC 在 MSL 中是 compute kernel 写设备缓冲区（`stage_out_var_id` 路径不同），实测**不发射任何修正**。GLSL 的 TESC/TESE 则都会发射。于是跨后端组合管线时：

- 若 vert 与 tese 都启用 `fixup_clipspace`，Z 被连续两次 `(z+w)/2`（或 GLSL 侧两次 `2z-w`），结果错误；
- 选项必须**只在最终产生光栅化 Position 的那个阶段开启一次**，后端不做跨阶段去重，因为它每次只看到单个 SPIR-V 模块。

仓库材料侧面印证了阶段差异：`reference/shaders-msl-no-opt/tese/depth-clip-enable.emulate-depth-clip-enable.tese` 中 TESE 有视口 Z 模拟而同名 vert 材料另有 clipspace 修正，二者是不同文件名标记驱动的独立测试（`test_shaders.py:386-387` 仅按文件名 `.fixup-clipspace.` 给 MSL 命令行加 flag）。

### 4.3 边界 C：MSL 捕获输出/禁光栅化时修正被整体跳过

MSL 的门控 `stage_out_var_id && !capture_output_to_buffer`（`spirv_msl.cpp:13885`）意味着 `--msl-capture-output`（顶点函数变 `void`、Position 写入 `device` 缓冲区）下，即使命令行带了 `--fixup-clipspace --flip-vert-y` 也**静默不发射任何修正**。实测：

```metal
vertex void main0(..., device main0_out* spvOut [[buffer(28)]], ...) {
    device main0_out& out = spvOut[...];
    out.gl_Position = in.inPos;          // 无 z、无 y 修正
}
```

对照仓库 `reference/shaders-msl/vert/basic.capture.vert`（`.capture.` 文件名触发 `--msl-capture-output`，`test_shaders.py:267-268`）。责任转移给读取缓冲区的后续 compute/顶点阶段。`--msl-disable-rasterization` 路径不同（函数体仍在），实测修正保留。

### 4.4 边界 D：Position 未声明时三后端行为分裂

实测一个只写自定义输出、从不声明 `gl_Position` 的 vert，加上两个开关：

| 后端 | 行为 | 源码依据 |
|---|---|---|
| GLSL | 仍发射 `gl_Position.z = 2.0*...`，依赖 GLSL 内建全局的隐式声明（读取未初始化值） | `spirv_glsl.cpp:18334` 只查执行模型 |
| HLSL | 完全不发射（`active_output_builtins` 无 Position） | `spirv_hlsl.cpp:3782` |
| MSL | **主动合成一个隐式 Position 内建**再修正零初始化变量 | `spirv_msl.cpp:1102-1173`（`need_position` → `mark_implicit_builtin(BuiltInPosition)`） |

MSL 合成的理由是 Metal 顶点函数返回结构体时必须带 `[[position]]`；配合 `--msl-auto-disable-rasterization` 则改为直接禁用光栅化、不合成 Position（实测结构体中该成员消失）。这意味着同一开关在三后端对"缺 Position 的模块"产生不同的接口形状。仓库 `shaders-msl/masking/write-outputs.mask-position.tesc` + reference 还展示了另一变体：Position 被 `--mask-stage-output-builtin Position` 掩码后转入 `gl_out_masked` 线程组缓冲区，正常 stage out 中不再存在该内建。

### 4.5 边界 E：Mesh 着色器是三套完全不同的专用路径

MeshEXT 不属于 `is_vertex_like_shader()`，常规 `emit_fixup()` 不覆盖它，各后端另起炉灶：

- **HLSL**：`flip_vert_y` 下沉到**每一条 store 指令**。`spirv_hlsl.cpp:5460-5471` 在 `emit_store` 中检查表达式标记 `access_meshlet_position_y`（标记在访问链构造时按 `BuiltInPosition + ExecutionModelMeshEXT` 打上，`spirv_glsl.cpp:11305-11310`），把对 Position 的写包进辅助函数 `spvFlipVertY`（定义于 `spirv_hlsl.cpp:2413-2425`，含 `float4` 与标量两个重载）。`fixup_clipspace` 在 mesh 下**无任何实现路径**。
- **MSL**：在逐顶点回填循环 `spvMesh.set_vertex` 之前对 `spvV` 的 Position 成员做 `.y` 翻转（`spirv_msl.cpp:21244-21248`），同样只有 Y、没有 Z 修正。
- **GLSL**：mesh 要求 `--vulkan-semantics`（否则抛异常 "Mesh shaders require Vulkan semantics"），而 Vulkan 语义下目标即源约定，两个开关均无的放矢，实测无修正语句。

仓库材料：`shaders-hlsl-no-opt/mesh/mesh-shader-basic-triangle.spv14.vk.nocompat.nofxc.flip-vert-y.mesh` 与 reference（第 45-65 行），可以看到标量分量写、swizzle 写、动态索引写分别被包成不同形态的 `spvFlipVertY(...)`；另一份 asm 来源材料 `...mesh-shader-dxil-spirv.flip-vert-y...` 第 44 行展示了单分量 `.y` 写入的包装。实测自构 mesh SPIR-V 复现了三后端的上述差异（HLSL 三次 `spvFlipVertY`、MSL 循环内一次 `.y = -(...)`、GLSL 零修正）。

### 4.6 边界 F：片元输入侧的隐式深度回写与显式写入互斥

MSL 的 `input_attachment_is_ds_attachment` 模式下，若片元着色器读取深度/模板输入附件但**没有**自己写 FragDepth，后端会合成 `out.gl_FragDepth = gl_FragCoord.z;` 回写（`spirv_msl.cpp:16459-16465`，门控 `!writes_to_depth`），防止 Metal 丢弃附件内容；一旦作者显式写了 `gl_FragDepth`，回写抑制。仓库对照材料：

- `shaders-msl/frag/force-depth-write.input-attachment-is-ds-attachment.frag`（无显式写）→ reference 第 16 行 `out.gl_FragDepth = gl_FragCoord.z;`
- `shaders-msl/frag/force-depth-write-false.input-attachment-is-ds-attachment.frag`（显式 `= 1.0`）→ reference 第 16 行保留用户值、无回写。

这是"接口组合"在片元侧的对应物：子通道输入附件的存在改变了输出接口的形状，而作者手写深度会改变后端的合成决策。

---

## 5. 责任划分总结

| 修正事项 | GLSL | HLSL | MSL |
|---|---|---|---|
| 裁剪空间 Z 范围 | `2z-w`（`[0,w]→[-w,w]`） | `(z+w)*0.5`（反向） | `(z+w)*0.5`（反向） |
| 顶点 Y 翻转 | `-y`，Return 块 | `-y`，Return 块（mesh 除外） | `-y`，Return 块（mesh 在回填循环） |
| 修正发射阶段 | vert/tesc/tese/geom | vert/tesc/tese/geom（需 Position 活跃） | vert/tese（需 stage out 且未 capture） |
| Mesh 阶段 | 不支持（需 Vulkan 语义） | 仅 Y，store 级 `spvFlipVertY` | 仅 Y，set_vertex 循环内 |
| 半像素（D3D9） | 无 | SM≤30 自动，`gl_HalfPixel`  uniform | 无 |
| 片元坐标原点 | 不修正（靠顶点 Y 翻转间接闭合） | 原生同约定，仅 `w=1/w` | 原生同约定；sample-rate 加采样位置修正 |
| 片元深度手写值 | 原样透传 | 原样透传（语义名按 SM 降级） | 原样透传；depth-clamp 模式追加 clamp；可被 disable 选项丢弃 |
| 曲面域 Y 原点 | 无 | 无 | `tess_domain_origin_lower_left`，输入侧 `1-y` |
| 多阶段去重 | 无 | 无 | 无（调用方负责只开一次） |

**核心结论：**

1. SPIRV-Cross 把跨 API 坐标修正建模为**两个默认关闭、按阶段独立生效的顶点侧布尔选项**，外加 MSL/HLSL 各自的专用模拟选项；不存在覆盖整条管线的统一修正层。
2. `fixup_clipspace` 的方向是后端相关的（同名选项，GLSL 与 HLSL/MSL 互为逆变换）；`flip_vert_y` 方向一致但发射点位随阶段/后端变化。
3. 手写 `gl_FragDepth` 不参与任何 Z 刻度重写——顶点侧修正与片元侧手写深度之间的语义缝隙由着色器作者/引擎负责闭合，GLSL 目标风险最高。
4. 多阶段组合的真实边界包括：GS 中修正晚于 `EmitVertex` 而失效、跨阶段重复变换、capture-output 下静默跳过、Position 缺失时三后端接口形状分裂、mesh 的三套专用路径，以及 MSL 深度附件的隐式回写与显式写入互斥。

## 附：实测复现方式

```bash
# 仓库根目录
make -j4 spirv-cross
glslangValidator -V shader.vert -o shader.vert.spv
./spirv-cross --fixup-clipspace --flip-vert-y shader.vert.spv          # GLSL
./spirv-cross --hlsl --shader-model 50 --fixup-clipspace shader.vert.spv
./spirv-cross --msl  --fixup-clipspace --msl-emulate-depth-clip-enable shader.vert.spv
```

本报告引用的全部仓库 reference 输出均由 `test_shaders.py` 按文件名关键字（`.fixup-clipspace.` → 第 386-387 行、`.flip-vert-y.` → 第 610-611 行、`.capture.` → 第 267-268 行、`.domain.` → 第 270 行、`.emulate-depth-clip-enable.` → 第 384-385 行）装配命令行后回归生成，可作为持续验证基线。
