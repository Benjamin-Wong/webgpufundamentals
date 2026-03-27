Title: WebGPU 透明与混合
Description: 在 WebGPU 中混合像素
TOC: 透明与混合

透明与混合这个主题不太容易系统讲清，
因为很多时候，不同场景下需要采用的做法并不一样。
所以这篇文章更多会像一份 WebGPU 相关特性的导览，
这样等我们后面讲具体技术时，就可以回过头来引用这里的内容。

## <a href="a-alphamode"></a>Canvas `alphaMode`

首先要注意的一点是，WebGPU 内部有透明与混合，
但 WebGPU canvas 与 HTML 页面之间也同样存在透明与混合。

默认情况下，WebGPU canvas 是不透明的。
它的 alpha 通道会被忽略。
如果想让它不被忽略，
我们在调用 `configure` 时就必须把它的 `alphaMode`
设为 `'premultiplied'`。
默认值是 `'opaque'`。

```js
  context.configure({
    device,
    format: presentationFormat,
+    alphaMode: 'premultiplied',
  });
```

理解 `alphaMode: 'premultiplied'` 到底是什么意思非常重要。
它的意思是：
你写进 canvas 的颜色值，必须已经先乘过 alpha 值。

我们来做一个尽可能小的例子。
只创建一个 render pass，然后设置 clear color。

```js
async function main() {
  const adapter = await navigator.gpu?.requestAdapter();
  const device = await adapter?.requestDevice();
  if (!device) {
    fail('need a browser that supports WebGPU');
    return;
  }

  // Get a WebGPU context from the canvas and configure it
  const canvas = document.querySelector('canvas');
  const context = canvas.getContext('webgpu');
  const presentationFormat = navigator.gpu.getPreferredCanvasFormat();
  context.configure({
    device,
    format: presentationFormat,
+    alphaMode: 'premultiplied',
  });

  const clearValue = [1, 0, 0, 0.01];
  const renderPassDescriptor = {
    label: 'our basic canvas renderPass',
    colorAttachments: [
      {
        // view: <- to be filled out when we render
        clearValue,
        loadOp: 'clear',
        storeOp: 'store',
      },
    ],
  };

  function render() {
    const encoder = device.createCommandEncoder({ label: 'clear encoder' });
    const canvasTexture = context.getCurrentTexture();
    renderPassDescriptor.colorAttachments[0].view =
        canvasTexture.createView();

    const pass = encoder.beginRenderPass(renderPassDescriptor);
    pass.end();

    const commandBuffer = encoder.finish();
    device.queue.submit([commandBuffer]);
  }

  const observer = new ResizeObserver(entries => {
    for (const entry of entries) {
      const canvas = entry.target;
      const width = entry.contentBoxSize[0].inlineSize;
      const height = entry.contentBoxSize[0].blockSize;
      canvas.width = Math.max(1, Math.min(width, device.limits.maxTextureDimension2D));
      canvas.height = Math.max(1, Math.min(height, device.limits.maxTextureDimension2D));
      render();
    }
  });
  observer.observe(canvas);
}
```

再给 canvas 的 CSS 背景设置成一个灰色棋盘格

```css
canvas {
  background-color: #404040;
  background-image:
     linear-gradient(45deg, #808080 25%, transparent 25%),
     linear-gradient(-45deg, #808080 25%, transparent 25%),
     linear-gradient(45deg, transparent 75%, #808080 75%),
     linear-gradient(-45deg, transparent 75%, #808080 75%);
  background-size: 32px 32px;
  background-position: 0 0, 0 16px, 16px -16px, -16px 0px;
}
```

再加一个 UI，让我们可以调节
clear value 的 alpha 和颜色，
以及它是否已经做过 premultiply。

```js
+import GUI from '../3rdparty/muigui-0.x.module.js';

...

+  const color = [1, 0, 0];
+  const settings = {
+    premultiply: false,
+    color,
+    alpha: 0.01,
+  };
+
+  const gui = new GUI().onChange(render);
+  gui.add(settings, 'premultiply');
+  gui.add(settings, 'alpha', 0, 1);
+  gui.addColor(settings, 'color');

  function render() {
    const encoder = device.createCommandEncoder({ label: 'clear encoder' });
    const canvasTexture = context.getCurrentTexture();
    renderPassDescriptor.colorAttachments[0].view =
        canvasTexture.createView();

+    const { alpha } = settings;
+    clearValue[3] = alpha;
+    if (settings.premultiply) {
+      // premultiply the colors by the alpha
+      clearValue[0] = color[0] * alpha;
+      clearValue[1] = color[1] * alpha;
+      clearValue[2] = color[2] * alpha;
+    } else {
+      // use un-premultiplied colors
+      clearValue[0] = color[0];
+      clearValue[1] = color[1];
+      clearValue[2] = color[2];
+    }

    const pass = encoder.beginRenderPass(renderPassDescriptor);
    pass.end();

    const commandBuffer = encoder.finish();
    device.queue.submit([commandBuffer]);
  }
```

