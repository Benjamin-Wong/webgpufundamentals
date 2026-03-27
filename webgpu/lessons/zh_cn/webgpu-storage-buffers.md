Title: WebGPU 存储缓冲区(Storage Buffer)
Description: 向着色器中传递大量数据
TOC: 存储缓冲区（Storage Buffer）

本文介绍存储缓冲区（Storage Buffer），是[上一篇文章](webgpu-uniforms.html)的延续。

存储缓冲区在很多方面与 uniform 缓冲区相似。如果我们只是把 JavaScript 中的 `UNIFORM` 改为 `STORAGE`，把 WGSL 中的 `var<uniform>` 改为 `var<storage, read>`，上一篇文章中的示例就能直接正常工作。

实际上，在不重命名变量的情况下，区别只有以下这些：

```js
    const staticUniformBuffer = device.createBuffer({
      label: `static uniforms for obj: ${i}`,
      size: staticUniformBufferSize,
-      usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
+      usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
    });


...

    const uniformBuffer = device.createBuffer({
      label: `changing uniforms for obj: ${i}`,
      size: uniformBufferSize,
-      usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
+      usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
    });
```

在 WGSL 中修改如下：

```wsgl
      @group(0) @binding(0) var<storage, read> ourStruct: OurStruct;
      @group(0) @binding(1) var<storage, read> otherStruct: OtherStruct;
```

不做任何其他改动，效果和之前一样：

{{{example url="../webgpu-simple-triangle-storage-split-minimal-changes.html"}}}

## uniform 缓冲区与存储缓冲区的区别

uniform 缓冲区和存储缓冲区的主要区别在于：

1. uniform 缓冲区在典型的使用情况下速度更快
   这确实取决于具体用例。一个典型的应用需要绘制大量不同的内容。比如一款 3D 游戏，可能需要绘制汽车、建筑物、岩石、灌木丛、人物等……每一种都需要传递方向和材质属性，类似于上面的示例。在这种情况下，建议使用 uniform 缓冲区。

2. 存储缓冲区的大小可以比 uniform 缓冲区大得多。

    - uniform 缓冲区的最大大小默认至少为 64KiB（65536 字节）
    - 存储缓冲区的最大大小默认至少为 128MiB（134217728 字节）

    所有 WebGPU 实现都必须至少支持上述大小。我们将在[另一篇文章](webgpu-limits-and-features.html)中详细介绍如何检查和请求更大的限制。

3. 存储缓冲区可读写，uniform 缓冲区只读

    我们在[第一篇文章](webgpu-fundamentals.html)的计算着色器示例中已经看到了向存储缓冲区写入数据的例子。

## <a id="a-instancing"></a>使用存储缓冲区实现实例化绘制（Instancing）

基于上述前两点，让我们把上一个示例改为在一次绘制调用中绘制所有 100 个三角形。这*可能*是存储缓冲区的一个合适用例。我之所以说"可能"，是因为 WebGPU 和其他编程语言类似——实现同一目标有很多种方式：`array.forEach` vs `for (const elem of array)` vs `for (let i = 0; i < array.length; ++i)`，每种都有它的用途。WebGPU 也是如此，每件事都有多种实现方式。在绘制三角形时，WebGPU 只关心你从顶点着色器返回 `builtin(position)`，以及从片段着色器返回 `location(0)` 的颜色/值。[^colorAttachments]

[^colorAttachments]: 我们可以有多个颜色附件（color attachments），那时还需要为 `location(1)`、`location(2)` 等返回更多颜色/值。

我们要做的第一件事是将存储声明改为运行时大小的数组（runtime-sized arrays）。

```wgsl
-@group(0) @binding(0) var<storage, read> ourStruct: OurStruct;
-@group(0) @binding(1) var<storage, read> otherStruct: OtherStruct;
+@group(0) @binding(0) var<storage, read> ourStructs: array<OurStruct>;
+@group(0) @binding(1) var<storage, read> otherStructs: array<OtherStruct>;
```

然后修改着色器来使用这些值：

