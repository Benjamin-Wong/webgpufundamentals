Title: WebGPU 跨阶段变量（Inter-stage Variables）
Description: 从顶点着色器传递数据到片段着色器
TOC: 跨阶段变量

在[上一篇文章](webgpu-fundamentals.html)中，我们介绍了 WebGPU 的一些基础知识。本文将介绍*跨阶段变量*（inter-stage variables）的基础知识。

跨阶段变量在顶点着色器和片段着色器之间传递数据。

当顶点着色器输出 3 个位置时，一个三角形就会被光栅化。顶点着色器可以在每个位置上输出额外的值，默认情况下，这些值会在 3 个顶点之间进行插值。

让我们来看一个小例子。我们从上一篇文章中的三角形着色器开始，所要做的只是修改着色器。

```js
  const module = device.createShaderModule({
-    label: 'our hardcoded red triangle shaders',
+    label: 'our hardcoded rgb triangle shaders',
    code: /* wgsl */ `
+      struct OurVertexShaderOutput {
+        @builtin(position) position: vec4f,
+        @location(0) color: vec4f,
+      };

      @vertex fn vs(
        @builtin(vertex_index) vertexIndex : u32
-      ) -> @builtin(position) vec4f {
+      ) -> OurVertexShaderOutput {
        let pos = array(
          vec2f( 0.0,  0.5),  // top center
          vec2f(-0.5, -0.5),  // bottom left
          vec2f( 0.5, -0.5)   // bottom right
        );
+        var color = array<vec4f, 3>(
+          vec4f(1, 0, 0, 1), // red
+          vec4f(0, 1, 0, 1), // green
+          vec4f(0, 0, 1, 1), // blue
+        );

-        return vec4f(pos[vertexIndex], 0.0, 1.0);
+        var vsOutput: OurVertexShaderOutput;
+        vsOutput.position = vec4f(pos[vertexIndex], 0.0, 1.0);
+        vsOutput.color = color[vertexIndex];
+        return vsOutput;
      }