如果运行这个例子，我希望你能看到一个问题

{{{example url="../webgpu-canvas-alphamode-premultiplied.html"}}}

这里显示出来的颜色其实是**未定义的**！

在我的机器上，我看到的是下面这些颜色

<img src="resources/canvas-invalid-color.png" class="center" style="width: 440px">

看出问题了吗？
我们的 alpha 设置成了 0.01。
背景本来应该是中灰和深灰相间。
颜色则设成了红色 `(1, 0, 0)`。
把 0.01 的红色叠到一个中灰 / 深灰棋盘格上，
理论上应该几乎看不出来。
那为什么结果却是两种很亮的粉色？

原因是，**这是一个非法颜色！**
我们 canvas 里的颜色是 `1, 0, 0, 0.01`，
但这并不是一个 premultiplied 颜色。
所谓 “premultiplied”，意思是写进 canvas 的颜色值
必须已经先乘过 alpha。
如果 alpha 值是 0.01，
那其他通道值就都不应该大于 0.01。

如果你勾选 `'premultiplied'` 这个复选框，
代码就会对颜色做 premultiply。
写进 canvas 的值会变成
`0.01, 0, 0, 0.01`，
这样看起来就正确了，几乎察觉不到。

在勾选 `'premultiplied'` 的情况下，
再去调节 alpha，
你会看到随着 alpha 接近 1，
颜色会逐渐过渡到纯红。

> 注意：因为示例中的 `1, 0, 0, 0.01` 是非法颜色，
> 所以它最终会怎样显示，是未定义行为。
> 浏览器可以自行决定如何处理非法颜色，
> 所以不要去使用非法颜色，
> 也不要期待不同设备上会得到一样的结果。

假设我们的颜色是 `1, 0.5, 0.25`，也就是橙色，
而我们希望它有 33% 的透明度，
那么 alpha 就是 0.33。
此时，我们的“premultiplied color”应该是

```
                      premultiplied
   ---------------------------------
   r = 1    * 0.33   = 0.33
   g = 0.5  * 0.33   = 0.165
   g = 0.25 * 0.33   = 0.0825
   a = 0.33          = 0.33
```

至于如何得到 premultiplied 颜色，由你自己决定。
如果你手里的是未 premultiply 的颜色，
那么在着色器里也可以这样做 premultiply

```wgsl
   return vec4f(color.rgb * color.a, color.a)`;
```

我们在[导入纹理那篇文章](webgpu-importing-textures.html)
里讲过的 `copyExternalImageToTexture` 函数，
支持一个 `premultipliedAlpha: true` 选项。
（[见下文](#copyExternalImageToTexture)）
这意味着，当你通过 `copyExternalImageToTexture`
把图像加载到纹理里时，
可以让 WebGPU 在复制进纹理的同时，
顺便帮你把颜色 premultiply。
这样之后你在调用 `textureSample` 时，
取到的值就已经是 premultiplied 的了。

这一节的重点是：

1. 解释 WebGPU canvas 配置项 `alphaMode: 'premultiplied'`

   它让 WebGPU canvas 具备透明能力。

2. 介绍 premultiplied alpha 颜色这一概念

   至于你如何得到 premultiplied 颜色，由你自己决定。
   上面的例子里，我们是在 JavaScript 中
   构造了一个 premultiplied 的 `clearValue`。

   我们也可以从片元着色器（以及 / 或者）
   其他着色器里返回颜色。
   我们可能会直接把 premultiplied 颜色传给这些着色器，
   也可能在着色器内部再做乘法，
   也可能通过后处理 pass
   去做 premultiply。
   重要的是，如果我们使用的是 `alphaMode: 'premultiplied'`，
   那么无论通过什么方式，
   canvas 里的颜色最终都必须是 premultiplied 的。

   关于 premultiplied 与 un-premultiplied 颜色，
   一个很好的参考文章是：
   [GPUs prefer premultiplication](https://www.realtimerendering.com/blog/gpus-prefer-premultiplication/)。

## <a href="a-discard"></a>Discard

`discard` 是一个 WGSL 语句，
你可以在片元着色器中使用它，丢弃当前片元。
换句话说，就是不去绘制这个像素。

让我们拿[跨阶段变量那篇文章](webgpu-inter-stage-variables.html#a-builtin-position)中，
利用 `@builtin(position)` 在片元着色器里绘制棋盘格的例子来改。

这次我们不再绘制双色棋盘格，
而是在两种情况中的一种直接 `discard`。

```wgsl
@fragment fn fs(fsInput: OurVertexShaderOutput) -> @location(0) vec4f {
-  let red = vec4f(1, 0, 0, 1);
  let cyan = vec4f(0, 1, 1, 1);

  let grid = vec2u(fsInput.position.xy) / 8;
  let checker = (grid.x + grid.y) % 2 == 1;

+        if (checker) {
+          discard;
+        }
+
+        return cyan;

-  return select(red, cyan, checker);
}
```

还需要做一些别的修改。
我们把上面那段 CSS 也加进来，
让 canvas 有一个 CSS 棋盘格背景。
同时把 `alphaMode` 设为 `'premultiplied'`。
然后把 `clearValue`
设成 `[0, 0, 0, 0]`。

```js
  context.configure({
    device,
    format: presentationFormat,
+    alphaMode: 'premultiplied',
  });

  ...

  const renderPassDescriptor = {
    label: 'our basic canvas renderPass',
    colorAttachments: [
      {
        // view: <- to be filled out when we render
-        clearValue: [0.3, 0.3, 0.3, 1],
+        clearValue: [0, 0, 0, 0],
        loadOp: 'clear',
        storeOp: 'store',
      },
    ],
  };