```wgsl
@vertex fn vs(
  @builtin(vertex_index) vertexIndex : u32,
+  @builtin(instance_index) instanceIndex: u32
) -> @builtin(position) {
  let pos = array(
    vec2f( 0.0,  0.5),  // top center
    vec2f(-0.5, -0.5),  // bottom left
    vec2f( 0.5, -0.5)   // bottom right
  );

+  let otherStruct = otherStructs[instanceIndex];
+  let ourStruct = ourStructs[instanceIndex];

   return vec4f(
     pos[vertexIndex] * otherStruct.scale + ourStruct.offset, 0.0, 1.0);
}
```

我们为顶点着色器添加了一个名为 `instanceIndex` 的新参数，并赋予它 `@builtin(instance_index)` 属性，这意味着它会从 WebGPU 获取当前正在绘制的"实例"编号。当我们调用 `draw` 时，可以传入第二个参数作为*实例数量*，每绘制一个实例，当前正在处理的实例编号就会传递给我们的函数。

利用 `instanceIndex`，我们就能从结构体数组中获取特定的元素。

我们还需要从正确的数组元素中获取颜色并传递给片段着色器。片段着色器无法访问 `@builtin(instance_index)`，因为这没有意义。我们可以将它作为[跨阶段变量](webgpu-inter-stage-variables.html)传递，但更常见的做法是在顶点着色器中查找颜色，然后直接将颜色传递过去。

为此，我们将使用另一个结构体，就像在[跨阶段变量](webgpu-inter-stage-variables.html)那篇文章中所做的那样。

```wgsl
+struct VSOutput {
+  @builtin(position) position: vec4f,
+  @location(0) color: vec4f,
+}

@vertex fn vs(
  @builtin(vertex_index) vertexIndex : u32,
  @builtin(instance_index) instanceIndex: u32
-) -> @builtin(position) vec4f {
+) -> VSOutput {
  let pos = array(
    vec2f( 0.0,  0.5),  // top center
    vec2f(-0.5, -0.5),  // bottom left
    vec2f( 0.5, -0.5)   // bottom right
  );

  let otherStruct = otherStructs[instanceIndex];
  let ourStruct = ourStructs[instanceIndex];

-  return vec4f(
-    pos[vertexIndex] * otherStruct.scale + ourStruct.offset, 0.0, 1.0);
+  var vsOut: VSOutput;
+  vsOut.position = vec4f(
+      pos[vertexIndex] * otherStruct.scale + ourStruct.offset, 0.0, 1.0);
+  vsOut.color = ourStruct.color;
+  return vsOut;
}

-@fragment fn fs() -> @location(0) vec4f {
-  return ourStruct.color;
+@fragment fn fs(vsOut: VSOutput) -> @location(0) vec4f {
+  return vsOut.color;
}

```

修改好 WGSL 着色器后，让我们来更新 JavaScript 代码。

设置部分如下：

```js
const kNumObjects = 100;
const objectInfos = [];

// create 2 storage buffers
const staticUnitSize =
    4 * 4 + // color is 4 32bit floats (4bytes each)
    2 * 4 + // offset is 2 32bit floats (4bytes each)
    2 * 4; // padding
const changingUnitSize = 2 * 4; // scale is 2 32bit floats (4bytes each)
const staticStorageBufferSize = staticUnitSize * kNumObjects;
const changingStorageBufferSize = changingUnitSize * kNumObjects;

const staticStorageBuffer = device.createBuffer({
    label: 'static storage for objects',
    size: staticStorageBufferSize,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
});

const changingStorageBuffer = device.createBuffer({
    label: 'changing storage for objects',
    size: changingStorageBufferSize,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
});

// offsets to the various uniform values in float32 indices
const kColorOffset = 0;
const kOffsetOffset = 4;

const kScaleOffset = 0;

{
    const staticStorageValues = new Float32Array(staticStorageBufferSize / 4);
    for (let i = 0; i < kNumObjects; ++i) {
        const staticOffset = i * (staticUnitSize / 4);

        // These are only set once so set them now
        staticStorageValues.set(
            [rand(), rand(), rand(), 1],
            staticOffset + kColorOffset
        ); // set the color
        staticStorageValues.set(
            [rand(-0.9, 0.9), rand(-0.9, 0.9)],
            staticOffset + kOffsetOffset
        ); // set the offset

        objectInfos.push({
            scale: rand(0.2, 0.5),
        });
    }
    device.queue.writeBuffer(staticStorageBuffer, 0, staticStorageValues);
}

// a typed array we can use to update the changingStorageBuffer
const storageValues = new Float32Array(changingStorageBufferSize / 4);

const bindGroup = device.createBindGroup({
    label: 'bind group for objects',
    layout: pipeline.getBindGroupLayout(0),
    entries: [
        { binding: 0, resource: staticStorageBuffer  },
        { binding: 1, resource: changingStorageBuffer  },
    ],
});
```

