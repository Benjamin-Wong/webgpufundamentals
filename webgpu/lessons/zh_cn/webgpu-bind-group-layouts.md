Title: WebGPU Bind Group Layout
Description: 显式 Bind Group Layout
TOC: Bind Group Layout

Bind Group Layout 的作用，
是让 WebGPU 能够更简单、更高效地把 Bind Group
和 Compute / Render Pipeline 匹配起来。

## 工作原理

一个 Pipeline，比如 `GPUComputePipeline`
或者 `GPURenderPipeline`，
会使用一个 `GPUPipelineLayout`。
而 `GPUPipelineLayout`
又定义了 0 个或多个
`GPUBindGroupLayout`。
每个 `GPUBindGroupLayout`
都对应一个特定的 group index。

<div class="webgpu_center"><img src="resources/webgpu-bind-group-layouts.svg" style="width: 900px;"></div>

Bind Group 本身在创建时，
也必须指定某个具体的 `GPUBindGroupLayout`。

当你调用 `draw` 或 `dispatchWorkgroups` 时，
WebGPU 实际上只需要检查：
当前 pipeline 的 `GPUPipelineLayout`
中，每个 group index 上的 `GPUBindGroupLayout`
是否与当前已绑定的 bind group
（也就是通过 `setBindGroup` 设置的那些）
相匹配。
这个检查本身非常简单。
绝大多数细节检查，
都已经在创建 bind group 的时候完成了。
这样一来，等真正进入绘制或计算阶段时，
几乎就没什么东西需要再检查了。

如果你在创建 pipeline 时使用 `layout: 'auto'`，
那么 pipeline 会自动生成自己的 `GPUPipelineLayout`，
并自动填充对应的 `GPUBindGroupLayout`。
这个网站上的大多数示例，
都是这么做的。

有两个主要原因，会让你**不想**使用 `layout: 'auto'`。

1. **你想要一个不同于默认 `'auto'` 布局的 layout**

   例如你想把 `rgba32float`
   当作纹理使用，
   但在尝试时却报错了。
   （下面会讲）

2. **你想让一个 bind group 被多个 pipeline 共用**

   你不能把一个 bind group
   从某个使用 `layout: 'auto'`
   自动生成的 bindGroupLayout 创建出来之后，
   再拿去给另一个不同的 pipeline 使用。

## <a id="a-rgba32float"></a>使用不同于 `layout: 'auto'` 的 bind group layout - `'rgba32float'`