...

```

{{{example url="../webgpu-transparency-fragment-shader-discard.html"}}}

你应该会看到每隔一个格子就是“透明”的，
因为它们实际上根本没有被绘制。

在用于透明效果的着色器中，
一个很常见的做法就是根据 alpha 值来 `discard`，
例如

```wgsl
@fragment fn fs(fsInput: OurVertexShaderOutput) -> @location(0) vec4f {
    let color = ... compute a color ....

    if (color.a < threshold) {
      discard;
    }

    return color;
}
```

其中 `threshold`
可以来自 uniform，也可以是常量，
或者任何适合你场景的来源。

这种做法大概最常见于 sprite，
以及草、叶子之类的植被。
因为如果我们在绘制时还使用了深度纹理，
就像我们在[正交投影那篇文章](webgpu-orthograpic-projection.html#a-depth-textures)
里介绍过的那样，
那么当我们绘制一个 sprite、树叶或草叶时，
它后面的其他 sprite、树叶或草叶
就都不会被画出来，即使当前像素的 alpha 实际上是 0，
因为我们仍然会更新深度纹理。
所以，与其“画出来但透明”，更合理的做法是直接 `discard`。
这个话题我们会在后面的文章里再详细展开。

## <a href="a-blending"></a>混合设置

终于讲到 blend settings 了。
当你创建 render pipeline 时，
对于片元着色器中的每个 `target`，
你都可以设置 blending 状态。
换句话说，下面是我们前面的示例里常见的一种 pipeline

```js
    const pipeline = device.createRenderPipeline({
      label: 'hardcoded textured quad pipeline',
      layout: pipelineLayout,
      vertex: {
        module,
      },
      fragment: {
        module,
        targets: [
          {
            format: presentationFormat,
          },
        ],
      },
    });
```

而下面是在 `target[0]` 上添加 blending 之后的版本

```js
    const pipeline = device.createRenderPipeline({
      label: 'hardcoded textured quad pipeline',
      layout: pipelineLayout,
      vertex: {
        module,
      },
      fragment: {
        module,
        targets: [
          {
            format: presentationFormat,
+            blend: {
+              color: {
+                srcFactor: 'one',
+                dstFactor: 'one-minus-src-alpha'
+              },
+              alpha: {
+                srcFactor: 'one',
+                dstFactor: 'one-minus-src-alpha'
+              },
+            },
          },
        ],
      },
    });
```

完整的默认设置如下：

```js
blend: {
  color: {
    operation: 'add',
    srcFactor: 'one',
    dstFactor: 'zero',
  },
  alpha: {
    operation: 'add',
    srcFactor: 'one',
    dstFactor: 'zero',
  },
}
```

其中 `color` 对应颜色里的 `rgb` 部分，
`alpha` 对应 `a`（alpha）部分。

`operation` 可以取以下值之一

  * 'add'
  * 'subtract'
  * 'reverse-subtract'
  * 'min'
  * 'max'

`srcFactor` 和 `dstFactor` 则都可以取以下值之一

  * 'zero'
  * 'one'
  * 'src'
  * 'one-minus-src'
  * 'src-alpha'
  * 'one-minus-src-alpha'
  * 'dst'
  * 'one-minus-dst'
  * 'dst-alpha'
  * 'one-minus-dst-alpha'
  * 'src-alpha-saturated'
  * 'constant'
  * 'one-minus-constant'

这些大多数都比较直观。
可以把它理解成

```
   result = operation((src * srcFactor),  (dst * dstFactor))
```

其中 `src` 是你的片元着色器返回的值，
`dst` 是你当前正在绘制到的纹理中，
已经存在的值。

看一下默认情况：
`operation` 是 `'add'`，
`srcFactor` 是 `'one'`，
`dstFactor` 是 `'zero'`。
那么就得到

```
   result = add((src * 1), (dst * 0))
   result = add(src * 1, dst * 0)
   result = add(src, 0)
   result = src;