上面我们创建了 2 个存储缓冲区，一个用于存放 `OurStruct` 数组，另一个用于 `OtherStruct` 数组。

然后用偏移量和颜色填充 `OurStruct` 数组的值，并将数据上传到 `staticStorageBuffer`。

我们只需创建一个绑定组来引用这两个缓冲区。

新的渲染代码如下：

```js
  function render() {
    // Get the current texture from the canvas context and
    // set it as the texture to render to.
    renderPassDescriptor.colorAttachments[0].view =
        context.getCurrentTexture().createView();

    const encoder = device.createCommandEncoder();
    const pass = encoder.beginRenderPass(renderPassDescriptor);
    pass.setPipeline(pipeline);

    // Set the uniform values in our JavaScript side Float32Array
    const aspect = canvas.width / canvas.height;

-    for (const {scale, bindGroup, uniformBuffer, uniformValues} of objectInfos) {
-      uniformValues.set([scale / aspect, scale], kScaleOffset); // set the scale
-      device.queue.writeBuffer(uniformBuffer, 0, uniformValues);
-
-      pass.setBindGroup(0, bindGroup);
-      pass.draw(3);  // call our vertex shader 3 times
-    }

+    // set the scales for each object
+    objectInfos.forEach(({scale}, ndx) => {
+      const offset = ndx * (changingUnitSize / 4);
+      storageValues.set([scale / aspect, scale], offset + kScaleOffset); // set the scale
+    });
+    // upload all scales at once
+    device.queue.writeBuffer(changingStorageBuffer, 0, storageValues);
+
+    pass.setBindGroup(0, bindGroup);
+    pass.draw(3, kNumObjects);  // call our vertex shader 3 times for each instance


    pass.end();

    const commandBuffer = encoder.finish();
    device.queue.submit([commandBuffer]);
  }
```

上面的代码将绘制 `kNumObjects` 个实例。对于每个实例，WebGPU 会调用顶点着色器 3 次，`vertex_index` 分别为 0、1、2，`instance_index` 则为 0 ~ kNumObjects - 1。

{{{example url="../webgpu-simple-triangle-storage-buffer-split.html"}}}

我们只用一次绘制调用，就绘制出了所有 100 个三角形，每个三角形都有不同的缩放、颜色和偏移量。当你需要绘制同一物体的大量实例时，这就是一种可行的方法。

## 为顶点数据使用存储缓冲区

到目前为止，我们一直在着色器中直接使用硬编码的三角形。存储缓冲区的一个用例是存储顶点数据。就像上面的示例中用 `instance_index` 对存储缓冲区进行索引一样，我们也可以用 `vertex_index` 索引另一个存储缓冲区来获取顶点数据。

让我们开始吧！

```wgsl
struct OurStruct {
  color: vec4f,
  offset: vec2f,
};

struct OtherStruct {
  scale: vec2f,
};

+struct Vertex {
+  position: vec2f,
+};

struct VSOutput {
  @builtin(position) position: vec4f,
  @location(0) color: vec4f,
};

@group(0) @binding(0) var<storage, read> ourStructs: array<OurStruct>;
@group(0) @binding(1) var<storage, read> otherStructs: array<OtherStruct>;
+@group(0) @binding(2) var<storage, read> pos: array<Vertex>;

@vertex fn vs(
  @builtin(vertex_index) vertexIndex : u32,
  @builtin(instance_index) instanceIndex: u32
) -> VSOutput {
-  let pos = array(
-    vec2f( 0.0,  0.5),  // top center
-    vec2f(-0.5, -0.5),  // bottom left
-    vec2f( 0.5, -0.5)   // bottom right
-  );

  let otherStruct = otherStructs[instanceIndex];
  let ourStruct = ourStructs[instanceIndex];

  var vsOut: VSOutput;
  vsOut.position = vec4f(
-      pos[vertexIndex] * otherStruct.scale + ourStruct.offset, 0.0, 1.0);
+      pos[vertexIndex].position * otherStruct.scale + ourStruct.offset, 0.0, 1.0);
  vsOut.color = ourStruct.color;
  return vsOut;
}

@fragment fn fs(vsOut: VSOutput) -> @location(0) vec4f {
  return vsOut.color;
}
```

