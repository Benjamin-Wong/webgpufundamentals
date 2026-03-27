Title: WebGPU 将图像加载到纹理中
Description: 如何将图片、Canvas、视频加载到纹理
TOC: 加载图像

我们已经在[上一篇文章](webgpu-textures.html)中介绍了纹理的一些基础知识。
这篇文章会讲解如何把图像加载进纹理，
以及如何在 GPU 上生成 mipmap。

在上一篇文章中，我们通过调用 `device.createTexture` 创建纹理，
然后再调用 `device.queue.writeTexture` 把数据写入纹理。`device.queue`
上还有另一个函数叫做 `device.queue.copyExternalImageToTexture`，它可以让我们
把图像复制进纹理。

它可以接收一个 `ImageBitmap`，所以我们拿上一篇文章里的
[magFilter 示例](webgpu-textures.html#a-mag-filter) 来改造一下，
让它加载几张图片。

首先我们需要一些代码，把一张图片变成 `ImageBitmap`

```js
  async function loadImageBitmap(url) {
    const res = await fetch(url);
    const blob = await res.blob();
    return await createImageBitmap(blob, { colorSpaceConversion: 'none' });
  }
```

上面的代码会用图片的 url 调用 `fetch`，返回一个 `Response`。然后我们
从中读取出一个 `Blob`，它以一种不透明的方式表示图片文件的数据。接着再把它传给
`createImageBitmap`，这是浏览器里用来创建 `ImageBitmap` 的标准函数。
我们传入 `{ colorSpaceConversion: 'none' }`，是为了告诉浏览器不要应用任何色彩空间转换。
是否要让浏览器处理色彩空间由你决定。在 WebGPU 中，我们经常会加载法线贴图、
高度图，或者其他并不是颜色数据的图片。
在这些情况下，我们显然不希望浏览器去“动手脚”修改图像里的数据。

现在我们已经有了创建 `ImageBitmap` 的代码，接下来就加载一张图，并创建一个同尺寸的纹理。

我们将加载这张图片

<div class="webgpu_center"><img src="../resources/images/f-texture.png"></div>

以前有人教过我，纹理里带一个 `F` 是非常好的示例，
因为我们可以立刻看出它的朝向。

<div class="webgpu_center"><img src="resources/f-orientation.svg"></div>


```js
-  const texture = device.createTexture({
-    label: 'yellow F on red',
-    size: [kTextureWidth, kTextureHeight],
-    format: 'rgba8unorm',
-    usage:
-      GPUTextureUsage.TEXTURE_BINDING |
-      GPUTextureUsage.COPY_DST,
-  });
+  const url = 'resources/images/f-texture.png';
+  const source = await loadImageBitmap(url);
+  const texture = device.createTexture({
+    label: url,
+    format: 'rgba8unorm',
+    size: [source.width, source.height],
+    usage: GPUTextureUsage.TEXTURE_BINDING |
+           GPUTextureUsage.COPY_DST |
+           GPUTextureUsage.RENDER_ATTACHMENT,
+  });
```

注意，`copyExternalImageToTexture` 要求我们包含
`GPUTextureUsage.COPY_DST` 和 `GPUTextureUsage.RENDER_ATTACHMENT`
这两个 usage 标志。

接着我们就可以把 `ImageBitmap` 复制进纹理

```js
-  device.queue.writeTexture(
-      { texture },
-      textureData,
-      { bytesPerRow: kTextureWidth * 4 },
-      { width: kTextureWidth, height: kTextureHeight },
-  );
+  device.queue.copyExternalImageToTexture(
+    { source, flipY: true },
+    { texture },
+    { width: source.width, height: source.height },
+  );
```

`copyExternalImageToTexture` 的参数分别是：
源、目标、尺寸。对于源参数，
如果我们希望加载时翻转纹理，就可以指定 `flipY: true`。

这样就能工作了！

{{{example url="../webgpu-simple-textured-quad-import-no-mips.html"}}}

## <a id="a-generating-mips-on-the-gpu"></a>在 GPU 上生成 mips

在[上一篇文章](webgpu-textures.html#a-mipmap-filter)里，我们同样也生成过 mipmap，
但当时我们能很方便地直接访问图像数据。加载图片时，我们
可以先把图片画到 2D canvas 上，再调用 `getImageData` 取回数据，
最后生成 mips 并上传。这样会非常慢。
而且还可能有损，因为 canvas 2D 的渲染方式本来就有意依赖具体实现。

当时我们生成 mip 级别时，用的是双线性插值，
这恰好也是 GPU 在 `minFilter: linear` 下做的事情。
所以我们可以直接利用这个特性在 GPU 上生成 mip 级别。

让我们修改上一篇文章中的
[mipmapFilter 示例](webgpu-textures.html#a-mipmap-filter)，
让它加载图片并使用 GPU 生成 mips。

首先，我们要修改创建纹理的代码，让它创建多级 mip。我们需要先知道
要创建多少级，可以这样计算

```js
  const numMipLevels = (...sizes) => {
    const maxSize = Math.max(...sizes);
    return 1 + Math.log2(maxSize) | 0;
  };
```

我们可以给它传 1 个或多个数字，它会返回所需的 mip 层数。比如
`numMipLevels(123, 456)` 返回 `9`。

> * level 0: 123, 456
> * level 1: 61, 228
> * level 2: 30, 114
> * level 3: 15, 57
> * level 4: 7, 28
> * level 5: 3, 14
> * level 6: 1, 7
> * level 7: 1, 3
> * level 8: 1, 1
> 
> 共 9 个 mip 级别

`Math.log2` 会告诉我们一个数对应的 2 的几次方。
换句话说，`Math.log2(8) = 3`，因为 2<sup>3</sup> = 8。也可以换个说法：
`Math.log2` 告诉我们
这个数最多可以被 2 除多少次。

> ```
> Math.log2(8)
>           8 / 2 = 4
>                   4 / 2 = 2
>                           2 / 2 = 1
> ```

所以 8 可以被 2 除 3 次。这正是我们计算需要生成多少级 mip 的依据。
公式就是 `Math.log2(largestSize) + 1`。加上的这个 1
对应原始尺寸的 mip level 0。

所以现在，我们就可以创建出正确数量的 mip 级别了

```js
  const texture = device.createTexture({
    label: url,
    format: 'rgba8unorm',
    mipLevelCount: numMipLevels(source.width, source.height),
    size: [source.width, source.height],
    usage: GPUTextureUsage.TEXTURE_BINDING |
           GPUTextureUsage.COPY_DST |
           GPUTextureUsage.RENDER_ATTACHMENT,
  });
  device.queue.copyExternalImageToTexture(
    { source, flipY: true, },
    { texture },
    { width: source.width, height: source.height },
  );
```

为了生成下一个 mip 级别，我们会像之前那样绘制一个带纹理的四边形，
从已有的 mip 级别绘制到下一级，并使用 `minFilter: linear`。

代码如下

```js
  const generateMips = (() => {
    let sampler;
    let module;
    const pipelineByFormat = {};

    return function generateMips(device, texture) {
      if (!module) {
        module = device.createShaderModule({
          label: 'textured quad shaders for mip level generation',
          code: /* wgsl */ `
            struct VSOutput {
              @builtin(position) position: vec4f,
              @location(0) texcoord: vec2f,
            };

            @vertex fn vs(
              @builtin(vertex_index) vertexIndex : u32
            ) -> VSOutput {
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

              var vsOutput: VSOutput;
              let xy = pos[vertexIndex];
              vsOutput.position = vec4f(xy * 2.0 - 1.0, 0.0, 1.0);
              vsOutput.texcoord = vec2f(xy.x, 1.0 - xy.y);
              return vsOutput;
            }

            @group(0) @binding(0) var ourSampler: sampler;
            @group(0) @binding(1) var ourTexture: texture_2d<f32>;

            @fragment fn fs(fsInput: VSOutput) -> @location(0) vec4f {
              return textureSample(ourTexture, ourSampler, fsInput.texcoord);
            }
          `,
        });

        sampler = device.createSampler({
          minFilter: 'linear',
        });
      }

      if (!pipelineByFormat[texture.format]) {
        pipelineByFormat[texture.format] = device.createRenderPipeline({
          label: 'mip level generator pipeline',
          layout: 'auto',
          vertex: {
            module,
          },
          fragment: {
            module,
            targets: [{ format: texture.format }],
          },
        });
      }
      const pipeline = pipelineByFormat[texture.format];

      const encoder = device.createCommandEncoder({
        label: 'mip gen encoder',
      });

      for (let baseMipLevel = 1; baseMipLevel < texture.mipLevelCount; ++baseMipLevel) {
        const bindGroup = device.createBindGroup({
          layout: pipeline.getBindGroupLayout(0),
          entries: [
            { binding: 0, resource: sampler },
            {
              binding: 1,
              resource: texture.createView({
                baseMipLevel: baseMipLevel - 1,
                mipLevelCount: 1,
              }),
            },
          ],
        });

        const renderPassDescriptor = {
          label: 'our basic canvas renderPass',
          colorAttachments: [
            {
              view: texture.createView({
                baseMipLevel,
                mipLevelCount: 1,
              }),
              loadOp: 'clear',
              storeOp: 'store',
            },
          ],
        };

        const pass = encoder.beginRenderPass(renderPassDescriptor);
        pass.setPipeline(pipeline);
        pass.setBindGroup(0, bindGroup);
        pass.draw(6);  // call our vertex shader 6 times
        pass.end();
      }
      const commandBuffer = encoder.finish();
      device.queue.submit([commandBuffer]);
    };
  })();
```

上面的代码看起来很长，但它和我们到目前为止在纹理示例里使用的代码几乎完全一样。
变化主要有这些：

* 我们创建了一个闭包，用来保存 3 个变量：`module`、`sampler`、`pipelineByFormat`。
  对于 `module` 和 `sampler`，我们会检查它们是否尚未创建；如果还没有，
  就创建一个 `GPUShaderModule` 和 `GPUSampler`，
  以便后续重复使用。

* 我们有一对着色器，它们几乎和前面的所有示例完全相同。
  唯一的区别是这一部分

  ```wgsl
  -  vsOutput.position = uni.matrix * vec4f(xy, 0.0, 1.0);
  -  vsOutput.texcoord = xy * vec2f(1, 50);
  +  vsOutput.position = vec4f(xy * 2.0 - 1.0, 0.0, 1.0);
  +  vsOutput.texcoord = vec2f(xy.x, 1.0 - xy.y);
  ```

  着色器里硬编码的四边形位置数据范围是 0.0 到 1.0，
  所以如果直接使用，它只会覆盖目标纹理右上角的四分之一区域，
  就像前面那些示例一样。我们需要它覆盖整个区域，
  所以通过乘以 2 再减去 1，就能得到一个从 -1,-1 到 +1,+1 的四边形。

  我们还翻转了 Y 方向的纹理坐标。因为在向纹理渲染时，+1,+1 位于右上角，
  而我们也希望被采样纹理的右上角对应到那里。被采样纹理的右上角坐标是 +1, 0。

* 我们使用了一个对象 `pipelineByFormat`，把它当作纹理格式到 pipeline 的映射表。
  这是因为 pipeline 需要提前知道要使用的格式。

* 我们会检查某个特定格式的 pipeline 是否已经存在，如果不存在就创建一个
  
  ```js
      if (!pipelineByFormat[texture.format]) {
        pipelineByFormat[texture.format] = device.createRenderPipeline({
          label: 'mip level generator pipeline',
          layout: 'auto',
          vertex: {
            module,
          },
          fragment: {
            module,
  +          targets: [{ format: texture.format }],
          },
        });
      }
      const pipeline = pipelineByFormat[texture.format];
  ```

  这里最大的不同是 `targets` 使用的是纹理本身的格式，
  而不是我们渲染到 canvas 时用的 `presentationFormat`。

* 最后，我们给 `texture.createView` 传入了一些参数。

  这是我们第一次在把纹理绑定到 bind group 时，
  以及把纹理设为 color target 时使用 `createView`。
  当你把一个纹理绑定进 bind group，或者把纹理指定为渲染目标
  （设置 `colorTargets`）时，
  你既可以直接传入纹理，
  也可以传入一个 `GPUTextureView`。


  ```js
     { binding: resource: someTexture },
  ```

  和

  ```js
     { binding: resource: someTexture.createView(...) }, 
  ```

  直接使用纹理，本质上等价于调用一次不带参数的 `texture.createView`。
  不带参数表示你想访问整个纹理。
  而带参数时，`createView` 允许你只选择纹理的一个子集。
  在这里，我们用 `createView` 选择要读取的 mip 级别，
  并把它设置到 bindGroup 中。然后我们再次使用 `createView`，
  在 render pass descriptor 中选择要渲染到哪个 mip 级别。

  我们会遍历所有需要生成的 mip 级别。
  先为上一个已有数据的 mip 创建 bind group，
  再把 renderPassDescriptor 设置为渲染到当前 mip 级别。然后编码
  这个特定 mip 级别的 render pass。等全部完成后，
  所有 mip 都会被填充出来。

  ```js
      for (let baseMipLevel = 1; baseMipLevel < texture.mipLevelCount; ++baseMipLevel) {
        const bindGroup = device.createBindGroup({
          layout: pipeline.getBindGroupLayout(0),
          entries: [
            { binding: 0, resource: sampler },
  +          {
  +            binding: 1,
  +            resource: texture.createView({
  +              baseMipLevel: baseMipLevel - 1,
  +              mipLevelCount: 1,
  +            }),
  +          },
          ],
        });

        const renderPassDescriptor = {
          label: 'our basic canvas renderPass',
          colorAttachments: [
            {
  +            view: texture.createView({baseMipLevel, mipLevelCount: 1}),
              loadOp: 'clear',
              storeOp: 'store',
            },
          ],
        };

        const pass = encoder.beginRenderPass(renderPassDescriptor);
        pass.setPipeline(pipeline);
        pass.setBindGroup(0, bindGroup);
        pass.draw(6);  // call our vertex shader 6 times
        pass.end();
      }

      const commandBuffer = encoder.finish();
      device.queue.submit([commandBuffer]);
  ```

> 注意：这个函数目前只处理 2D 纹理。
> [关于立方体贴图的文章](webgpu-cube-maps.html#a-texture-helpers)
> 讲了如何把这个函数扩展到 2D-array 纹理和
> cube map。

## <a id="a-texture-helpers"></a>简单的图像加载辅助函数

让我们写几个辅助函数，方便把图像加载到纹理中，
并生成 mips。

下面这个函数会更新第一个 mip 级别，并且可以选择是否翻转图像。
如果这个纹理带有多个 mip 级别，我们就生成它们。

```js
  function copySourceToTexture(device, texture, source, {flipY} = {}) {
    device.queue.copyExternalImageToTexture(
      { source, flipY, },
      { texture },
      { width: source.width, height: source.height },
    );

    if (texture.mipLevelCount > 1) {
      generateMips(device, texture);
    }
  }
```

<a id="a-create-texture-from-source"></a>下面这个函数接收一个 source（这里是 `ImageBitmap`），
会先创建一个对应大小的纹理，然后调用上一个函数
把数据填进去

```js
  function createTextureFromSource(device, source, options = {}) {
    const texture = device.createTexture({
      format: 'rgba8unorm',
*      mipLevelCount: options.mips ? numMipLevels(source.width, source.height) : 1,
      size: [source.width, source.height],
      usage: GPUTextureUsage.TEXTURE_BINDING |
             GPUTextureUsage.COPY_DST |
             GPUTextureUsage.RENDER_ATTACHMENT,
    });
    copySourceToTexture(device, texture, source, options);
    return texture;
  }
```

下面这个函数则接收一个 url，把该 url 加载为 `ImageBitmap`，
然后调用前一个函数创建纹理并用图像内容填充它。

```js
  async function createTextureFromImage(device, url, options) {
    const imgBitmap = await loadImageBitmap(url);
    return createTextureFromSource(device, imgBitmap, options);
  }
```

有了这些辅助函数之后，对上一篇文章中的
[mipmapFilter 示例](webgpu-textures.html#a-mipmap-filter)
来说，唯一比较大的改动就是下面这一段

```js
-  const textures = [
-    createTextureWithMips(createBlendedMipmap(), 'blended'),
-    createTextureWithMips(createCheckedMipmap(), 'checker'),
-  ];
+  const textures = await Promise.all([
+    await createTextureFromImage(device,
+        'resources/images/f-texture.png', {mips: true, flipY: false}),
+    await createTextureFromImage(device,
+        'resources/images/coins.jpg', {mips: true}),
+    await createTextureFromImage(device,
+        'resources/images/Granite_paving_tileable_512x512.jpeg', {mips: true}),
+  ]);
```

上面的代码会加载前面那张 F 纹理，以及下面这两张可平铺纹理

<div class="webgpu_center side-by-side">
  <div class="separate">
    <img src="../resources/images/coins.jpg">
    <div class="copyright">
      <a href="https://renderman.pixar.com/pixar-one-thirty">CC-BY: Pixar</a>
    </div>
  </div>
  <div class="separate">
    <img src="../resources/images/Granite_paving_tileable_512x512.jpeg">
    <div class="copyright">
       <a href="https://commons.wikimedia.org/wiki/File:Granite_paving_tileable_2048x2048.jpg">CC-BY-SA: Coyau</a>
    </div>
  </div>
</div>

效果如下

{{{example url="../webgpu-simple-textured-quad-import.html"}}}

## <a id="a-loading-canvas"></a>加载 Canvas

`copyExternalImageToTexture` 还能接收其他类型的 *source*。其中一种就是 `HTMLCanvasElement`。
我们可以利用它先在 2D canvas 里绘制内容，再把结果放进 WebGPU 的纹理中。
当然，你也可以直接用 WebGPU 往纹理里渲染，然后再把这个纹理拿去做别的渲染。
事实上我们刚刚就这么做过：先渲染到某个 mip 级别，
再把那个 mip 级别作为纹理附件，用来渲染下一个 mip 级别。

不过，有时候 2D canvas 会让某些事情变得更简单，
因为 2D canvas 提供的是相对更高层的 API。

所以，先来做一个简单的 canvas 动画。

```js
const size = 256;
const half = size / 2;

const ctx = document.createElement('canvas').getContext('2d');
ctx.canvas.width = size;
ctx.canvas.height = size;

const hsl = (h, s, l) => `hsl(${h * 360 | 0}, ${s * 100}%, ${l * 100 | 0}%)`;

function update2DCanvas(time) {
  time *= 0.0001;
  ctx.clearRect(0, 0, size, size);
  ctx.save();
  ctx.translate(half, half);
  const num = 20;
  for (let i = 0; i < num; ++i) {
    ctx.fillStyle = hsl(i / num * 0.2 + time * 0.1, 1, i % 2 * 0.5);
    ctx.fillRect(-half, -half, size, size);
    ctx.rotate(time * 0.5);
    ctx.scale(0.85, 0.85);
    ctx.translate(size / 16, 0);
  }
  ctx.restore();
}

function render(time) {
  update2DCanvas(time);
  requestAnimationFrame(render);
}
requestAnimationFrame(render);
```

{{{example url="../canvas-2d-animation.html"}}}

要把这个 canvas 加载到 WebGPU 中，只需要对前一个示例做少量修改。

我们需要先创建一个尺寸正确的纹理。最简单的办法
就是复用上面写好的同一套代码

```js
+  const texture = createTextureFromSource(device, ctx.canvas, {mips: true});

  const textures = await Promise.all([
-    await createTextureFromImage(device,
-        'resources/images/f-texture.png', {mips: true, flipY: false}),
-    await createTextureFromImage(device,
-        'resources/images/coins.jpg', {mips: true}),
-    await createTextureFromImage(device,
-        'resources/images/Granite_paving_tileable_512x512.jpeg', {mips: true}),
+    texture,
  ]);
```

然后我们需要切换到 `requestAnimationFrame` 循环里，更新 2D canvas，
再把它上传给 WebGPU

```js
-  function render() {
+  function render(time) {
+    update2DCanvas(time);
+    copySourceToTexture(device, texture, ctx.canvas);

     ...


    requestAnimationFrame(render);
  }
  requestAnimationFrame(render);

  const observer = new ResizeObserver(entries => {
    for (const entry of entries) {
      const canvas = entry.target;
      const width = entry.contentBoxSize[0].inlineSize;
      const height = entry.contentBoxSize[0].blockSize;
      canvas.width = Math.max(1, Math.min(width, device.limits.maxTextureDimension2D));
      canvas.height = Math.max(1, Math.min(height, device.limits.maxTextureDimension2D));
-      render();
    }
  });
  observer.observe(canvas);

  canvas.addEventListener('click', () => {
    texNdx = (texNdx + 1) % textures.length;
-    render();
  });
```

这样我们就能上传一个 canvas，并为它生成 mip 级别了。

{{{example url="../webgpu-simple-textured-quad-import-canvas.html"}}}

## <a id="a-loading-video"></a>加载视频

以这种方式加载视频也没什么不同。我们可以创建一个 `<video>` 元素，
然后把它像前一个示例中的 canvas 一样传给同样的函数；
做一些很小的调整后，它就能工作。

下面是一段视频

<div class="webgpu_center">
  <div>
     <video muted controls src="../resources/videos/Golden_retriever_swimming_the_doggy_paddle-360-no-audio.webm" style="width: 720px";></video>
     <div class="copyright"><a href="https://commons.wikimedia.org/wiki/File:Golden_retriever_swimming_the_doggy_paddle.webm">CC-BY: Golden Woofs</a></div>
  </div>
</div>

`ImageBitmap` 和 `HTMLCanvasElement` 的宽高属性分别是 `width` 和 `height`，
但 `HTMLVideoElement` 的宽高属性则是 `videoWidth` 和 `videoHeight`。
所以我们先把代码更新一下，处理这个差异

```js
+  function getSourceSize(source) {
+    return [
+      source.videoWidth || source.width,
+      source.videoHeight || source.height,
+    ];
+  }

  function copySourceToTexture(device, texture, source, {flipY} = {}) {
    device.queue.copyExternalImageToTexture(
      { source, flipY, },
      { texture },
-      { width: source.width, height: source.height },
+      getSourceSize(source),
    );

    if (texture.mipLevelCount > 1) {
      generateMips(device, texture);
    }
  }

  function createTextureFromSource(device, source, options = {}) {
+    const size = getSourceSize(source);
    const texture = device.createTexture({
      format: 'rgba8unorm',
-      mipLevelCount: options.mips ? numMipLevels(source.width, source.height) : 1,
-      size: [source.width, source.height],
+      mipLevelCount: options.mips ? numMipLevels(...size) : 1,
+      size,
      usage: GPUTextureUsage.TEXTURE_BINDING |
             GPUTextureUsage.COPY_DST |
             GPUTextureUsage.RENDER_ATTACHMENT,
    });
    copySourceToTexture(device, texture, source, options);
    return texture;
  }
```

然后，我们来设置一个 video 元素

```js
  const video = document.createElement('video');
  video.muted = true;
  video.loop = true;
  video.preload = 'auto';
  video.src = 'resources/videos/Golden_retriever_swimming_the_doggy_paddle-360-no-audio.webm';

  const texture = createTextureFromSource(device, video, {mips: true});
```

并在渲染时更新它

```js
-  function render(time) {
-    update2DCanvas(time);
-    copySourceToTexture(device, texture, ctx.canvas);
+  function render() {
+    copySourceToTexture(device, texture, video);
```

视频有一个麻烦点：在把它交给 WebGPU 之前，
我们必须先等它真正开始播放。在现代浏览器中，
可以通过调用 `video.requestVideoFrameCallback` 来做到这一点。
每当有新的视频帧可用时，它都会回调我们，
因此我们可以用它来确认至少已经有一帧可用了。

作为后备方案，我们可以等时间推进然后祈祷，因为
遗憾的是，老浏览器很难判断什么时候才可以安全地使用视频。

```js
+  function startPlayingAndWaitForVideo(video) {
+    return new Promise((resolve, reject) => {
+      video.addEventListener('error', reject);
+      if ('requestVideoFrameCallback' in video) {
+        video.requestVideoFrameCallback(resolve);
+      } else {
+        const timeWatcher = () => {
+          if (video.currentTime > 0) {
+            resolve();
+          } else {
+            requestAnimationFrame(timeWatcher);
+          }
+        };
+        timeWatcher();
+      }
+      video.play().catch(reject);
+    });
+  }

  const video = document.createElement('video');
  video.muted = true;
  video.loop = true;
  video.preload = 'auto';
  video.src = 'resources/videos/Golden_retriever_swimming_the_doggy_paddle-360-no-audio.webm';
+  await startPlayingAndWaitForVideo(video);

  const texture = createTextureFromSource(device, video, {mips: true});
```

另一个麻烦点是，我们必须等用户与页面发生交互之后，
才能开始播放视频[^autoplay]。所以我们加一点 HTML，
放一个播放按钮。

[^autoplay]: 让视频自动播放的方法有很多种，通常是在没有音频的情况下，
不必等待用户与页面交互。
这些做法会随着时间变化，所以这里不展开讨论。

```html
  <body>
    <canvas></canvas>
+    <div id="start">
+      <div>鈻讹笍</div>
+    </div>
  </body>
```

再写一点 CSS 把它居中

```css
#start {
  position: fixed;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}
#start>div {
  font-size: 200px;
  cursor: pointer;
}
```

接着写一个函数，等待用户点击并把它隐藏起来。

```js
+  function waitForClick() {
+    return new Promise(resolve => {
+      window.addEventListener(
+        'click',
+        () => {
+          document.querySelector('#start').style.display = 'none';
+          resolve();
+        },
+        { once: true });
+    });
+  }

  const video = document.createElement('video');
  video.muted = true;
  video.loop = true;
  video.preload = 'auto';
  video.src = 'resources/videos/Golden_retriever_swimming_the_doggy_paddle-360-no-audio.webm';
+  await waitForClick();
  await startPlayingAndWaitForVideo(video);

  const texture = createTextureFromSource(device, video, {mips: true});
```

我们还可以再加一个点击事件，用来暂停视频

```js
  const video = document.createElement('video');
  video.muted = true;
  video.loop = true;
  video.preload = 'auto';
  video.src = 'resources/videos/pexels-anna-bondarenko-5534310 (540p).mp4'; /* webgpufundamentals: url */
  await waitForClick();
  await startPlayingAndWaitForVideo(video);

+  canvas.addEventListener('click', () => {
+    if (video.paused) {
+      video.play();
+    } else {
+      video.pause();
+    }
+  });
```

这样我们就能把视频放进纹理里了

{{{example url="../webgpu-simple-textured-quad-import-video.html"}}}

还有一个可以做的优化：我们可以只在视频发生变化时
更新纹理。

例如

```js
  const video = document.createElement('video');
  video.muted = true;
  video.loop = true;
  video.preload = 'auto';
  video.src = 'resources/videos/Golden_retriever_swimming_the_doggy_paddle-360-no-audio.webm';
  await waitForClick();
  await startPlayingAndWaitForVideo(video);

+  let alwaysUpdateVideo = !('requestVideoFrameCallback' in video);
+  let haveNewVideoFrame = false;
+  if (!alwaysUpdateVideo) {
+    function recordHaveNewFrame() {
+      haveNewVideoFrame = true;
+      video.requestVideoFrameCallback(recordHaveNewFrame);
+    }
+    video.requestVideoFrameCallback(recordHaveNewFrame);
+  }

  ...

  function render() {
+    if (alwaysUpdateVideo || haveNewVideoFrame) {
+      haveNewVideoFrame = false;
      copySourceToTexture(device, texture, video);
+    }

    ...
```

这样改完后，我们只会在每个新视频帧到来时更新纹理。比如，
在一台显示刷新率为每秒 120 帧的设备上，我们仍然会以每秒 120 帧去渲染，
所以动画、镜头移动等依然会很平滑。
但是视频纹理本身只会按它自己的帧率更新，
比如 30fps。

**不过！WebGPU 对高效使用视频提供了专门的支持**

我们会在[另一篇文章](webgpu-textures-external-video.html)里介绍它。
上面这种通过 `device.query.copyExternalImageToTexture` 的方式，
实际上是在做**一次拷贝**。拷贝是要花时间的。比如一个 4K 视频的分辨率
通常是 3840 × 2160，如果格式是 `rgba8unorm`，
那每帧需要复制的数据量大约就是 31MB。
[外部纹理](webgpu-textures-external-video.html)
可以让你直接使用视频数据本身而不进行复制，但需要不同的方法，
并且也有一些限制。

## <a id="a-texture-atlases"></a>纹理图集

从上面的例子中我们可以看出，想要用纹理绘制一个东西，
我们需要创建纹理，把数据放进去，用采样器把它绑定到 bindGroup，
然后在着色器里引用它。那么如果我们想在一个物体上
绘制多张不同的纹理呢？比如一把椅子，椅腿和靠背
是木头材质，而坐垫是布料材质。

<div class="webgpu_center">
  <div class="center">
    <model-viewer 
      src="/webgpu/resources/models/gltf/cc0_chair.glb"
      camera-controls
      touch-action="pan-y"
      camera-orbit="45deg 70deg 2.5m"
      interaction-prompt="none"
      disable-zoom
      disable-pan
      style="width: 400px; height: 400px;"></model-viewer>
  </div>
  <div>
    <a href="https://skfb.ly/opnwY"></a>"[CC0] Chair" by adadadad5252341 <a href="http://creativecommons.org/licenses/by/4.0/">CC-BY 4.0</a>
  </div>
</div>

或者一辆汽车，轮胎是橡胶，车身是油漆，
保险杠和轮毂盖则是铬金属。

<div class="webgpu_center">
  <div class="center">
    <model-viewer 
      src="/webgpu/resources/models/gltf/classic_muscle_car.glb"
      camera-controls
      touch-action="pan-y"
      camera-orbit="45deg 70deg 20m"
      interaction-prompt="none"
      disable-zoom
      disable-pan
      style="width: 700px; height: 400px;"></model-viewer>
  </div>
  <div>
    <a href="https://skfb.ly/6Usqo"></a>"Classic Muscle car" by Lexyc16 <a href="http://creativecommons.org/licenses/by/4.0/">CC-BY 4.0</a>
  </div>
</div>

如果不做别的处理，你可能会觉得椅子要绘制 2 次：
木头部分绘制一次，用木纹理；坐垫部分再绘制一次，用布料纹理。
汽车则会有更多次 draw：轮胎一次、车身一次、
保险杠一次，等等。

这样会变得很慢，因为每个物体都需要多次 draw call。
我们也可以尝试通过给着色器增加更多输入来解决，
比如 2、3、4 张纹理，再配上各自的纹理坐标，
但那样既不够灵活，也同样会比较慢，
因为我们需要读取全部 4 张纹理，还要额外写代码从中做选择。

处理这种情况最常见的方法，是使用所谓的
[Texture Atlas](https://www.google.com/search?q=texture+atlas)。
Texture Atlas 其实就是一个“装了多张图的纹理”的高端叫法。
然后我们再用纹理坐标来决定每一部分使用哪一块图像。

让我们把这 6 张图包到一个立方体上

<div class="webgpu_table_div_center">
  <style>
    table.webgpu_table_center {
      border-spacing: 0.5em;
      border-collapse: separate;
    }
    table.webgpu_table_center img {
      display:block;
    }
  </style>
  <table class="webgpu_table_center">
    <tr><td><img src="resources/noodles-01.jpg" /></td><td><img src="resources/noodles-02.jpg" /></td></tr>
    <tr><td><img src="resources/noodles-03.jpg" /></td><td><img src="resources/noodles-04.jpg" /></td></tr>
    <tr><td><img src="resources/noodles-05.jpg" /></td><td><img src="resources/noodles-06.jpg" /></td></tr>
  </table>
</div>

用 Photoshop 或者 [Photopea](https://photopea.com) 这样的图像编辑软件，
我们可以把这 6 张图片合成到一张图里

<img class="webgpu_center" src="../resources/images/noodles.jpg" />

然后我们就可以制作一个立方体，并提供对应的纹理坐标，
让图像的各个部分映射到立方体的不同面上。为了简单起见，我把
上面这 6 张图按 4x2 的网格排进同一张纹理里，都是方块，
所以计算每个方块对应的纹理坐标会比较容易。

<div class="webgpu_center center diagram">
  <div>
    <div data-diagram="texture-atlas" style="display: inline-block; width: 600px;"></div>
  </div>
</div>

> 上面的图可能会让人有点困惑，因为很多资料常说纹理坐标
> 的 0,0 在左下角。但实际上并不存在绝对意义上的“下方”。
> 真正的概念只是：纹理坐标 0,0 指向纹理数据中的第一个像素。
> 而纹理数据中的第一个像素，正是图像的左上角。
> 如果你坚持认为 0,0 = 左下角，那么我们的纹理坐标
> 会像下面这样可视化。**但它们仍然是同一组坐标**。

<div class="webgpu_center center diagram">
  <div>
    <div data-diagram="texture-atlas-bottom-left" style="display: inline-block; width: 600px;"></div>
    <div class="center">0,0 在左下角</div>
  </div>
</div>


下面是立方体的位置顶点数据，以及对应的纹理坐标

```js
function createCubeVertices() {
  const vertexData = new Float32Array([
     //  position   |  texture coordinate
     //-------------+----------------------
     // front face     select the top left image
    -1,  1,  1,        0   , 0  ,
    -1, -1,  1,        0   , 0.5,
     1,  1,  1,        0.25, 0  ,
     1, -1,  1,        0.25, 0.5,
     // right face     select the top middle image
     1,  1, -1,        0.25, 0  ,
     1,  1,  1,        0.5 , 0  ,
     1, -1, -1,        0.25, 0.5,
     1, -1,  1,        0.5 , 0.5,
     // back face      select to top right image
     1,  1, -1,        0.5 , 0  ,
     1, -1, -1,        0.5 , 0.5,
    -1,  1, -1,        0.75, 0  ,
    -1, -1, -1,        0.75, 0.5,
    // left face       select the bottom left image
    -1,  1,  1,        0   , 0.5,
    -1,  1, -1,        0.25, 0.5,
    -1, -1,  1,        0   , 1  ,
    -1, -1, -1,        0.25, 1  ,
    // bottom face     select the bottom middle image
     1, -1,  1,        0.25, 0.5,
    -1, -1,  1,        0.5 , 0.5,
     1, -1, -1,        0.25, 1  ,
    -1, -1, -1,        0.5 , 1  ,
    // top face        select the bottom right image
    -1,  1,  1,        0.5 , 0.5,
     1,  1,  1,        0.75, 0.5,
    -1,  1, -1,        0.5 , 1  ,
     1,  1, -1,        0.75, 1  ,

  ]);

  const indexData = new Uint16Array([
     0,  1,  2,  2,  1,  3,  // front
     4,  5,  6,  6,  5,  7,  // right
     8,  9, 10, 10,  9, 11,  // back
    12, 13, 14, 14, 13, 15,  // left
    16, 17, 18, 18, 17, 19,  // bottom
    20, 21, 22, 22, 21, 23,  // top
  ]);

  return {
    vertexData,
    indexData,
    numVertices: indexData.length,
  };
}
```

这个示例会从[相机那篇文章](webgpu-cameras.html)中的一个示例开始改。
如果你还没读过那篇文章，可以先去看，它属于学习 3D 的那一组系列文章。
现在最重要的是，和上面一样，我们会在顶点着色器中输出位置与纹理坐标，
然后在片元着色器中用这些纹理坐标去查找纹理中的值。
下面是从相机示例的着色器改过来的版本。

```wgsl
struct Uniforms {
  matrix: mat4x4f,
};

struct Vertex {
  @location(0) position: vec4f,
-  @location(1) color: vec4f,
+  @location(1) texcoord: vec2f,
};

struct VSOutput {
  @builtin(position) position: vec4f,
-  @location(0) color: vec4f,
+  @location(0) texcoord: vec2f,
};

@group(0) @binding(0) var<uniform> uni: Uniforms;
+@group(0) @binding(1) var ourSampler: sampler;
+@group(0) @binding(2) var ourTexture: texture_2d<f32>;

@vertex fn vs(vert: Vertex) -> VSOutput {
  var vsOut: VSOutput;
  vsOut.position = uni.matrix * vert.position;
-  vsOut.color = vert.color;
+  vsOut.texcoord = vert.texcoord;
  return vsOut;
}

@fragment fn fs(vsOut: VSOutput) -> @location(0) vec4f {
-  return vsOut.color;
+  return textureSample(ourTexture, ourSampler, vsOut.texcoord);
}
```

我们做的事情其实很简单：把“每个顶点传一个颜色”
改成“每个顶点传一个纹理坐标”，再像上面那样把纹理坐标传给片元着色器。
然后在片元着色器里，也像前面一样使用它。

在 JavaScript 中，我们需要把那个示例的 pipeline
从接收颜色改成接收纹理坐标

```js
  const pipeline = device.createRenderPipeline({
    label: '2 attributes',
    layout: 'auto',
    vertex: {
      module,
      buffers: [
        {
-          arrayStride: (4) * 4, // (3) floats 4 bytes each + one 4 byte color
+          arrayStride: (3 + 2) * 4, // (3+2) floats 4 bytes each
          attributes: [
            {shaderLocation: 0, offset: 0, format: 'float32x3'},  // position
-            {shaderLocation: 1, offset: 12, format: 'unorm8x4'},  // color
+            {shaderLocation: 1, offset: 12, format: 'float32x2'},  // texcoord
          ],
        },
      ],
    },
    fragment: {
      module,
      targets: [{ format: presentationFormat }],
    },
    primitive: {
      cullMode: 'back',
    },
    depthStencil: {
      depthWriteEnabled: true,
      depthCompare: 'less',
      format: 'depth24plus',
    },
  });
```

为了让数据更小，我们会像[顶点缓冲区那篇文章](webgpu-vertex-buffers.html)里讲的那样，
改用索引。

```js
-  const { vertexData, numVertices } = createFVertices();
+  const { vertexData, indexData, numVertices } = createCubeVertices();
  const vertexBuffer = device.createBuffer({
    label: 'vertex buffer vertices',
    size: vertexData.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST,
  });
  device.queue.writeBuffer(vertexBuffer, 0, vertexData);

+  const indexBuffer = device.createBuffer({
+    label: 'index buffer',
+    size: indexData.byteLength,
+    usage: GPUBufferUsage.INDEX | GPUBufferUsage.COPY_DST,
+  });
+  device.queue.writeBuffer(indexBuffer, 0, indexData);
```

我们需要把前面所有“纹理加载”和“mip 生成”的代码复制到这个示例里，
然后用它加载这张纹理图集图片。我们还需要创建一个采样器，
并把它们加进 bindGroup

```js
+  const texture = await createTextureFromImage(device,
+      'resources/images/noodles.jpg', {mips: true, flipY: false});
+
+  const sampler = device.createSampler({
+    magFilter: 'linear',
+    minFilter: 'linear',
+    mipmapFilter: 'linear',
+  });

  const bindGroup = device.createBindGroup({
    label: 'bind group for object',
    layout: pipeline.getBindGroupLayout(0),
    entries: [
      { binding: 0, resource: uniformBuffer },
+      { binding: 1, resource: sampler },
+      { binding: 2, resource: texture },
    ],
  });
```

我们还需要做一点 3D 数学运算，来设置用于 3D 绘制的矩阵。
（同样地，关于 3D 数学的细节可以参考[相机那篇文章](webgpu-cameras.html)。）

```js
  const degToRad = d => d * Math.PI / 180;

  const settings = {
    rotation: [degToRad(20), degToRad(25), degToRad(0)],
  };

  const radToDegOptions = { min: -360, max: 360, step: 1, converters: GUI.converters.radToDeg };

  const gui = new GUI();
  gui.onChange(render);
  gui.add(settings.rotation, '0', radToDegOptions).name('rotation.x');
  gui.add(settings.rotation, '1', radToDegOptions).name('rotation.y');
  gui.add(settings.rotation, '2', radToDegOptions).name('rotation.z');

  ...

  function render() {

    ...

    const aspect = canvas.clientWidth / canvas.clientHeight;
    mat4.perspective(
        60 * Math.PI / 180,
        aspect,
        0.1,      // zNear
        10,      // zFar
        matrixValue,
    );
    const view = mat4.lookAt(
      [0, 1, 5],  // camera position
      [0, 0, 0],  // target
      [0, 1, 0],  // up
    );
    mat4.multiply(matrixValue, view, matrixValue);
    mat4.rotateX(matrixValue, settings.rotation[0], matrixValue);
    mat4.rotateY(matrixValue, settings.rotation[1], matrixValue);
    mat4.rotateZ(matrixValue, settings.rotation[2], matrixValue);

    // upload the uniform values to the uniform buffer
    device.queue.writeBuffer(uniformBuffer, 0, uniformValues);
```

渲染时，我们还需要改成使用索引绘制

```js
    const encoder = device.createCommandEncoder();
    const pass = encoder.beginRenderPass(renderPassDescriptor);
    pass.setPipeline(pipeline);
    pass.setVertexBuffer(0, vertexBuffer);
+    pass.setIndexBuffer(indexBuffer, 'uint16');

    ...

    pass.setBindGroup(0, bindGroup);
-    pass.draw(numVertices);
+    pass.drawIndexed(numVertices);

    pass.end();
```

最终我们就能得到一个立方体，每一面显示不同的图像，
而且只使用了一张纹理。

{{{example url="../webgpu-texture-atlas.html"}}}

使用纹理图集的好处是，只需要加载 1 张纹理，着色器也能保持简单，
因为它只需要引用 1 张纹理；同时绘制这个物体也只需要 1 次 draw call，
而不是像把图片分开存放时那样，每张纹理都要单独 draw 一次。

<!-- keep this at the bottom of the article -->
<script type="module" src="/3rdparty/model-viewer.3.3.0.min.js"></script>
<script type="module" src="webgpu-importing-textures.js"></script>