```

可以看到，默认结果最终就只是 `src`。

上面的 blend factor 中，有 2 个提到了常量：
`'constant'`
和 `'one-minus-constant'`。
这里说的 constant，
是在 render pass 中通过 `setBlendConstant` 命令设置的，
默认值是 `[0, 0, 0, 0]`。
这让你可以在不同 draw 之间动态修改它。

最常见的一种混合设置，大概是

```js
{
  operation: 'add',
  srcFactor: 'one',
  dstFactor: 'one-minus-src-alpha'
}
```

这种模式最常配合 “premultiplied alpha” 使用，
也就是说它假定 “src” 的 RGB 颜色
已经像前面讲过的那样，先乘过 alpha 值了。

下面我们做一个例子，来展示这些选项。

首先用 JavaScript 创建两张带 alpha 的 canvas 2D 图像。
然后把这两张 canvas 加载成 WebGPU 纹理。

先写一段代码，用来生成将作为 dst 纹理的图像。

```js
const hsl = (h, s, l) => `hsl(${h * 360 | 0}, ${s * 100}%, ${l * 100 | 0}%)`;

function createDestinationImage(size) {
  const canvas = document.createElement('canvas');
  canvas.width = size;
  canvas.height = size;
  const ctx = canvas.getContext('2d');

  const gradient = ctx.createLinearGradient(0, 0, size, size);
  for (let i = 0; i <= 6; ++i) {
    gradient.addColorStop(i / 6, hsl(i / -6, 1, 0.5));
  }

  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, size, size);

  ctx.fillStyle = 'rgba(0, 0, 0, 255)';
  ctx.globalCompositeOperation = 'destination-out';
  ctx.rotate(Math.PI / -4);
  for (let i = 0; i < size * 2; i += 32) {
    ctx.fillRect(-size, i, size * 2, 16);
  }

  return canvas;
}
```

运行效果如下。

{{{example url="../webgpu-blend-dest-canvas.html"}}}

下面再写一段代码，用来生成将作为
src 纹理的图像。

```js
const hsla = (h, s, l, a) => `hsla(${h * 360 | 0}, ${s * 100}%, ${l * 100 | 0}%, ${a})`;