现在需要再设置一个存储缓冲区来存放顶点数据。首先，让我们写一个函数来生成顶点数据——一个圆。

```js
function createCircleVertices({
    radius = 1,
    numSubdivisions = 24,
    innerRadius = 0,
    startAngle = 0,
    endAngle = Math.PI * 2,
} = {}) {
    // 2 triangles per subdivision, 3 verts per tri, 2 values (xy) each.
    const numVertices = numSubdivisions * 3 * 2;
    const vertexData = new Float32Array(numSubdivisions * 2 * 3 * 2);

    let offset = 0;
    const addVertex = (x, y) => {
        vertexData[offset++] = x;
        vertexData[offset++] = y;
    };

    // 2 vertices per subdivision
    //
    // 0--1 4
    // | / /|
    // |/ / |
    // 2 3--5
    for (let i = 0; i < numSubdivisions; ++i) {
        const angle1 =
            startAngle + ((i + 0) * (endAngle - startAngle)) / numSubdivisions;
        const angle2 =
            startAngle + ((i + 1) * (endAngle - startAngle)) / numSubdivisions;

        const c1 = Math.cos(angle1);
        const s1 = Math.sin(angle1);
        const c2 = Math.cos(angle2);
        const s2 = Math.sin(angle2);

        // first triangle
        addVertex(c1 * radius, s1 * radius);
        addVertex(c2 * radius, s2 * radius);
        addVertex(c1 * innerRadius, s1 * innerRadius);

        // second triangle
        addVertex(c1 * innerRadius, s1 * innerRadius);
        addVertex(c2 * radius, s2 * radius);
        addVertex(c2 * innerRadius, s2 * innerRadius);
    }

    return {
        vertexData,
        numVertices,
    };
}
```

上面的代码用三角形来构建圆形，如下图所示：

<div class="webgpu_center"><div class="center"><div data-diagram="circle" style="width: 300px;"></div></div></div>

这样，我们就可以用它来生成圆的顶点数据并填入存储缓冲区：

```js
// setup a storage buffer with vertex data
const { vertexData, numVertices } = createCircleVertices({
    radius: 0.5,
    innerRadius: 0.25,
});
const vertexStorageBuffer = device.createBuffer({
    label: 'storage buffer vertices',
    size: vertexData.byteLength,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
});
device.queue.writeBuffer(vertexStorageBuffer, 0, vertexData);
```

然后将其添加到绑定组中：

```js
const bindGroup = device.createBindGroup({
    label: 'bind group for objects',
    layout: pipeline.getBindGroupLayout(0),
    entries: [
        { binding: 0, resource: staticStorageBuffer  },
        { binding: 1, resource: changingStorageBuffer  },
        +{ binding: 2, resource: vertexStorageBuffer  },
    ],
});
```

最后在渲染时，需要请求绘制圆中的所有顶点：

```js
-pass.draw(3, kNumObjects); // call our vertex shader 3 times for several instances
+pass.draw(numVertices, kNumObjects);
```

{{{example url="../webgpu-storage-buffer-vertices.html"}}}

上面我们使用了：

```wsgl
struct Vertex {
  pos: vec2f;
};

@group(0) @binding(2) var<storage, read> pos: array<Vertex>;
```

我们完全可以不使用结构体，直接使用 `vec2f`：

```wgsl
-@group(0) @binding(2) var<storage, read> pos: array<Vertex>;
+@group(0) @binding(2) var<storage, read> pos: array<vec2f>;
...
-pos[vertexIndex].position * otherStruct.scale + ourStruct.offset, 0.0, 1.0);
+pos[vertexIndex] * otherStruct.scale + ourStruct.offset, 0.0, 1.0);
```

但使用结构体的话，以后添加逐顶点数据就会更容易。

通过存储缓冲区传递顶点数据的方式正变得越来越流行。不过据说在一些较旧的设备上，这种方式比*传统*方式要慢，我们将在下一篇关于[顶点缓冲区](webgpu-vertex-buffers.html)的文章中介绍传统方式。

<!-- keep this at the bottom of the article -->
<script type="module" src="./webgpu-storage-buffers.js"></script>
