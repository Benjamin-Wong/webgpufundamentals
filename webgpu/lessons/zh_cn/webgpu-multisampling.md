Title: WebGPU 多重采样
Description: Multisampling / MSAA
TOC: Multisampling / MSAA

MSAA 是 Multi-Sampling Anti-aliasing 的缩写，也就是多重采样抗锯齿。
抗锯齿的意思是，尽量避免 aliasing（锯齿）问题，
而 aliasing 指的就是当我们试图把矢量形状绘制成离散像素时，
出现的那种块状、锯齿状效果。

我们在[基础那篇文章](webgpu-fundamentals.html)里已经展示过 WebGPU 是如何绘制图形的。
它会取顶点着色器中返回给 `@builtin(position)` 的裁剪空间顶点，
每 3 个顶点组成一个三角形，然后对于落在这个三角形内部、且位于像素中心的每个像素，
调用片元着色器，询问这个像素应该是什么颜色。

<div class="webgpu_center side-by-side flex-gap" style="max-width: 850px">
  <div class="multisample-example">
    <div data-diagram="clip-space-to-texels"></div>
    <div>拖动顶点</div>
  </div>
  <div class="multisample-example">
    <div data-diagram="clip-space-to-texels-result"></div>
    <div>结果</div>
  </div>
</div>

上面的三角形看起来非常锯齿化。
我们当然可以提高分辨率，但最终能显示的最高分辨率
仍然受限于显示器本身，而这可能依旧不足以让它看起来不那么块状。

一种解决方案是以更高分辨率进行渲染。
例如，我们把分辨率提高到 4 倍
（宽和高都提升 2 倍），然后再把结果通过“双线性过滤”
缩回到 canvas 上。
我们在[纹理那篇文章](webgpu-textures.html)里已经讲过
“双线性过滤”。

<div class="webgpu_center side-by-side flex-gap" style="max-width: 850px">
  <div class="multisample-example">
    <div data-diagram="clip-space-to-texels-4x"></div>
    <div>4 倍分辨率</div>
  </div>
  <div class="multisample-example">
    <div data-diagram="clip-space-to-texels-4x-result"></div>
    <div>双线性过滤后的结果</div>
  </div>
</div>

这种方案是有效的，但也很浪费。
左边图像里每 2x2 的 4 个像素，最后会变成右边图像里的 1 个像素，
但很多时候，这 4 个像素其实全部都在三角形内部，
因此根本不需要抗锯齿。因为这 4 个像素全都是红色。

<div class="webgpu_center side-by-side flex-gap">
  <div class="multisample-example">
    <div data-diagram="clip-space-to-texels-4x-waste"></div>
    <div>每 4 个<span style="color: cyan;">青色</span>像素里有 3 个都被浪费了</div>
  </div>
</div>

绘制 4 个红色像素而不是 1 个像素，本身就是时间浪费。
GPU 为此调用了 4 次片元着色器。
片元着色器往往可能很大、工作量也很多，
所以我们希望尽量少调用它。即使三角形只跨过 3 个像素，
结果也会像这样

<div class="webgpu_center">
  <img src="resources/antialias-4x.svg" width="600">
</div>

上图中，使用 4 倍分辨率渲染时，三角形覆盖了 3 个像素中心，
因此片元着色器会被调用 3 次。
之后我们再对结果做双线性过滤。

而 multisampling 更高效的地方就在这里。
我们会创建一种特殊的“多重采样纹理”。
当我们向一张多重采样纹理绘制三角形时，只要 4 个 *sample*
里有任意一个落在三角形内部，GPU 就只调用一次片元着色器，
然后只把结果写入那些真正位于三角形内部的 *sample*。

<div class="webgpu_center">
  <img src="resources/antialias-multisample-4.svg" width="600">
</div>

上图中，使用多重采样渲染时，三角形覆盖了 3 个 *sample*，
但片元着色器只被调用了 1 次。
之后我们再对结果进行 *resolve*。
如果三角形覆盖了全部 4 个 sample point，
过程也是类似的：片元着色器仍然只调用一次，
只是它的结果会被写入全部 4 个 sample。

注意，这和前面“4 倍分辨率渲染”的方案不一样。
在 4 倍分辨率渲染中，CPU 检查的是 4 个像素的中心点是否位于三角形内部；
而在多重采样渲染中，GPU 检查的是“sample 位置”，
这些位置并不是规则网格上的点。
同样，这些 sample 的值本身也不代表一个规则网格，
因此“resolve”的过程也不是双线性过滤，
而是交由 GPU 自己决定。
这些不位于中心的 sample 位置，在大多数情况下会带来更好的抗锯齿效果。

## <a id="a-multisampling"></a>如何使用多重采样

那么我们该如何使用多重采样呢？
基本上分成 3 步

