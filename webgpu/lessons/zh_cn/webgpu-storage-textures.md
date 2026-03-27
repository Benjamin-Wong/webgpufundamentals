Title: WebGPU Storage Texture
Description: 如何使用 Storage Texture
TOC: Storage Texture

Storage texture 本质上就是一种[纹理](webgpu-textures.html)，
只不过你可以直接往里面写，也就是“存储”数据。
通常情况下，我们是在顶点着色器里指定三角形，
由 GPU 间接帮我们更新纹理；而使用 storage texture 时，
我们可以在任何想写的位置直接写入纹理。

Storage texture 并不是某种特殊的纹理类型，
它仍然只是通过 `createTexture` 创建出来的一张普通纹理。
你只需要额外加上 `STORAGE_BINDING` 这个 usage 标志，
这样这张纹理除了具备你需要的其他用途之外，
也能作为 storage texture 来使用。

从某种意义上说，storage texture 很像一个被当成 2D 数组使用的 storage buffer。
例如，我们可以创建一个 storage buffer，并在代码中像这样引用它

```wgsl
@group(0) @binding(0)
  var<storage> buf: array<f32>;

...
fn loadValueFromBuffer(pos: vec2u) -> f32 {
  return buffer[pos.y * width + pos.x];
}

fn storeValueToBuffer(pos: vec2u, v: f32) {
  buffer[pos.y * width + pos.x] = v;
}

...
  let pos = vec2u(2, 3);
  var v = loadValueFromBuffer(pos);
  storeValueToBuffer(pos, v * 2.0);

```

而 storage texture 则是这样

```
@group(0) @binding(0)
  var tex: texture_storage_2d<r32float, read_write>;

...

   let pos = vec2u(2, 3);
   let mipLevel = 0;
   var v = textureLoad(tex, pos, mipLevel);
   textureStore(tex, pos, mipLevel, v * 2);

```

既然看起来这两者很像，那手动使用 storage buffer
和使用 storage texture 的区别到底是什么呢？

* Storage texture 依然是一张纹理。

  你可以在一个着色器里把它当作 storage texture 使用，
  在另一个着色器里又把它当成普通纹理来使用
  （带 sampler、mipmap 等等）。

* Storage texture 带有格式解释能力，而 storage buffer 没有。

  例如：

  ```wsgl
  @group(0) @binding(0) var tex: texture_storage_2d<rgba8unorm, read>;
  @group(0) @binding(1) var buf: array<f32>;

     ...
      let t = textureLoad(tex, pos, 0);
      let b = buffer[pos.y * bufferWidth + pos.x];
  ```

  上面在调用 `textureLoad` 时，纹理的格式是 `rgba8unorm`，
  这意味着会读取 4 个字节，并自动转换成 4 个
  0 到 1 之间的浮点值，最后以 `vec4f` 返回。

  而在 buffer 的情况下，读取的 4 个字节会被当作一个单独的 `f32` 值。
  我们当然也可以把 buffer 改成 `array<u32>`，
  先读出一个值，再手动把它拆成 4 个字节值，
  然后再自己把它们转换成浮点数。但如果这正是我们想要的行为，
  用 storage texture 就可以免费得到。

* Storage texture 具有维度信息

  对于 buffer 来说，它唯一的维度就是长度，
  更准确地说，是它的绑定长度 [^binding]。上面的例子里，
  当我们把一个 buffer 当成 2D 数组使用时，
  必须依赖 `width` 才能把 2D 坐标换算成 1D 的 buffer 索引。
  这个 `width` 要么需要硬编码，
  要么需要通过某种方式传进来[^how-to-pass-data]。而对于纹理，
  我们可以直接调用 `textureDimensions`
  来获取纹理尺寸。

  [^binding]: 当你创建 bind group 并指定 buffer 时，
  可以额外指定 offset 和 length。
  在着色器里，数组长度是根据这个 binding 的长度计算出来的，
  而不是根据整个 buffer 的长度。
  如果你没有指定 offset，它默认是 0；
  length 默认则是整个 buffer 的大小。

  [^how-to-pass-data]: 你可以通过 [uniform](webgpu-uniforms.html)、
  另一个 [storage buffer](webgpu-storage-buffers.html)，
  甚至放在同一个 buffer 的第一个值里，把 buffer 的宽度传进来。