-      @fragment fn fs() -> @location(0) vec4f {
-        return vec4f(1, 0, 0, 1);
+      @fragment fn fs(fsInput: OurVertexShaderOutput) -> @location(0) vec4f {
+        return fsInput.color;
      }
    `,
  });
```

首先，我们声明一个 `struct`（结构体）。这是在顶点着色器和片段着色器之间协调跨阶段变量的一种简便方法。

```wgsl
      struct OurVertexShaderOutput {
        @builtin(position) position: vec4f,
        @location(0) color: vec4f,
      };
```

然后，我们声明顶点着色器返回该类型的结构体：

```wgsl
      @vertex fn vs(
        @builtin(vertex_index) vertexIndex : u32
-      ) -> @builtin(position) vec4f {
+      ) -> OurVertexShaderOutput {
```

我们创建一个有 3 种颜色的数组。

```wgsl
        var color = array<vec4f, 3>(
          vec4f(1, 0, 0, 1), // red
          vec4f(0, 1, 0, 1), // green
          vec4f(0, 0, 1, 1), // blue
        );
```

然后，我们不再仅返回一个 `vec4f` 表示位置，而是声明一个结构体实例，填充它的字段并返回：

```wgsl
-        return vec4f(pos[vertexIndex], 0.0, 1.0);
+        var vsOutput: OurVertexShaderOutput;
+        vsOutput.position = vec4f(pos[vertexIndex], 0.0, 1.0);
+        vsOutput.color = color[vertexIndex];
+        return vsOutput;
```

在片段着色器中，我们将该结构体声明为函数的参数：

```wgsl
      @fragment fn fs(fsInput: OurVertexShaderOutput) -> @location(0) vec4f {
        return fsInput.color;
      }
```

然后返回其中的颜色。

运行后我们会发现，GPU 每次调用片段着色器时，传入的颜色值都是在 3 个顶点之间插值得到的。

{{{example url="../webgpu-inter-stage-variables-triangle.html"}}}

跨阶段变量最常见的用途是在三角形内插值纹理坐标，我们将在[纹理相关文章](webgpu-textures.html)中介绍。另一个常见用途是在三角形内插值法线，这将在[第一篇光照文章](webgpu-lighting-directional.html)中介绍。

## 跨阶段变量通过 `location` 连接

重要的一点是，与 WebGPU 中几乎所有东西一样，顶点着色器和片段着色器之间的连接是通过索引实现的。对于跨阶段变量来说，它们通过 `location` 索引进行连接。

为了理解这一点，让我们只修改片段着色器，让它在 `location(0)` 处接受一个 `vec4f` 参数，而不是使用结构体：

```wgsl
      @fragment fn fs(@location(0) color: vec4f) -> @location(0) vec4f {
        return color;
      }
```

运行后可以看到，它仍然能正常工作。

{{{example url="../webgpu-inter-stage-variables-triangle-by-fn-param.html"}}}

## `@builtin(position)`

这也揭示了另一个特殊之处。我们最初的着色器在顶点着色器和片段着色器中使用了同一个结构体，其中有一个名为 `position` 的字段，但它没有 `location`，而是被声明为 `@builtin(position)`。

```wgsl
      struct OurVertexShaderOutput {
*        @builtin(position) position: vec4f,
        @location(0) color: vec4f,
      };
```

该字段**不是**跨阶段变量，而是一个内置变量（`builtin`）。`@builtin(position)` 在顶点着色器和片段着色器中具有不同的含义。实际上，更好的理解方式是：顶点着色器和片段着色器只是两个恰好有一个同名参数的不同函数。

想象一下我们有两个 JavaScript 函数：

```js
// 在位置 [x, y] 处绘制半径为 radius 的圆
function drawCircle({ ctx, position, radius }) {
  // 来自 CanvasRenderingContext2D
  ctx.beginPath();
  ctx.arc(...position, radius, 0, Math.PI * 2);
  ctx.fill();
}

// 从 position 开始，返回数组中某个元素的索引
function findIndex({ array, position, value }) {
  return array.indexOf(value, position);
}
```

上面两个函数都有一个名为 `position` 的参数，但它们之间并不会产生混淆。顶点着色器和片段着色器也是类似的——它们的内置变量是不同且不相关的，只是恰好都有一个名为 `position` 的 `@builtin`。编译每个着色器入口点时，WGSL 代码只为该入口点单独解析。

在顶点着色器中，`@builtin(position)` 是你提供给 GPU 的输出坐标，GPU 用它来绘制三角形/线/点。

在片段着色器中，`@builtin(position)` 是一个输入，它是片段着色器当前被要求计算颜色的像素坐标。

像素坐标以像素的边缘为基准。传给片段着色器的值是像素中心的坐标。

如果我们要绘制的纹理大小为 3x2 像素，那么坐标如下图所示：

<div class="webgpu_center"><img src="resources/webgpu-pixels.svg" style="width: 500px;"></div>

我们可以更改着色器来使用这个位置。例如，让我们画一个棋盘格。

```js
const module = device.createShaderModule({
    label: 'our hardcoded checkerboard triangle shaders',
    code: /* wgsl */ `
      struct OurVertexShaderOutput {
        @builtin(position) position: vec4f,
-        @location(0) color: vec4f,
      };

      @vertex fn vs(
        @builtin(vertex_index) vertexIndex : u32
      ) -> OurVertexShaderOutput {
        let pos = array(
          vec2f( 0.0,  0.5),  // top center
          vec2f(-0.5, -0.5),  // bottom left
          vec2f( 0.5, -0.5)   // bottom right
        );
-        var color = array<vec4f, 3>(
-          vec4f(1, 0, 0, 1), // red
-          vec4f(0, 1, 0, 1), // green
-          vec4f(0, 0, 1, 1), // blue
-        );

        var vsOutput: OurVertexShaderOutput;
        vsOutput.position = vec4f(pos[vertexIndex], 0.0, 1.0);
-        vsOutput.color = color[vertexIndex];
        return vsOutput;
      }

      @fragment fn fs(fsInput: OurVertexShaderOutput) -> @location(0) vec4f {
-        return fsInput.color;
+        let red = vec4f(1, 0, 0, 1);
+        let cyan = vec4f(0, 1, 1, 1);
+
+        let grid = vec2u(fsInput.position.xy) / 8;
+        let checker = (grid.x + grid.y) % 2 == 1;
+
+        return select(red, cyan, checker);
      }
    `,
});
```

上面的代码取 `fsInput.position`（声明为 `@builtin(position)`）的 `xy` 坐标，将其转换为 `vec2u`（2 个无符号整数）。然后除以 8，这样每 8 个像素计数增加 1。接着将 x 和 y 网格坐标相加，对 2 取模，并将结果与 1 比较，得到一个每隔一个整数交替为 true/false 的布尔值。最后使用 WGSL 函数 `select`，根据布尔条件在两个值中选择一个。用 JavaScript 来表达 `select` 的话：

```js
// If condition is false return `a`, otherwise return `b`
select = (a, b, condition) => (condition ? b : a);
```

{{{example url="../webgpu-fragment-shader-builtin-position.html"}}}

即使在片段着色器中不使用 `@builtin(position)`，它的存在也很方便，因为这意味着我们可以在顶点着色器和片段着色器中使用同一个结构体。需要强调的是，顶点着色器中的 `position` 字段与片段着色器中的 `position` 字段**完全无关**，它们是完全不同的变量。

但如前所述，对于跨阶段变量来说，真正重要的是 `@location(?)`。因此，为顶点着色器的输出和片段着色器的输入声明不同的结构体是很常见的。

为了让这一点更清楚：在我们的示例中，顶点着色器和片段着色器写在同一个字符串里只是为了方便。我们也完全可以将它们拆分到不同的模块中：

```js
-  const module = device.createShaderModule({
-    label: 'hardcoded checkerboard triangle shaders',
+  const vsModule = device.createShaderModule({
+    label: 'hardcoded triangle',
    code: /* wgsl */ `
      struct OurVertexShaderOutput {
        @builtin(position) position: vec4f,
      };

      @vertex fn vs(
        @builtin(vertex_index) vertexIndex : u32
      ) -> OurVertexShaderOutput {
        let pos = array(
          vec2f( 0.0,  0.5),  // top center
          vec2f(-0.5, -0.5),  // bottom left
          vec2f( 0.5, -0.5)   // bottom right
        );

        var vsOutput: OurVertexShaderOutput;
        vsOutput.position = vec4f(pos[vertexIndex], 0.0, 1.0);
        return vsOutput;
      }
+    `,
+  });
+
+  const fsModule = device.createShaderModule({
+    label: 'checkerboard',
+    code: /* wgsl */ `
-      @fragment fn fs(fsInput: OurVertexShaderOutput) -> @location(0) vec4f {
+      @fragment fn fs(@builtin(position) pixelPosition: vec4f) -> @location(0) vec4f {
        let red = vec4f(1, 0, 0, 1);
        let cyan = vec4f(0, 1, 1, 1);

-        let grid = vec2u(fsInput.position.xy) / 8;
+        let grid = vec2u(pixelPosition.xy) / 8;
        let checker = (grid.x + grid.y) % 2 == 1;

        return select(red, cyan, checker);
      }
    `,
  });
```

相应地，我们也需要更新管线创建的代码：

```js
  const pipeline = device.createRenderPipeline({
    label: 'hardcoded checkerboard triangle pipeline',
    layout: 'auto',
    vertex: {
-      module,
+      module: vsModule,
    },
    fragment: {
-      module,
+      module: fsModule,
      targets: [{ format: presentationFormat }],
    },
  });

```

效果完全一样：

{{{example url="../webgpu-fragment-shader-builtin-position-separate-modules.html"}}}

关键在于，大多数 WebGPU 示例中把两个着色器放在同一个字符串里只是为了方便。实际上，WebGPU 首先会解析 WGSL，确保语法正确。然后分别查看你指定的每个 `entryPoint`，只关注该入口点引用的部分，不处理其他内容。

共享字符串的好处是：多个着色器可以共用结构体、绑定和组位置、常量以及函数，无需重复编写。但从 WebGPU 的视角来看，这就好像你为每个入口点各复制了一份所有内容。

注意：使用 `@builtin(position)` 生成棋盘格并不常见。棋盘格和其他图案更常见的做法是[使用纹理](webgpu-textures.html)。实际上，如果你调整窗口大小，就会发现一个问题——因为棋盘格是基于画布的像素坐标生成的，它是相对于画布的，而不是相对于三角形的。

## <a id="a-interpolate"></a>插值设置（Interpolation Settings）

我们在上面看到，跨阶段变量（顶点着色器的输出）在传递给片段着色器时会被插值。有两组设置可以改变插值行为——插值类型和插值采样。将它们设为非默认值并不常见，但在其他文章中会介绍一些用例。

插值类型:

-   `perspective`: 以正确的透视方式插值 (**默认**)
-   `linear`: 以线性、非透视校正的方式插值
-   `flat`: 不进行插值。使用 `flat` 时不使用插值采样

插值采样:

-   `center`: 插值在像素中心进行 (**默认**)
-   `centroid`: 在当前图元中片段所覆盖的所有采样点范围内的某一点进行插值。该值对图元中的所有采样点相同。
-   `sample`: 逐采样点进行插值。使用此属性时，每个采样点都会调用一次片段着色器。
-   `first`: 仅在插值类型为 `flat` 时使用。（**默认**）取图元第一个顶点的值
-   `either`: 仅在插值类型为 `flat` 时使用。取图元的第一个或最后一个顶点的值（具体取决于实现）。

你可以通过属性来指定它们。例如：

```wgsl
  @location(2) @interpolate(linear, center) myVariableFoo: vec4f;
  @location(3) @interpolate(flat) myVariableBar: vec4f;
```

请注意，如果跨阶段变量是整数类型，则必须将其插值设为 `flat`。

如果将插值类型设为 `flat`，默认情况下传递给片段着色器的值是该三角形第一个顶点的跨阶段变量值。对于大多数 `flat` 的用例，你应该选择 `either`。原因我们将在[另一篇文章](webgpu-compatibility-mode.html#flat)中介绍。

[在下一篇文章中，我们将介绍 uniforms](webgpu-uniforms.html)，这是另一种向着色器传递数据的方式。