关于 bind group layout 自动生成的规则，
规范里有[详细说明](https://www.w3.org/TR/webgpu/#abstract-opdef-default-pipeline-layout)，
不过这里我们先举一个具体例子……

假设我们想用一张 `rgba32float` 纹理。
让我们从[纹理那篇文章](webgpu-textures.html)中的第一个纹理示例开始，
那个示例会画出一个上下颠倒的 5x7 texel 的字母 “F”。
我们把它改成使用 `rgba32float` 纹理。

改动如下。

```js
  const kTextureWidth = 5;
  const kTextureHeight = 7;
-  const _ = [255,   0,   0, 255];  // red
-  const y = [255, 255,   0, 255];  // yellow
-  const b = [  0,   0, 255, 255];  // blue
-  const textureData = new Uint8Array([
+  const _ = [1, 0, 0, 1];  // red
+  const y = [1, 1, 0, 1];  // yellow
+  const b = [0, 0, 1, 1];  // blue
+  const textureData = new Float32Array([
    b, _, _, _, _,
    _, y, y, y, _,
    _, y, _, _, _,
    _, y, y, _, _,
    _, y, _, _, _,
    _, y, _, _, _,
    _, _, _, _, _,
  ].flat());

  const texture = device.createTexture({
    label: 'yellow F on red',
    size: [kTextureWidth, kTextureHeight],
-    format: 'rgba8unorm',
+    format: 'rgba32float',
    usage:
      GPUTextureUsage.TEXTURE_BINDING |
      GPUTextureUsage.COPY_DST,
  });
  device.queue.writeTexture(
      { texture },
      textureData,
-      { bytesPerRow: kTextureWidth * 4 },
+      { bytesPerRow: kTextureWidth * 4 * 4 },
      { width: kTextureWidth, height: kTextureHeight },
  );

```

运行之后，我们会得到一个错误。

{{{example url="../webgpu-bind-group-layouts-rgba32float-broken.html"}}}

我在测试浏览器里看到的错误是：

> - WebGPU GPUValidationError: None of the supported sample types (UnfilterableFloat) of [Texture "yellow F on red"] match the expected sample types (Float).`<br>
> - While validating entries[1] as a Sampled Texture. Expected entry layout: {sampleType: TextureSampleType::Float, viewDimension: 2, multisampled: 0}`<br>
> - While validating [BindGroupDescriptor] against [BindGroupLayout (unlabeled)]`<br>
> - While calling [Device].CreateBindGroup([BindGroupDescriptor])`

这是怎么回事？
原来，`rgba32float`
（以及所有 `xxx32float`）
纹理默认都是不可过滤的。
确实存在一个[可选特性](webgpu-limits-and-features.html)
能让它们变成可过滤的，
但这个特性并不保证在所有设备上都可用。
尤其是在移动设备上，
至少到 2024 年为止，这种情况非常常见。

默认情况下，当你像下面这样声明一个
`texture_2d<f32>` 绑定：

```wgsl
      @group(0) @binding(1) var ourTexture: texture_2d<f32>;
```

并且在创建 pipeline 时使用 `layout: 'auto'`，
WebGPU 会自动生成一个 bind group layout，
它明确要求这里绑定的必须是“可过滤”的纹理。
如果你尝试绑定一个不可过滤的纹理，
就会报错。

如果你想使用一个不能过滤的纹理，
那就必须手动创建 bind group layout。

有一个工具，[在这里](resources/wgsl-offset-computer.html)，
你只要把 shader 粘进去，
它就会帮你生成自动布局。
把上面这个示例的 shader 粘进去后，
我得到的是

```js
const bindGroupLayoutDescriptors = [
  {
    entries: [
      {
        binding: 0,
        visibility: GPUShaderStage.FRAGMENT,
        sampler: {
          type: "filtering",
        },
      },
      {
        binding: 1,
        visibility: GPUShaderStage.FRAGMENT,
        texture: {
          sampleType: "float",
          viewDimension: "2d",
          multisampled: false,
        },
      },
    ],
  },
];
```

这是一个 `GPUBindGroupLayoutDescriptor`
数组。
从上面可以看到，这个 bind group 使用了
`sampleType: "float"`。
这对 `'rgba8unorm'`
是对的，
但对 `'rgba32float'`
却不是。
具体某种纹理格式支持哪些 sample type，
可以在规范里的
[这张表](https://www.w3.org/TR/webgpu/#texture-format-caps)
中查到。

要修复这个示例，
我们需要同时调整纹理绑定和 sampler 绑定。
sampler 绑定需要改成
`'non-filtering'` sampler。
纹理绑定则需要改成
`'unfilterable-float'`。

所以第一步，我们需要创建一个 `GPUBindGroupLayout`

```js
  const bindGroupLayout = device.createBindGroupLayout({
    entries: [
      {
        binding: 0,
        visibility: GPUShaderStage.FRAGMENT,
        sampler: {
*          type: 'non-filtering',
        },
      },
      {
        binding: 1,
        visibility: GPUShaderStage.FRAGMENT,
        texture: {
*          sampleType: 'unfilterable-float',
          viewDimension: '2d',
          multisampled: false,
        },
      },
    ],
  });
```

上面标出来的两处，就是关键改动。

然后我们还需要创建一个 `GPUPipelineLayout`，
它本质上就是某个 pipeline 所用
`GPUBindGroupLayout`
的数组。

```js
  const pipelineLayout = device.createPipelineLayout({
    bindGroupLayouts: [ bindGroupLayout ],
  });
```

`createPipelineLayout` 接收一个对象，
其中包含一个 `GPUBindGroupLayout` 数组。
它们的顺序就对应 group index：
第一个元素对应 `@group(0)`，
第二个元素对应 `@group(1)`，
以此类推。
如果你需要跳过某个 group，
那就必须在数组中插入空元素或 `undefined`。

最后，在创建 pipeline 时，
把这个 pipeline layout 传进去

```js
  const pipeline = device.createRenderPipeline({
    label: 'hardcoded textured quad pipeline',
-    layout: 'auto',
+    layout: pipelineLayout,
    vertex: {
      module,
    },
    fragment: {
      module,
      targets: [{ format: presentationFormat }],
    },
  });
```

这样一来，示例就又能工作了，
而且现在它用的是一张 `rgba32float`
纹理。

{{{example url="../webgpu-bind-group-layouts-rgba32float-fixed.html"}}}

注意：这个示例之所以能工作，
一方面是因为我们上面手动创建了一个
接受 `unfilterable-float`
的 bind group layout；
另一方面也因为示例中使用的 `GPUSampler`
只用了 `'nearest'`
过滤。
如果把 `magFilter`、`minFilter`
或者 `mipmapFilter`
中的任何一个设成 `'linear'`，
就会报错，说我们试图在一个
`'non-filtering'` 的 sampler 绑定上，
使用 `'filtering'` sampler。

## 使用不同于 `layout: 'auto'` 的 bind group layout - 动态偏移量

默认情况下，当你创建 bind group 并绑定一个 uniform buffer
或 storage buffer 时，绑定的是整个 buffer。
你也可以在创建 bind group 时，
额外传入 offset 和 length。
但无论是哪一种方式，一旦设置好，
后面就不能改了。

WebGPU 提供了一种机制，
允许你在调用 `setBindGroup` 时动态修改 offset。
要使用这个特性，
你必须手动创建 bind group layout，
并对每个你希望之后动态指定 offset 的 binding
设置 `hasDynamicOffsets: true`。

为了简单起见，我们继续使用
[基础那篇文章](webgpu-fundamentals.html#a-run-computations-on-the-gpu)
里的简单 compute 示例。
我们把它改成从同一个 buffer 中读取两组值相加，
并通过 dynamic offset 来决定使用哪一组。

先把 shader 改成这样

```wgsl
@group(0) @binding(0) var<storage, read_write> a: array<f32>;
@group(0) @binding(1) var<storage, read_write> b: array<f32>;
@group(0) @binding(2) var<storage, read_write> dst: array<f32>;

@compute @workgroup_size(1) fn computeSomething(
  @builtin(global_invocation_id) id: vec3u
) {
  let i = id.x;
  dst[i] = a[i] + b[i];
}
```

可以看到，它只是把 `a`
和 `b`
相加后写到 `dst`
里。

接着创建 bind group layout

```js
  const bindGroupLayout = device.createBindGroupLayout({
    entries: [
      {
        binding: 0,
        visibility: GPUShaderStage.COMPUTE,
        buffer: {
          type: 'storage',
          hasDynamicOffset: true,
        },
      },
      {
        binding: 1,
        visibility: GPUShaderStage.COMPUTE,
        buffer: {
          type: 'storage',
          hasDynamicOffset: true,
        },
      },
      {
        binding: 2,
        visibility: GPUShaderStage.COMPUTE,
        buffer: {
          type: 'storage',
          hasDynamicOffset: true,
        },
      },
    ],
  });
```

这里所有 binding
都被标记成了 `hasDynamicStorage: true`。

然后用它来创建 pipeline

```js
  const pipelineLayout = device.createPipelineLayout({
    bindGroupLayouts: [ bindGroupLayout ],
  });

  const pipeline = device.createComputePipeline({
-    label: 'double compute pipeline',
-    layout: 'auto',
+    label: 'add elements compute pipeline',
+    layout: pipelineLayout,
    compute: {
      module,
    },
  });
```

接下来设置 buffer。
offset 必须是 256 的倍数 [^minStorageBufferOffsetAlignment]，
所以我们创建一个大小为
256 * 3 字节的 buffer，
这样至少就有 3 个有效偏移量：
0、256 和 512。

[^minStorageBufferOffsetAlignment]: 你的设备也有可能支持更小的 offset。
可以查看 [limits and features](webgpu-limits-and-features.html)
中的 `minStorageBufferOffsetAlignment`
或 `minUniformBufferOffsetAlignment`。

```js
-  const input = new Float32Array([1, 3, 5]);
+  const input = new Float32Array(64 * 3);
+  input.set([1, 3, 5]);
+  input.set([11, 12, 13], 64);

  // create a buffer on the GPU to hold our computation
  // input and output
  const workBuffer = device.createBuffer({
    label: 'work buffer',
    size: input.byteLength,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_SRC | GPUBufferUsage.COPY_DST,
  });
  // Copy our input data to that buffer
  device.queue.writeBuffer(workBuffer, 0, input);
```

上面的代码创建了一个 `64 * 3`
个 32 位浮点数的数组，
也就是 768 字节。

由于原始示例本来就是在同一个 buffer 上读写，
这里我们也只是把同一个 buffer
绑定 3 次。

```js
  // Setup a bindGroup to tell the shader which
  // buffers to use for the computation
  const bindGroup = device.createBindGroup({
    label: 'bindGroup for work buffer',
    layout: pipeline.getBindGroupLayout(0),
    entries: [
-      { binding: 0, resource: workBuffer  },
+      { binding: 0, resource: { buffer: workBuffer, size: 256 } },
+      { binding: 1, resource: { buffer: workBuffer, size: 256 } },
+      { binding: 2, resource: { buffer: workBuffer, size: 256 } },
    ],
  });
```

注意，这里必须显式指定 size，
否则它会默认绑定整个 buffer。
那样一来，如果之后再设置大于 0 的 offset，
就会报错，因为你等于指定了一个超出范围的 buffer 区间。

现在，在 `setBindGroup` 中，
我们需要为每个带 dynamic offset 的 buffer
都传一个 offset。
因为我们在 bind group layout 中把这 3 个 entry
都标记成了 `hasDynamicOffset: true`，
所以这里就必须按 binding slot 的顺序传入 3 个 offset。

```js
  ...
  pass.setPipeline(pipeline);
-  pass.setBindGroup(0, bindGroup);
+  pass.setBindGroup(0, bindGroup, [0, 256, 512]);
  pass.dispatchWorkgroups(3);
  pass.end();
```

最后还要把显示结果的代码也改一下

```js
-  console.log(input);
-  console.log(result);
+  console.log('a', input.slice(0, 3));
+  console.log('b', input.slice(64, 64 + 3));
+  console.log('dst', result.slice(128, 128 + 3));
```

{{{example url="../webgpu-bind-group-layouts-dynamic-offsets.html"}}}

注意，使用 dynamic offset
会比非动态 offset 稍微慢一些。
原因在于，非动态 offset 情况下，
offset 和 size 是否在 buffer 合法范围内，
会在创建 bind group 时就检查完；
而 dynamic offset 的检查，
必须等到调用 `setBindGroup` 时才能做。
如果你只调用几百次 `setBindGroup`，
这点差别通常无所谓。
但如果你要调用几千次，
影响就可能更明显一些。

## <a id="a-sharing-bind-groups"></a>让一个 bind group 被多个 pipeline 共享

另一个手动创建 bind group layout 的原因，
就是让同一个 bind group
能够被多个 pipeline 共用。

一个很常见、很希望能复用 bind group 的场景，
就是带阴影的基础 3D 场景渲染器。

在基础 3D 场景渲染器中，
通常会把绑定拆成几类：

* globals（例如透视矩阵和视图矩阵）
* materials（纹理、颜色等材质信息）
* locals（例如模型矩阵）

然后渲染过程大概是这样

```
setBindGroup(0, globalsBG)
for each material
  setBindGroup(1, materialBG)
  for each object that uses material
    setBindGroup(2, localBG)
    draw(...)
```

当你再加入[阴影](webgpu-shadows.html)时，
就需要先用一个专门的 shadow map pipeline
去绘制阴影贴图。
这时候，如果要为这些数据分别准备两套 bind group，
一套给正常绘制用，
另一套给 shadow map pipeline 用，
事情就会变得很麻烦。
显然，直接做一套 bind group，
然后两种 pipeline 都能用，
会简单得多。

不过那样的示例写起来会很大，
单纯为了展示 bind group 共享有点不划算。
虽然[阴影那篇文章](webgpu-shadows.html)
里就实际用到了共享 bind group，
但这里我们还是继续拿
[基础那篇文章](webgpu-fundamentals.html#a-run-computations-on-the-gpu)
里的简单 compute 示例来演示：
让 2 个 compute pipeline
共用同一个 bind group。

首先，再增加一个 shader module，
它的功能是加 3

```js
-  const module = device.createShaderModule({
+  const moduleTimes2 = device.createShaderModule({
    label: 'doubling compute module',
    code: /* wgsl */ `
      @group(0) @binding(0) var<storage, read_write> data: array<f32>;

      @compute @workgroup_size(1) fn computeSomething(
        @builtin(global_invocation_id) id: vec3u
      ) {
        let i = id.x;
        data[i] = data[i] * 2.0;
      }
    `,
  });

+  const modulePlus3 = device.createShaderModule({
+    label: 'adding 3 compute module',
+    code: /* wgsl */ `
+      @group(0) @binding(0) var<storage, read_write> data: array<f32>;
+
+      @compute @workgroup_size(1) fn computeSomething(
+        @builtin(global_invocation_id) id: vec3u
+      ) {
+        let i = id.x;
+        data[i] = data[i] + 3.0;
+      }
+    `,
+  });
```

然后创建一个 `GPUBindGroupLayout`
和一个 `GPUPipelineLayout`，
让这两个 pipeline
都能共用同一个 `GPUBindGroup`。

```js
  const bindGroupLayout = device.createBindGroupLayout({
    entries: [
      {
        binding: 0,
        visibility: GPUShaderStage.COMPUTE,
        buffer: {
          type: 'storage',
          minBindingSize: 0,
        },
      },
    ],
  });

  const pipelineLayout = device.createPipelineLayout({
    bindGroupLayouts: [ bindGroupLayout ],
  });
```

接着在创建 pipeline 时都用这套布局。

```js
-  const pipeline = device.createComputePipeline({
+  const pipelineTimes2 = device.createComputePipeline({
    label: 'doubling compute pipeline',
-    layout: 'auto',
+    layout: pipelineLayout,
    compute: {
      module: moduleTimes2,
    },
  });

+  const pipelinePlus3 = device.createComputePipeline({
+    label: 'plus 3 compute pipeline',
+    layout: pipelineLayout,
+    compute: {
+      module: modulePlus3,
+    },
+  });
```

在设置 bind group 时，
我们也直接使用这个 `bindGroupLayout`

```js
  // Setup a bindGroup to tell the shader which
  // buffer to use for the computation
  const bindGroup = device.createBindGroup({
    label: 'bindGroup for work buffer',
-    layout: pipeline.getBindGroupLayout(0),
+    layout: bindGroupLayout,
    entries: [
      { binding: 0, resource: workBuffer  },
    ],
  });
```

最后，在计算时依次使用这两个 pipeline

```js
  // Encode commands to do the computation
  const encoder = device.createCommandEncoder();
  const pass = encoder.beginComputePass();
-  pass.setPipeline(pipeline);
+  pass.setPipeline(pipelineTimes2);
  pass.setBindGroup(0, bindGroup);
  pass.dispatchWorkgroups(input.length);
+  pass.setPipeline(pipelinePlus3);
+  pass.dispatchWorkgroups(input.length);
  pass.end();
```

最终效果就是：
我们先乘以 2，再加 3，
而且全程只用了一个 bind group。

{{{example url="../webgpu-bind-group-layouts-multiple-pipelines.html"}}}

例子本身不算特别惊艳，
但至少它是一个简单且真正可运行的共享示例。

到底什么时候该手动创建 bind group layout，
什么时候不该，
其实取决于你自己。
比如上面这个例子里，
严格来说可能直接创建 2 个 bind group
（每个 pipeline 各一个）
反而更简单。

对于简单场景，
通常并不一定需要手动创建 bind group layout。
但随着你的 WebGPU 程序越来越复杂，
手动创建 bind group layout
很可能会变成你经常使用的一种技术。

## <a id="a-bind-group-layout-notes"></a>Bind Group Layout 备注

关于创建 `GPUBindGroupLayout`，
还有一些需要注意的点：

* ## 每个 entry 都必须声明它对应哪个 `binding`

* ## 每个 entry 都必须声明它在哪些 shader stage 可见

  上面的例子里，
  我们通常只声明了一个 visibility。
  比如，如果我们想让某个 bind group
  同时在 vertex shader 和 fragment shader 中都可用，
  那就会写成：

  ```js
     visibility: GPUShaderStage.FRAGMENT | GPUShaderStage.VERTEX
  ```

  如果希望 3 个 stage 全都可见：

  ```js
     visibility: GPUShaderStage.COMPUTE |
                 GPUShaderStage.FRAGMENT | 
                 GPUShaderStage.VERTEX
  ```

* ## 有一些默认值

  对于 `texture:`
  绑定，默认值是：

  ```js
  {
    sampleType: 'float',
    viewDimension: '2d',
    multisampled: false,
  }
  ```

  对于 `sampler:`
  绑定，默认值是：

  ```js
  {
    type: 'filtering',
  }
  ```

  这意味着，在最常见的 sampler 和 texture 使用方式下，
  你完全可以像下面这样来声明 entry

  ```js
  const bindGroupLayout = device.createBindGroupLayout({
    entries: [
      {
        binding: 0,
        visibility: GPUShaderStage.FRAGMENT,
        sampler: {},  // use the defaults
      },
      {
        binding: 1,
        visibility: GPUShaderStage.FRAGMENT,
        texture: {},  // use the defaults
      },
    ],
  });
  ```

* ## buffer entry 在可能的情况下，应该声明 `minBindingSize`

  当你声明一个 buffer binding 时，
  可以指定 `minBindingSize`。

  一个很好的例子是：
  你定义了一个用于 uniforms 的 struct。
  例如在[uniform 那篇文章](webgpu-uniforms.html)里，
  我们曾经有这样一个结构体：

  ```wgsl
  struct OurStruct {
    color: vec4f,
    scale: vec2f,
    offset: vec2f,
  };

  @group(0) @binding(0) var<uniform> ourStruct: OurStruct;
  ``` 

  它需要 32 字节，
  所以我们应该像这样声明它的 `minBindingSize`

  ```js
  const bindGroupLayout = device.createBindGroupLayout({
    entries: [
      {
        binding: 0,
        visibility: GPUShaderStage.COMPUTE,
        buffer: {
          type: 'uniform',
          minBindingSize: 32,
        },
      },
    ],
  });
  ```

  之所以要声明 `minBindingSize`，
  是因为这样 WebGPU 就能在你调用
  `createBindGroup` 时，
  直接检查 buffer 的大小 / offset 是否正确。
  如果你不设置 `minBindingSize`，
  那 WebGPU 就只能等到 draw / dispatchWorkgroups 阶段，
  再去检查当前 buffer
  对这个 pipeline 来说尺寸是否正确。
  每次 draw 都检查，
  显然会比在创建 bind group 时只检查一次更慢。

  不过另一方面，在上面那个通过 storage buffer
  去做数字翻倍之类的例子里，
  我们并没有声明 `minBindingSize`。
  那是因为，storage buffer
  在 shader 中被声明成了 `array`，
  这样根据你实际传入多少值，
  就可以绑定不同大小的 buffer。


规范中的[这一部分](https://www.w3.org/TR/webgpu/#dictdef-gpubindgrouplayoutentry)
列出了创建 bind group layout 时的全部选项。

[这篇文章](https://toji.dev/webgpu-best-practices/bind-groups)
里也有一些关于 bind group 和 bind group layout 的建议。

[这个库](https://greggman.github.io/webgpu-utils)
可以帮你自动计算 struct 大小，
以及默认的 bind group layout。