不过，storage texture 也有一些限制。

* 只有某些格式可以是 `read_write`

  这些格式是 `r32float`、`r32sint` 和 `r32uint`。

  其他支持的格式，在单个着色器中只能是 `read`
  或 `write` 之一。

* 只有某些格式可以被用作 storage texture

  纹理格式有很多种，但只有其中某些
  能作为 storage texture 使用。

  * `rgba8(unorm/snorm/sint/uint)`
  * `rgba16(float/sint/uint)`
  * `rg32(float/sint/uint)`
  * `rgba32(float/sint/uint)`

  这里缺少了一个值得注意的格式：`bgra8unorm`，
  我们稍后会讲到。

* Storage texture 不能使用 sampler

  如果一张纹理被当作普通的 `TEXTURE_BINDING` 使用，
  我们就可以调用 `textureSample` 之类的函数，
  让它在不同 mip 级别之间最多读取 16 个 texel，
  然后进行混合。
  但如果一张纹理被当作 `STORAGE_BINDING` 使用，
  我们就只能调用 `textureLoad` 和/或 `textureStore`，
  一次只读写一个 texel。

## <a id="canvas-as-storage-texture"></a>把 Canvas 当作 Storage Texture

你可以把 canvas 的纹理当作 storage texture 使用。
要做到这一点，需要在配置 context 时，
让它返回一张可以作为 storage texture 使用的纹理。

```js
  const presentationFormat = navigator.gpu.getPreferredCanvasFormat()
  context.configure({
    device,
    format: presentationFormat,
+    usage: GPUTextureUsage.TEXTURE_BINDING |
+           GPUTextureUsage.STORAGE_BINDING,
  });
```

这里 `TEXTURE_BINDING` 是必需的，
因为浏览器本身要把这张纹理显示到页面上。
而 `STORAGE_BINDING` 则允许我们把 canvas 的纹理
当作 storage texture 使用。
如果我们还想像这个网站上大多数示例那样，
通过 render pass 向这张纹理渲染，
那还需要再加上 `RENDER_ATTACHMENT` 这个 usage。

不过这里有一个小麻烦。正如我们在
[第一篇文章](webgpu-fundamentals.html)里提到的，
通常我们会调用 `navigator.gpu.getPreferredCanvasFormat`
来获取推荐的 canvas 格式。
`getPreferredCanvasFormat` 会根据用户系统上性能更好的格式，
返回 `rgba8unorm` 或 `bgra8unorm` 其中之一。

但正如前面提到的，默认情况下，
我们不能把 `bgra8unorm`
纹理当作 storage texture 使用。

好在有一个[特性](webgpu-limits-and-features.html)
叫做 `'bgra8unorm-storage'`。启用这个特性后，
就可以把 `bgra8unorm` 纹理作为 storage texture 使用。
通常来说，凡是把 `bgra8unorm` 报告为 preferred canvas format 的平台，
理论上*都应该*支持这个特性，
但仍然存在少数不支持的可能。
因此，我们需要先检查 `'bgra8unorm-storage'`
这个 *feature* 是否存在。如果存在，我们就在请求 device 时启用它，
并使用系统推荐的 canvas format。
如果不存在，那我们就选择 `rgba8unorm`
作为 canvas format。

```js
  const adapter = await navigator.gpu?.requestAdapter();
-  const device = await adapter?.requestDevice();
+  const hasBGRA8unormStorage = adapter.features.has('bgra8unorm-storage');
+  const device = await adapter?.requestDevice({
+    requiredFeatures: hasBGRA8unormStorage
+      ? ['bgra8unorm-storage']
+      : [],
+  });
  if (!device) {
    fail('need a browser that supports WebGPU');
    return;
  }

  // Get a WebGPU context from the canvas and configure it
  const canvas = document.querySelector('canvas');
  const context = canvas.getContext('webgpu');
-  const presentationFormat = navigator.gpu.getPreferredCanvasFormat()
+  const presentationFormat = hasBGRA8unormStorage
+     ? navigator.gpu.getPreferredCanvasFormat()
+     : 'rgba8unorm';
  context.configure({
    device,
    format: presentationFormat,
    usage: GPUTextureUsage.TEXTURE_BINDING |
           GPUTextureUsage.STORAGE_BINDING,
  });
```