function createSourceImage(size) {
  const canvas = document.createElement('canvas');
  canvas.width = size;
  canvas.height = size;
  const ctx = canvas.getContext('2d');
  ctx.translate(size / 2, size / 2);

  ctx.globalCompositeOperation = 'screen';
  const numCircles = 3;
  for (let i = 0; i < numCircles; ++i) {
    ctx.rotate(Math.PI * 2 / numCircles);
    ctx.save();
    ctx.translate(size / 6, 0);
    ctx.beginPath();

    const radius = size / 3;
    ctx.arc(0, 0, radius, 0, Math.PI * 2);

    const gradient = ctx.createRadialGradient(0, 0, radius / 2, 0, 0, radius);
    const h = i / numCircles;
    gradient.addColorStop(0.5, hsla(h, 1, 0.5, 1));
    gradient.addColorStop(1, hsla(h, 1, 0.5, 0));

    ctx.fillStyle = gradient;
    ctx.fill();
    ctx.restore();
  }
  return canvas;
}
```

运行效果如下。

{{{example url="../webgpu-blend-src-canvas.html"}}}

现在我们两张图都有了，
接下来可以基于
[导入纹理那篇文章](webgpu-import-textures.html#a-loading-canvas)里的 canvas 导入示例
来修改。

首先，创建这两张 canvas 图像

```js
const size = 300;
const srcCanvas = createSourceImage(size);
const dstCanvas = createDestinationImage(size);
```

再修改一下着色器，
不再把纹理坐标乘以 50，
因为这次我们不会把一个长平面画到远处。

```wgsl
@vertex fn vs(
  @builtin(vertex_index) vertexIndex : u32
) -> OurVertexShaderOutput {
  let pos = array(
    // 1st triangle
    vec2f( 0.0,  0.0),  // center
    vec2f( 1.0,  0.0),  // right, center
    vec2f( 0.0,  1.0),  // center, top

    // 2nd triangle
    vec2f( 0.0,  1.0),  // center, top
    vec2f( 1.0,  0.0),  // right, center
    vec2f( 1.0,  1.0),  // right, top
  );

  var vsOutput: OurVertexShaderOutput;
  let xy = pos[vertexIndex];
  vsOutput.position = uni.matrix * vec4f(xy, 0.0, 1.0);
-  vsOutput.texcoord = xy * vec2f(1, 50);
+  vsOutput.texcoord = xy;
  return vsOutput;
}
```

把 `createTextureFromSource` 函数更新一下，
让它可以接收 `premultipliedAlpha: true/false`，
并把这个参数继续传给 `copyExternalTextureToImage`。

```js
-  function copySourceToTexture(device, texture, source, {flipY} = {}) {
+  function copySourceToTexture(device, texture, source, {flipY, premultipliedAlpha} = {}) {
    device.queue.copyExternalImageToTexture(
      { source, flipY, },
-      { texture },
+      { texture, premultipliedAlpha },
      { width: source.width, height: source.height },
    );

    if (texture.mipLevelCount > 1) {
      generateMips(device, texture);
    }
  }
```

然后，用它为每张纹理各创建两个版本：
一个是 premultiplied 的，
另一个是 “un-premultiplied” 或者说“未 premultiply”的。

```js
  const srcTextureUnpremultipliedAlpha =
      createTextureFromSource(
          device, srcCanvas,
          {mips: true});
  const dstTextureUnpremultipliedAlpha =
      createTextureFromSource(
          device, dstCanvas,
          {mips: true});

  const srcTexturePremultipliedAlpha =
      createTextureFromSource(
          device, srcCanvas,
          {mips: true, premultipliedAlpha: true});
  const dstTexturePremultipliedAlpha =
      createTextureFromSource(
          device, dstCanvas,
          {mips: true, premultipliedAlpha: true});
```

注意：我们当然也可以在着色器里加一个选项来做 premultiply，
但这并不是特别常见的做法。
更常见的是根据你的需求，
统一决定所有包含颜色的纹理到底是 premultiplied
还是 un-premultiplied。
所以这里我们保持成不同纹理版本，
然后通过 UI 选择使用 premultiplied 的纹理还是未 premultiply 的纹理。

我们需要为两次 draw
各准备一个 uniform buffer，
以防我们想把它们画在不同位置，
或者两张纹理大小不同。

```js
  function makeUniformBufferAndValues(device) {
    // offsets to the various uniform values in float32 indices
    const kMatrixOffset = 0;

    // create a buffer for the uniform values
    const uniformBufferSize =
      16 * 4; // matrix is 16 32bit floats (4bytes each)
    const buffer = device.createBuffer({
      label: 'uniforms for quad',
      size: uniformBufferSize,
      usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
    });

    // create a typedarray to hold the values for the uniforms in JavaScript
    const values = new Float32Array(uniformBufferSize / 4);
    const matrix = values.subarray(kMatrixOffset, 16);
    return { buffer, values, matrix };
  }
  const srcUniform = makeUniformBufferAndValues(device);
  const dstUniform = makeUniformBufferAndValues(device);
```

我们还需要一个 sampler，
并且要为每张纹理分别创建 bindGroup。
这里会引出一个问题：
bindGroup 需要一个 bindGroup layout。
本网站上的大多数示例，
都是通过 `somePipeline.getBindGroupLayout(groupNumber)`
从 pipeline 上取 layout。
但在这个例子里，
我们要根据当前选择的 blend state 设置来动态创建 pipeline。
也就是说，在 render 阶段之前，
我们手上还没有 pipeline，
也就无法提前从它上面拿 bindGroupLayout。

一种办法是在 render 阶段再创建 bindGroup。
另一种办法是，我们自己手动创建 bindGroupLayout，
然后让所有 pipeline 都使用它。
这样一来，我们就可以在初始化时先把 bindGroup 创建好，
而且它们会和任何使用同一个 bindGroupLayout 的 pipeline 兼容。

关于如何创建 [bindGroupLayout](GPUBindGroupLayout)
和 [pipelineLayout](GPUPipelineLayout) 的细节，
在[另一篇文章](webgpu-bind-group-layouts.html)里有详细说明。
这里先直接给出与当前 shader module 对应的代码

```js
  const bindGroupLayout = device.createBindGroupLayout({
    entries: [
      { binding: 0, visibility: GPUShaderStage.FRAGMENT, sampler: { }, },
      { binding: 1, visibility: GPUShaderStage.FRAGMENT, texture: { } },
      { binding: 2, visibility: GPUShaderStage.VERTEX, buffer: { } },
    ],
  });

  const pipelineLayout = device.createPipelineLayout({
    bindGroupLayouts: [
      bindGroupLayout,
    ],
  });
```

有了 bindGroupLayout 之后，
我们就可以用它来创建 bindGroup。

```js
  const sampler = device.createSampler({
    magFilter: 'linear',
    minFilter: 'linear',
    mipmapFilter: 'linear',
  });


  const srcBindGroupUnpremultipliedAlpha = device.createBindGroup({
    layout: bindGroupLayout,
    entries: [
      { binding: 0, resource: sampler },
      { binding: 1, resource: srcTextureUnpremultipliedAlpha },
      { binding: 2, resource: { buffer: srcUniform.buffer }},
    ],
  });

  const dstBindGroupUnpremultipliedAlpha = device.createBindGroup({
    layout: bindGroupLayout,
    entries: [
      { binding: 0, resource: sampler },
      { binding: 1, resource: dstTextureUnpremultipliedAlpha },
      { binding: 2, resource: { buffer: dstUniform.buffer }},
    ],
  });

  const srcBindGroupPremultipliedAlpha = device.createBindGroup({
    layout: bindGroupLayout,
    entries: [
      { binding: 0, resource: sampler },
      { binding: 1, resource: srcTexturePremultipliedAlpha },
      { binding: 2, resource: { buffer: srcUniform.buffer }},
    ],
  });

  const dstBindGroupPremultipliedAlpha = device.createBindGroup({
    layout: bindGroupLayout,
    entries: [
      { binding: 0, resource: sampler },
      { binding: 1, resource: dstTexturePremultipliedAlpha },
      { binding: 2, resource: { buffer: dstUniform.buffer }},
    ],
  });
```

现在我们已经有了 bindGroup 和纹理，
接着把 premultiplied 版和 un-premultiplied 版纹理
分别整理成数组，
这样就可以很方便地在两套之间切换。

```js
  const textureSets = [
    {
      srcTexture: srcTexturePremultipliedAlpha,
      dstTexture: dstTexturePremultipliedAlpha,
      srcBindGroup: srcBindGroupPremultipliedAlpha,
      dstBindGroup: dstBindGroupPremultipliedAlpha,
    },
    {
      srcTexture: srcTextureUnpremultipliedAlpha,
      dstTexture: dstTextureUnpremultipliedAlpha,
      srcBindGroup: srcBindGroupUnpremultipliedAlpha,
      dstBindGroup: dstBindGroupUnpremultipliedAlpha,
    },
  ];
```

在 render pass descriptor 里，
我们把 `clearValue` 单独提出来，
这样后面访问起来更方便。

```js
+  const clearValue = [0, 0, 0, 0];
  const renderPassDescriptor = {
    label: 'our basic canvas renderPass',
    colorAttachments: [
      {
        // view: <- to be filled out when we render
-        clearValue: [0.3, 0.3, 0.3, 1];
+        clearValue,
        loadOp: 'clear',
        storeOp: 'store',
      },
    ],
  };
```

我们需要两个 render pipeline。
第一个用来绘制 dst 纹理，
它不需要 blending。
注意这里我们传入的是 `pipelineLayout`，
而不是像之前大多数示例那样使用 `auto`。

```js
  const dstPipeline = device.createRenderPipeline({
    label: 'hardcoded textured quad pipeline',
    layout: pipelineLayout,
    vertex: {
      module,
    },
    fragment: {
      module,
      targets: [ { format: presentationFormat } ],
    },
  });
```

第二个 pipeline 会在 render 阶段根据当前选择的 blend 参数来创建。

```js
  const color = {
    operation: 'add',
    srcFactor: 'one',
    dstFactor: 'one-minus-src',
  };

  const alpha = {
    operation: 'add',
    srcFactor: 'one',
    dstFactor: 'one-minus-src',
  };

  function render() {
    ...

    const srcPipeline = device.createRenderPipeline({
      label: 'hardcoded textured quad pipeline',
      layout: pipelineLayout,
      vertex: {
        module,
      },
      fragment: {
        module,
        targets: [
          {
            format: presentationFormat,
            blend: {
              color,
              alpha,
            },
          },
        ],
      },
    });

```

渲染时，我们先选定一组纹理，
然后先用 `dstPipeline`
绘制 dst 纹理（不做 blending），
再在它上面用 `srcPipeline`
绘制 src 纹理（启用 blending）。

```js
+  const settings = {
+    textureSet: 0,
+  };

  function render() {
    const srcPipeline = device.createRenderPipeline({
      label: 'hardcoded textured quad pipeline',
      layout: pipelineLayout,
      vertex: {
        module,
      },
      fragment: {
        module,
        targets: [
          {
            format: presentationFormat,
            blend: {
              color,
              alpha,
            },
          },
        ],
      },
    });

+    const {
+      srcTexture,
+      dstTexture,
+      srcBindGroup,
+      dstBindGroup,
+    } = textureSets[settings.textureSet];

    const canvasTexture = context.getCurrentTexture();
    // Get the current texture from the canvas context and
    // set it as the texture to render to.
    renderPassDescriptor.colorAttachments[0].view =
        canvasTexture.createView();

+    function updateUniforms(uniform, canvasTexture, texture) {
+      const projectionMatrix = mat4.ortho(0, canvasTexture.width, canvasTexture.height, 0, -1, 1);
+
+      mat4.scale(projectionMatrix, [texture.width, texture.height, 1], uniform.matrix);
+
+      // copy the values from JavaScript to the GPU
+      device.queue.writeBuffer(uniform.buffer, 0, uniform.values);
+    }
+    updateUniforms(srcUniform, canvasTexture, srcTexture);
+    updateUniforms(dstUniform, canvasTexture, dstTexture);

    const encoder = device.createCommandEncoder({ label: 'render with blending' });
    const pass = encoder.beginRenderPass(renderPassDescriptor);

+    // draw dst
+    pass.setPipeline(dstPipeline);
+    pass.setBindGroup(0, dstBindGroup);
+    pass.draw(6);  // call our vertex shader 6 times
+
+    // draw src
+    pass.setPipeline(srcPipeline);
+    pass.setBindGroup(0, srcBindGroup);
+    pass.draw(6);  // call our vertex shader 6 times

    pass.end();

    const commandBuffer = encoder.finish();
    device.queue.submit([commandBuffer]);
  }
```

现在再做一套 UI，
用来设置这些值。

```js
+  const operations = [
+    'add',
+    'subtract',
+    'reverse-subtract',
+    'min',
+    'max',
+  ];
+
+  const factors = [
+    'zero',
+    'one',
+    'src',
+    'one-minus-src',
+    'src-alpha',
+    'one-minus-src-alpha',
+    'dst',
+    'one-minus-dst',
+    'dst-alpha',
+    'one-minus-dst-alpha',
+    'src-alpha-saturated',
+    'constant',
+    'one-minus-constant',
+  ];

  const color = {
    operation: 'add',
    srcFactor: 'one',
    dstFactor: 'one-minus-src',
  };

  const alpha = {
    operation: 'add',
    srcFactor: 'one',
    dstFactor: 'one-minus-src',
  };

  const settings = {
    textureSet: 0,
  };

+  const gui = new GUI().onChange(render);
+  gui.add(settings, 'textureSet', ['premultiplied alpha', 'un-premultiplied alpha']);
+  const colorFolder = gui.addFolder('color');
+  colorFolder.add(color, 'operation', operations);
+  colorFolder.add(color, 'srcFactor', factors);
+  colorFolder.add(color, 'dstFactor', factors);
+  const alphaFolder = gui.addFolder('alpha');
+  alphaFolder.add(alpha, 'operation', operations);
+  alphaFolder.add(alpha, 'srcFactor', factors);
+  alphaFolder.add(alpha, 'dstFactor', factors);
```

如果 `operation` 是 `'min'`
或 `'max'`，
那就必须把 `srcFactor` 和 `dstFactor`
都设成 `'one'`，
否则会报错。

```js
+  function makeBlendComponentValid(blend) {
+    const { operation } = blend;
+    if (operation === 'min' || operation === 'max') {
+      blend.srcFactor = 'one';
+      blend.dstFactor = 'one';
+    }
+  }

  function render() {
+    makeBlendComponentValid(color);
+    makeBlendComponentValid(alpha);
+    gui.updateDisplay();

    ...
```

我们再加上对 blend constant 的设置能力，
这样当 factor 选到
`'constant'`
或 `'one-minus-constant'`
时就能看到效果。

```js
+  const constant = {
+    color: [1, 0.5, 0.25],
+    alpha: 1,
+  };

  const settings = {
    textureSet: 0,
  };

  const gui = new GUI().onChange(render);
  gui.add(settings, 'textureSet', ['premultiplied alpha', 'un-premultiplied alpha']);
  ...
+  const constantFolder = gui.addFolder('constant');
+  constantFolder.addColor(constant, 'color');
+  constantFolder.add(constant, 'alpha', 0, 1);

  ...

  function render() {
    ...

    const pass = encoder.beginRenderPass(renderPassDescriptor);

    // draw dst
    pass.setPipeline(dstPipeline);
    pass.setBindGroup(0, dstBindGroup);
    pass.draw(6);  // call our vertex shader 6 times

    // draw src
    pass.setPipeline(srcPipeline);
    pass.setBindGroup(0, srcBindGroup);
+    pass.setBlendConstant([...constant.color, constant.alpha]);
    pass.draw(6);  // call our vertex shader 6 times

    pass.end();
  }
```

由于可能的组合有
13 * 13 * 5 * 13 * 13 * 5
那么多，
根本不可能全部都挨个试，
所以我们再提供一组预设。
如果某个预设没有给出 `alpha` 设置，
那我们就重复使用 `color` 的设置。

```js
+  const presets = {
+    'default (copy)': {
+      color: {
+        operation: 'add',
+        srcFactor: 'one',
+        dstFactor: 'zero',
+      },
+    },
+    'premultiplied blend (source-over)': {
+      color: {
+        operation: 'add',
+        srcFactor: 'one',
+        dstFactor: 'one-minus-src-alpha',
+      },
+    },
+    'un-premultiplied blend': {
+      color: {
+        operation: 'add',
+        srcFactor: 'src-alpha',
+        dstFactor: 'one-minus-src-alpha',
+      },
+    },
+    'destination-over': {
+      color: {
+        operation: 'add',
+        srcFactor: 'one-minus-dst-alpha',
+        dstFactor: 'one',
+      },
+    },
+    'source-in': {
+      color: {
+        operation: 'add',
+        srcFactor: 'dst-alpha',
+        dstFactor: 'zero',
+      },
+    },
+    'destination-in': {
+      color: {
+        operation: 'add',
+        srcFactor: 'zero',
+        dstFactor: 'src-alpha',
+      },
+    },
+    'source-out': {
+      color: {
+        operation: 'add',
+        srcFactor: 'one-minus-dst-alpha',
+        dstFactor: 'zero',
+      },
+    },
+    'destination-out': {
+      color: {
+        operation: 'add',
+        srcFactor: 'zero',
+        dstFactor: 'one-minus-src-alpha',
+      },
+    },
+    'source-atop': {
+      color: {
+        operation: 'add',
+        srcFactor: 'dst-alpha',
+        dstFactor: 'one-minus-src-alpha',
+      },
+    },
+    'destination-atop': {
+      color: {
+        operation: 'add',
+        srcFactor: 'one-minus-dst-alpha',
+        dstFactor: 'src-alpha',
+      },
+    },
+    'additive (lighten)': {
+      color: {
+        operation: 'add',
+        srcFactor: 'one',
+        dstFactor: 'one',
+      },
+    },
+  };

  ...

  const settings = {
    textureSet: 0,
+    preset: 'default (copy)',
  };

  const gui = new GUI().onChange(render);
  gui.add(settings, 'textureSet', ['premultiplied alpha', 'un-premultiplied alpha']);
+  gui.add(settings, 'preset', Object.keys(presets))
+    .name('blending preset')
+    .onChange(presetName => {
+      const preset = presets[presetName];
+      Object.assign(color, preset.color);
+      Object.assign(alpha, preset.alpha || preset.color);
+      gui.updateDisplay();
+    });

  ...
```

再让你能够选择 canvas 配置里的 `alphaMode`。

```js
  const settings = {
+    alphaMode: 'premultiplied',
    textureSet: 0,
    preset: 'default (copy)',
  };

  const gui = new GUI().onChange(render);
+  gui.add(settings, 'alphaMode', ['opaque', 'premultiplied']).name('canvas alphaMode');
  gui.add(settings, 'textureSet', ['premultiplied alpha', 'un-premultiplied alpha']);

  ...

  function render() {
    ...

+    context.configure({
+      device,
+      format: presentationFormat,
+      alphaMode: settings.alphaMode,
+    });

    const canvasTexture = context.getCurrentTexture();
    // Get the current texture from the canvas context and
    // set it as the texture to render to.
    renderPassDescriptor.colorAttachments[0].view =
        canvasTexture.createView();

```

最后，再加上一个选项，
让你可以设置 render pass 的 clearValue。

```js
+  const clear = {
+    color: [0, 0, 0],
+    alpha: 0,
+    premultiply: true,
+  };

  const settings = {
    alphaMode: 'premultiplied',
    textureSet: 0,
    preset: 'default (copy)',
  };

  const gui = new GUI().onChange(render);

  ...

+  const clearFolder = gui.addFolder('clear color');
+  clearFolder.add(clear, 'premultiply');
+  clearFolder.add(clear, 'alpha', 0, 1);
+  clearFolder.addColor(clear, 'color');

  function render() {
    ...

    const canvasTexture = context.getCurrentTexture();
    // Get the current texture from the canvas context and
    // set it as the texture to render to.
    renderPassDescriptor.colorAttachments[0].view =
        canvasTexture.createView();

+    {
+      const { alpha, color, premultiply } = clear;
+      const mult = premultiply ? alpha : 1;
+      clearValue[0] = color[0] * mult;
+      clearValue[1] = color[1] * mult;
+      clearValue[2] = color[2] * mult;
+      clearValue[3] = alpha;
+    }
```

上面的选项确实很多，甚至可能多得有点过头了。
不过无论如何，
现在我们终于有了一个可以自由试验 blend settings 的例子。

{{{example url="../webgpu-blend.html"}}}

给定我们的源图像

<div class="webgpu_center">
  <div data-diagram="original"></div>
</div>

下面是一些已知比较有用的混合设置

<div class="webgpu_center">
  <div data-diagram="blend-premultiplied blend (source-over)"></div>
</div>

<div class="webgpu_center">
  <div data-diagram="blend-destination-over"></div>
</div>

<div class="webgpu_center">
  <div data-diagram="blend-additive (lighten)"></div>
</div>

<div class="webgpu_center">
  <div data-diagram="blend-source-in"></div>
</div>

<div class="webgpu_center">
  <div data-diagram="blend-destination-in"></div>
</div>

<div class="webgpu_center">
  <div data-diagram="blend-source-out"></div>
</div>

<div class="webgpu_center">
  <div data-diagram="blend-destination-out"></div>
</div>

<div class="webgpu_center">
  <div data-diagram="blend-source-atop"></div>
</div>

<div class="webgpu_center">
  <div data-diagram="blend-destination-atop"></div>
</div>

<hr>

这些 blend setting 的名字，
来自 Canvas 2D 的
[`globalCompositeOperation`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/globalCompositeOperation)
选项。
那个规范里还列出了更多模式，
但其中大多数都需要超出这些基础 blending 设置能力范围之外的额外数学运算，
因此往往需要采用不同的解决方案。

现在我们已经掌握了 WebGPU 中这些与 blending 相关的基础知识，
后面在讲各种技术时，
就可以直接引用它们了。

<!-- keep this at the bottom of the article -->
<link href="webgpu-transparency.css" rel="stylesheet">
<script type="module" src="webgpu-transparency.js"></script>