1. 把 pipeline 设置为渲染到一张多重采样纹理
2. 创建一张与最终纹理大小相同的多重采样纹理
3. 把 render pass 设置为先渲染到多重采样纹理，再 *resolve* 到最终纹理（也就是我们的 canvas）

为了简单起见，我们拿[基础那篇文章](webgpu-fundamentals.html#a-resizing)结尾那个可响应尺寸变化的三角形示例，
给它加上多重采样。

### 把 pipeline 设置为渲染到一张多重采样纹理

```js
  const pipeline = device.createRenderPipeline({
    label: 'our hardcoded red triangle pipeline',
    layout: 'auto',
    vertex: {
      module,
    },
    fragment: {
      module,
      targets: [{ format: presentationFormat }],
    },
+    multisample: {
+      count: 4,
+    },
  });
```

上面添加的 `multisample` 设置，
使这个 pipeline 可以渲染到一张多重采样纹理。

### 创建一张与最终纹理大小相同的多重采样纹理

我们的最终纹理就是 canvas 对应的纹理。
由于 canvas 的尺寸可能发生变化，
比如用户调整了窗口大小，
因此我们会在渲染时创建这张纹理。

```js
+  let multisampleTexture;

  function render() {
+    // Get the current texture from the canvas context
+    const canvasTexture = context.getCurrentTexture();
+
+    // If the multisample texture doesn't exist or
+    // is the wrong size then make a new one.
+    if (!multisampleTexture ||
+        multisampleTexture.width !== canvasTexture.width ||
+        multisampleTexture.height !== canvasTexture.height) {
+
+      // If we have an existing multisample texture destroy it.
+      if (multisampleTexture) {
+        multisampleTexture.destroy();
+      }
+
+      // Create a new multisample texture that matches our
+      // canvas's size
+      multisampleTexture = device.createTexture({
+        format: canvasTexture.format,
+        usage: GPUTextureUsage.RENDER_ATTACHMENT,
+        size: [canvasTexture.width, canvasTexture.height],
*        sampleCount: 4,
+      });
+    }

  ...
```

上面的代码会在以下两种情况下创建一张新的多重采样纹理：
（a）我们还没有这张纹理，
或（b）已有纹理的尺寸和 canvas 不匹配。
这张纹理的大小与 canvas 一样，
但我们额外指定了 `sampleCount: 4`，
这样它就成了一张多重采样纹理。

### 把 render pass 设置为渲染到多重采样纹理，并 *resolve* 到最终纹理（我们的 canvas）

```js
-    // Get the current texture from the canvas context and
-    // set it as the texture to render to.
-    renderPassDescriptor.colorAttachments[0].view =
-        context.getCurrentTexture().createView();

+    // Set the multisample texture as the texture to render to
+    renderPassDescriptor.colorAttachments[0].view =
+        multisampleTexture.createView();
+    // Set the canvas texture as the texture to "resolve"
+    // the multisample texture to.
+    renderPassDescriptor.colorAttachments[0].resolveTarget =
+        canvasTexture.createView();
```

*Resolving* 指的是把多重采样纹理的结果转换成
我们真正需要的那个尺寸的纹理。
这里就是 canvas。
前面“4 倍分辨率渲染”的方案中，我们是手动对 4x 的纹理做双线性过滤，
把它缩回到 1x 纹理。
这里过程类似，但对多重采样纹理来说，
它并不是真正意义上的双线性过滤。
[见下文](#a-not-a-grid)

效果如下

{{{example url="../webgpu-multisample-simple.html"}}}

肉眼上并没有特别多可以直接看出来的地方，
但如果我们在低分辨率下把两者并排比较：
左边是没有使用多重采样的原始版本，
右边是使用了多重采样的版本，
就能看出右边已经做了抗锯齿。

<div class="webgpu_center side-by-side flex-gap" style="max-width: 850px">
  <div class="multisample-example">
    <div data-diagram="simple-triangle"></div>
    <div>原始版本</div>
  </div>
  <div class="multisample-example">
    <div data-diagram="simple-triangle-multisample"></div>
    <div>启用多重采样</div>
  </div>
</div>

有几点需要注意：

## `count` 必须是 `4`

在 WebGPU v1 中，你只能把 render pipeline 里的
`multisample: { count }`
设置成 4 或 1。
同样，纹理的 `sampleCount` 也只能设成 4 或 1。
其中 1 是默认值，表示这张纹理不是多重采样纹理。

## <a id="a-not-a-grid"></a>多重采样并不使用规则网格

正如前面提到的，多重采样并不是发生在规则网格上的。
对于 `sampleCount = 4`，
sample 的位置大致如下。

<div class="webgpu_center">
  <img src="resources/multisample-4x.svg" width="256">
  <div class="center">count: 4</div>
</div>

<div class="webgpu_center">
  <img src="resources/multisample-2x.svg" width="256">
  <div class="center">count: 2</div>
</div>

<div class="webgpu_center">
  <img src="resources/multisample-8x.svg" width="256">
  <div class="center">count: 8</div>
</div>

<div class="webgpu_center">
  <img src="resources/multisample-16x.svg" width="256">
  <div class="center">count: 16</div>
</div>

**但 WebGPU 当前只支持 count = 4**

## 你不必在每个 render pass 上都设置 resolve target

设置 `colorAttachment[0].resolveTarget`
等于告诉 WebGPU：
“当这个 render pass 中的所有绘制都完成后，
请把多重采样纹理下采样到 `resolveTarget` 指定的纹理中。”
如果你有多个 render pass，
通常只会希望在最后一个 pass 才执行 resolve。
虽然在最后一个 pass 里 resolve 通常最快，
但额外做一个空的最后 pass、专门只负责 resolve，
也是完全没问题的。
只要记得除了第一个 pass 之外，
后续 pass 的 `loadOp` 要设成 `'load'`，
而不是 `'clear'`，否则内容会被清掉。

## 你也可以选择让片元着色器对每个 sample point 都运行一次

前面我们说过，对于多重采样纹理里的每 4 个 sample，
片元着色器默认只运行一次。
它运行一次之后，把结果写入那些真正位于三角形内部的 sample。
这就是为什么它比以 4 倍分辨率渲染更快。

在[跨阶段变量那篇文章](webgpu-inter-stage-variables.html#a-interpolate)中，
我们提到过可以通过 `@interpolate(...)` 属性
来标记跨阶段变量的插值方式。
其中一个选项就是 `sample`，
在这种情况下，片元着色器会对每个 sample 分别运行一次。
还有一些内建值，比如 `@builtin(sample_index)`，
可以告诉你当前处理的是哪个 sample；
以及 `@builtin(sample_mask)`，
它作为输入时可以告诉你哪些 sample 在三角形内部，
作为输出时则允许你阻止某些 sample point 被更新。

## `center` 与 `centroid`

有 3 种与 *sampling* 相关的插值模式。
上面提到过 `'sample'` 模式，
即片元着色器会为每个 sample 分别执行一次。
另外两种则是
`'center'`（默认值）
和 `'centroid'`。

* `'center'` 会以像素中心为基准来插值。

<div class="webgpu_center">
  <img src="resources/multisample-centroid-issue.svg" width="400">
</div>

上图展示了一个单独的像素 / texel，
其中 sample point `s1` 和 `s3` 位于三角形内部。
此时片元着色器只会被调用一次，
并收到一组以像素中心 (`c`) 为基准插值得到的跨阶段变量值。
问题就在于，**`c` 实际上位于三角形外部**。

这有时可能没关系，
但也可能你的某些计算会默认这些值一定在三角形内部。
我一时想不到特别好的现实例子，
但假设我们给每个顶点加上重心坐标（barycentric coordinates）。
重心坐标本质上是 3 个坐标值，
每个值都在 0 到 1 之间，
并表示某个位置距离三角形某个顶点的相对程度。
要做到这一点，我们只需要这样添加 barycentric 点

```wgsl
+struct VOut {
+  @builtin(position) position: vec4f,
+  @location(0) baryCoord: vec3f,
+};

@vertex fn vs(
  @builtin(vertex_index) vertexIndex : u32
-) -> @builtin(position) vec4f {
+) -> VOut {
  let pos = array(
    vec2f( 0.0,  0.5),  // top center
    vec2f(-0.5, -0.5),  // bottom left
    vec2f( 0.5, -0.5)   // bottom right
  );
+  let bary = array(
+    vec3f(1, 0, 0),
+    vec3f(0, 1, 0),
+    vec3f(0, 0, 1),
+  );
-    return vec4f(pos[vertexIndex], 0.0, 1.0);
+  var vout: VOut;
+  vout.position = vec4f(pos[vertexIndex], 0.0, 1.0);
+  vout.baryCoord = bary[vertexIndex];
+  return vout;
}

-@fragment fn fs() -> @location(0) vec4f {
-  return vec4f(1, 0, 0, 1);
+@fragment fn fs(vin: VOut) -> @location(0) vec4f {
+  let allAbove0 = all(vin.baryCoord >= vec3f(0));
+  let allBelow1 = all(vin.baryCoord <= vec3f(1));
+  let inside = allAbove0 && allBelow1;
+  let red = vec4f(1, 0, 0, 1);
+  let yellow = vec4f(1, 1, 0, 1);
+  return select(yellow, red, inside);
}
```

上面我们把 `1, 0, 0` 关联给第一个点，
把 `0, 1, 0` 关联给第二个点，
把 `0, 0, 1` 关联给第三个点。
在它们之间进行插值时，
理论上任何值都不应该小于 0 或大于 1。

在片元着色器中，我们先用
`all(vin.baryCoord >= vec3f(0))`
检查这 3 个插值后的值（x、y、z）是否都 `>= 0`。
再用
`all(vin.baryCoord <= vec3f(1))`
检查它们是否都 `<= 1`。
最后把这两个结果用 `&` 合起来。
这样就能判断当前点是在三角形内部还是外部。
如果在内部，最终颜色选红色；否则选黄色。
按理说，因为我们是在顶点之间做插值，
结果应该始终位于三角形内部才对。

为了更容易观察结果，
我们还可以把示例改成更低分辨率。

```js
  const observer = new ResizeObserver(entries => {
    for (const entry of entries) {
      const canvas = entry.target;
-      const width = entry.contentBoxSize[0].inlineSize;
-      const height = entry.contentBoxSize[0].blockSize;
+      const width = entry.contentBoxSize[0].inlineSize / 16 | 0;
+      const height = entry.contentBoxSize[0].blockSize / 16 | 0;
      canvas.width = Math.max(1, Math.min(width, device.limits.maxTextureDimension2D));
      canvas.height = Math.max(1, Math.min(height, device.limits.maxTextureDimension2D));
      // re-render
      render();
    }
  });
  observer.observe(canvas);
```

再加一点 CSS

```js
canvas {
+  image-rendering: pixelated;
+  image-rendering: crisp-edges;
  display: block;  /* make the canvas act like a block   */
  width: 100%;     /* make the canvas fill its container */
  height: 100%;
}
```

运行后就会看到这个结果

{{{example url="../webgpu-multisample-center-issue.html"}}}

可以看到，边缘的一些像素里带有黄色。
这正是因为，正如前面指出的，
传给片元着色器的跨阶段变量插值结果
是以像素中心为基准计算的。
而在这些出现黄色的情况下，
这个中心点实际上位于三角形外面。

把插值采样模式切换成 `'centroid'`
可以尝试修复这个问题。
在 `'centroid'` 模式下，GPU 会使用当前像素内部、
且落在三角形区域内部分的几何中心来进行插值。

<div class="webgpu_center">
  <img src="resources/multisample-centroid-fix.svg" width="400">
</div>


如果我们把示例里的插值模式改成 `'centroid'`

```wgsl
struct VOut {
  @builtin(position) position: vec4f,
-  @location(0) baryCoord: vec3f,
+  @location(0) @interpolate(perspective, centroid) baryCoord: vec3f,
};
```

现在 GPU 就会传入相对于 centroid 进行插值后的跨阶段变量值，
那些黄色像素的问题也就消失了。

{{{example url="../webgpu-multisample-centroid.html"}}}

> 注意：GPU 可能真的会去计算像素内三角形覆盖区域的 centroid，
> 也可能不会。
> 唯一有保证的是，
> 跨阶段变量会相对于某个落在像素与三角形相交区域内部的位置来插值。

## 那三角形内部的抗锯齿怎么办？

多重采样通常只对三角形边缘有帮助。
因为它只调用一次片元着色器，
所以当所有 sample 位置都位于三角形内部时，
我们只会得到同一个片元着色器结果被写入所有 sample，
这意味着最终结果和不开启多重采样时不会有什么区别。

在上面的示例中，因为我们画的是纯红色，
显然这没有任何问题。
那如果我们是在三角形内部采样一张纹理呢？
纹理内部可能会有高对比度颜色彼此相邻。
难道我们不希望每个 sample 的颜色
都来自纹理中的不同位置吗？

对于三角形内部，我们通常依靠
[mipmap 和过滤](webgpu-textures.html)
来选出合适的颜色，
因此三角形内部的抗锯齿可能就没那么重要。
另一方面，这在某些渲染技术里确实也会成为问题，
这也是为什么除了多重采样之外，还存在其他抗锯齿方案；
也可能正因此，
如果你想做逐 sample 处理，
可以使用 `@interpolate(..., sample)`。

## 多重采样并不是唯一的抗锯齿方案

这一页里我们提到了 2 种方案。
1. 先渲染到更高分辨率的纹理，再以较低分辨率显示该纹理。
2. 使用多重采样。
当然，还有很多其他方法。
[这里有一篇介绍其中几种方案的文章](https://vr.arvilab.com/blog/anti-aliasing)。

其他一些参考资源：

* [A Quick overview of MSAA](https://therealmjp.github.io/posts/msaa-overview/)
* [Multisampling primer](https://www.rastergrid.com/blog/gpu-tech/2021/10/multisampling-primer/)

<!-- keep this at the bottom of the article -->
<link href="webgpu-multisampling.css" rel="stylesheet">
<script type="module" src="webgpu-multisampling.js"></script>