这样我们就可以把 canvas 纹理当作 storage texture 使用了。
下面我们写一个简单的 compute shader，
在这张纹理上画同心圆。

```js
  const module = device.createShaderModule({
    label: 'circles in storage texture',
    code: /* wgsl */ `
      @group(0) @binding(0)
      var tex: texture_storage_2d<${presentationFormat}, write>;

      @compute @workgroup_size(1) fn cs(
        @builtin(global_invocation_id) id : vec3u
      )  {
        let size = textureDimensions(tex);
        let center = vec2f(size) / 2.0;

        // the pixel we're going to write to
        let pos = id.xy;

        // The distance from the center of the texture
        let dist = distance(vec2f(pos), center);

        // Compute stripes based on the distance
        let stripe = dist / 32.0 % 2.0;
        let red = vec4f(1, 0, 0, 1);
        let cyan = vec4f(0, 1, 1, 1);
        let color = select(red, cyan, stripe < 1.0);

        // Write the color to the texture
        textureStore(tex, pos, color);
      }
    `,
  });
```

注意这里我们把 storage texture 标记成了 `write`，
并且必须在着色器里显式写出纹理的具体格式。
和 `TEXTURE_BINDING` 不同，
`STORAGE_BINDING` 必须知道纹理的精确格式。

这套设置方式和我们在[第一篇文章](webgpu-fundamentals.html#a-run-computations-on-the-gpu)
里写的 compute shader 很像。
创建完 shader module 之后，
我们再创建一个 compute pipeline 来使用它。

```js
  const pipeline = device.createComputePipeline({
    label: 'circles in storage texture',
    layout: 'auto',
    compute: {
      module,
    },
  });
```

渲染时，我们先获取 canvas 当前对应的纹理，
创建 bind group 把纹理传给着色器，
然后执行常规流程：设置 pipeline、
绑定 bind group、派发 workgroup。

```js
  function render() {
    const texture = context.getCurrentTexture();

    const bindGroup = device.createBindGroup({
      layout: pipeline.getBindGroupLayout(0),
      entries: [
        { binding: 0, resource: texture },
      ],
    });

    const encoder = device.createCommandEncoder({ label: 'our encoder' });
    const pass = encoder.beginComputePass();
    pass.setPipeline(pipeline);
    pass.setBindGroup(0, bindGroup);
    pass.dispatchWorkgroups(texture.width, texture.height);
    pass.end();

    const commandBuffer = encoder.finish();
    device.queue.submit([commandBuffer]);
  }
```

效果如下

{{{example url="../webgpu-storage-texture-canvas.html"}}}

如果使用一张普通纹理，改动其实也不大，
只是我们不再调用 `getCurrentTexture`，
而是改成调用 `createTexture` 来创建纹理，
并传入 `STORAGE_BINDING`
以及你需要的其他 usage 标志即可。

## 速度与数据竞争

上面的示例中，我们为每个像素派发了 1 个 workgroup。
这其实是很浪费的，GPU 本可以运行得更快。
如果要把这个着色器优化到更合适的工作粒度，
示例代码就会复杂不少。
而这里的重点是演示如何使用 storage texture，
不是写出最快的着色器。
如果你想了解一些优化
compute shader 的方法，可以继续阅读
[计算图像直方图那篇文章](webgpu-compute-shaders-histogram.html)。

同样地，由于你可以往 storage texture 的任意位置写，
因此你也要像我们在
[其他 compute shader 相关文章](webgpu-compute-shaders.html)
里提到的那样，注意竞争条件。
各个 invocation 的执行顺序并没有保证。
你需要自己避免竞争，
或者插入 `textureBarriers` 等其他机制，
确保两个或更多 invocation
不会互相踩到对方的数据。

## 示例

[compute.toys](https://compute.toys) 是一个网站，
上面有很多直接写 storage texture 的示例。
**注意**：虽然你能从 [compute.toys](https://compute.toys)
上的例子里学到不少东西，
但它们不一定代表最佳实践。
Compute toys 的目标，是只用 compute shader 做出有趣的效果。
它更像是一种有趣的谜题：思考如何只靠 compute shader
完成某种创意效果。
但请记住，其他方法
*可能* 会快上几十倍、几百倍，甚至上千倍。
