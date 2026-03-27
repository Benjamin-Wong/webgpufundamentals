Title: WebGPU Uniforms
Description: 向着色器传递常量数据
TOC: Uniforms

上一篇文章介绍了[跨阶段变量](webgpu-inter-stage-variables.html)。本文将介绍 uniforms。

Uniforms 就像着色器的全局变量。你可以在执行着色器之前设置它们的值，着色器的每次迭代都会使用这些值。下次让 GPU 执行着色器时，你可以将它们设为其他值。

我们从[第一篇文章](webgpu-fundamentals.html)中的三角形示例开始，修改它来使用 uniforms。

```js
const module = device.createShaderModule({
    label: 'triangle shaders with uniforms',
    code: /* wgsl */ `
+      struct OurStruct {
+        color: vec4f,
+        scale: vec2f,
+        offset: vec2f,
+      };
+
+      @group(0) @binding(0) var<uniform> ourStruct: OurStruct;

      @vertex fn vs(
        @builtin(vertex_index) vertexIndex : u32
      ) -> @builtin(position) vec4f {
        let pos = array(
          vec2f( 0.0,  0.5),  // top center
          vec2f(-0.5, -0.5),  // bottom left
          vec2f( 0.5, -0.5)   // bottom right
        );

-        return vec4f(pos[vertexIndex], 0.0, 1.0);
+        return vec4f(
+          pos[vertexIndex] * ourStruct.scale + ourStruct.offset, 0.0, 1.0);
      }

      @fragment fn fs() -> @location(0) vec4f {
-        return vec4f(1, 0, 0, 1);
+        return ourStruct.color;
      }
    `,
});
```

首先，我们声明了一个包含 3 个成员的结构体：

```wsgl
      struct OurStruct {
        color: vec4f,
        scale: vec2f,
        offset: vec2f,
      };
```

然后，我们声明了一个 uniform 变量，类型为该结构体。变量名是 `ourStruct`，类型是 `OurStruct`。

```wsgl
      @group(0) @binding(0) var<uniform> ourStruct: OurStruct;
```

接下来，我们修改顶点着色器的返回值来使用 uniforms：

```wgsl
      @vertex fn vs(
         ...
      ) ... {
        ...
        return vec4f(
          pos[vertexIndex] * ourStruct.scale + ourStruct.offset, 0.0, 1.0);
      }
```

可以看到，我们将顶点位置乘以 scale，然后加上 offset。这样就可以设置三角形的大小并定位它。

我们还修改了片段着色器，让它返回 uniform 中的颜色：

```wgsl
      @fragment fn fs() -> @location(0) vec4f {
        return ourStruct.color;
      }
```

既然着色器已经设置好使用 uniforms，我们就需要在 GPU 上创建一个缓冲区来存放这些 uniform 值。

如果你从未处理过原始数据和内存大小，那有不少东西要学。这是一个很大的话题，因此有[一篇独立的文章](webgpu-memory-layout.html)专门介绍。如果你不知道如何在内存中布局结构体，请先去[阅读那篇文章](webgpu-memory-layout.html)，然后再回来。本文假定你已经[阅读过它](webgpu-memory-layout.html)。

读完[那篇文章](webgpu-memory-layout.html)后，我们就可以在缓冲区中填入与着色器中结构体匹配的数据了。

首先，我们创建一个缓冲区，并设置使用标志，使其可用于 uniforms，同时也可以通过复制数据来更新它。

```js
const uniformBufferSize =
    4 * 4 + // color is 4 32bit floats (4bytes each)
    2 * 4 + // scale is 2 32bit floats (4bytes each)
    2 * 4; // offset is 2 32bit floats (4bytes each)
const uniformBuffer = device.createBuffer({
    size: uniformBufferSize,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});
```

然后，我们创建一个 `TypedArray`，以便在 JavaScript 中设置值：

```js
// create a typedarray to hold the values for the uniforms in JavaScript
const uniformValues = new Float32Array(uniformBufferSize / 4);
```

然后填入结构体中 2 个之后不会改变的值。偏移量的计算方法在[内存布局文章](webgpu-memory-layout.html)中已经介绍过。

```js
// offsets to the various uniform values in float32 indices
const kColorOffset = 0;
const kScaleOffset = 4;
const kOffsetOffset = 6;

uniformValues.set([0, 1, 0, 1], kColorOffset); // set the color
uniformValues.set([-0.5, -0.25], kOffsetOffset); // set the offset
```

我们将颜色设为绿色。偏移量会将三角形向画布左侧移动 1/4，向下移动 1/8（请记住，裁剪空间从 -1 到 1，宽度为 2 个单位，因此 0.25 是 2 的 1/8）。

接下来，正如[第一篇文章中的图](webgpu-fundamentals.html#a-draw-diagram)所示，要让着色器访问我们的缓冲区，需要创建一个绑定组，并将缓冲区绑定到着色器中设置的 `@group(?)` 和 `@binding(?)` 相同的位置上。

```js
const bindGroup = device.createBindGroup({
    layout: pipeline.getBindGroupLayout(0),
    entries: [{ binding: 0, resource: uniformBuffer  }],
});
```

现在，在提交命令缓冲区之前，我们需要设置 `uniformValues` 中的剩余值，然后将这些值复制到 GPU 上的缓冲区。我们在 `render` 函数的开头完成这项工作。

```js
  function render() {
    // Set the uniform values in our JavaScript side Float32Array
    const aspect = canvas.width / canvas.height;
    uniformValues.set([0.5 / aspect, 0.5], kScaleOffset); // set the scale

    // copy the values from JavaScript to the GPU
    device.queue.writeBuffer(uniformBuffer, 0, uniformValues);
```

> 注：`writeBuffer` 是将数据复制到缓冲区的一种方法。
> [这篇文章](webgpu-copying-data.html)还介绍了其他几种方法。

我们将缩放设为一半大小，同时考虑画布的宽高比，这样无论画布大小如何变化，三角形都能保持相同的宽高比。

最后，我们需要在绘制前设置绑定组：

```js
pass.setPipeline(pipeline);
+pass.setBindGroup(0, bindGroup);
pass.draw(3); // call our vertex shader 3 times
pass.end();
```

这样，我们就得到了一个如下所示的绿色三角形：

{{{example url="../webgpu-simple-triangle-uniforms.html"}}}

对于这个三角形，在执行 draw 命令时的状态大致如下：

<div class="webgpu_center"><img src="resources/webgpu-draw-diagram-triangle-uniform.svg" style="width: 863px;"></div>

到目前为止，着色器中使用的所有数据都是硬编码的（顶点着色器中的三角形顶点位置和片段着色器中的颜色）。现在我们可以向着色器传递数据了，这意味着可以使用不同的数据多次调用 `draw`。

我们可以通过更新缓冲区，以不同的偏移、缩放和颜色在不同位置进行绘制。但需要注意的是，我们的命令会被放入命令缓冲区，直到提交后才真正执行。因此，我们**不能这样做**：

```js
// BAD!
for (let x = -1; x < 1; x += 0.1) {
    uniformValues.set([x, x], kOffsetOffset);
    device.queue.writeBuffer(uniformBuffer, 0, uniformValues);
    pass.draw(3);
}
pass.end();

// Finish encoding and submit the commands
const commandBuffer = encoder.finish();
device.queue.submit([commandBuffer]);
```

因为 `device.queue.xxx` 函数是在"队列"上立即执行的，而 `pass.xxx` 函数只是将命令编码到命令缓冲区中。当我们实际调用 `submit` 提交命令缓冲区时，缓冲区中存储的只有我们最后一次写入的值。

我们可以改成这样：

```js
// BAD! Slow!
for (let x = -1; x < 1; x += 0.1) {
    uniformValues.set([x, 0], kOffsetOffset);
    device.queue.writeBuffer(uniformBuffer, 0, uniformValues);

    const encoder = device.createCommandEncoder();
    const pass = encoder.beginRenderPass(renderPassDescriptor);
    pass.setPipeline(pipeline);
    pass.setBindGroup(0, bindGroup);
    pass.draw(3);
    pass.end();

    // Finish encoding and submit the commands
    const commandBuffer = encoder.finish();
    device.queue.submit([commandBuffer]);
}
```

上面的代码更新一个缓冲区，创建一个命令缓冲区，添加绘制一个物体的命令，然后完成并提交。这样做虽然可行，但速度很慢，原因有很多。最重要的一点是：最佳实践是在单个命令缓冲区中完成更多工作。

因此，我们可以为每个要绘制的物体创建一个 uniform 缓冲区。而且，由于缓冲区是通过绑定组间接使用的，我们也需要为每个物体创建一个绑定组。这样，就可以把所有要绘制的内容都放到一个命令缓冲区中。

让我们来实现它。

首先编写一个随机数函数：

```js
// A random number between [min and max)
// With 1 argument it will be [0 to min)
// With no arguments it will be [0 to 1)
const rand = (min, max) => {
    if (min === undefined) {
        min = 0;
        max = 1;
    } else if (max === undefined) {
        max = min;
        min = 0;
    }
    return min + Math.random() * (max - min);
};
```

现在，让我们设置一批缓冲区，使用各种颜色和偏移量，这样就可以绘制一批独立的物体了。

```js
  // offsets to the various uniform values in float32 indices
  const kColorOffset = 0;
  const kScaleOffset = 4;
  const kOffsetOffset = 6;

+  const kNumObjects = 100;
+  const objectInfos = [];
+
+  for (let i = 0; i < kNumObjects; ++i) {
+    const uniformBuffer = device.createBuffer({
+      label: `uniforms for obj: ${i}`,
+      size: uniformBufferSize,
+      usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
+    });
+
+    // create a typedarray to hold the values for the uniforms in JavaScript
+    const uniformValues = new Float32Array(uniformBufferSize / 4);
-    uniformValues.set([0, 1, 0, 1], kColorOffset);        // set the color
-    uniformValues.set([-0.5, -0.25], kOffsetOffset);      // set the offset
+    uniformValues.set([rand(), rand(), rand(), 1], kColorOffset);        // set the color
+    uniformValues.set([rand(-0.9, 0.9), rand(-0.9, 0.9)], kOffsetOffset);      // set the offset
+
+    const bindGroup = device.createBindGroup({
+      label: `bind group for obj: ${i}`,
+      layout: pipeline.getBindGroupLayout(0),
+      entries: [
+        { binding: 0, resource: uniformBuffer },
+      ],
+    });
+
+    objectInfos.push({
+      scale: rand(0.2, 0.5),
+      uniformBuffer,
+      uniformValues,
+      bindGroup,
+    });
+  }
```

我们还没有设置 scale 的值，因为它需要考虑画布的宽高比，而我们在渲染之前不知道画布的宽高比。

在渲染时，我们会使用基于正确宽高比调整后的 scale 值来更新所有缓冲区。

```js
  function render() {
-    // Set the uniform values in our JavaScript side Float32Array
-    const aspect = canvas.width / canvas.height;
-    uniformValues.set([0.5 / aspect, 0.5], kScaleOffset); // set the scale
-
-    // copy the values from JavaScript to the GPU
-    device.queue.writeBuffer(uniformBuffer, 0, uniformValues);

    // Get the current texture from the canvas context and
    // set it as the texture to render to.
    renderPassDescriptor.colorAttachments[0].view =
        context.getCurrentTexture().createView();

    const encoder = device.createCommandEncoder();
    const pass = encoder.beginRenderPass(renderPassDescriptor);
    pass.setPipeline(pipeline);

+    // Set the uniform values in our JavaScript side Float32Array
+    const aspect = canvas.width / canvas.height;

+    for (const {scale, bindGroup, uniformBuffer, uniformValues} of objectInfos) {
+      uniformValues.set([scale / aspect, scale], kScaleOffset); // set the scale
+      device.queue.writeBuffer(uniformBuffer, 0, uniformValues);
       pass.setBindGroup(0, bindGroup);
       pass.draw(3);  // call our vertex shader 3 times
+    }
    pass.end();

    const commandBuffer = encoder.finish();
    device.queue.submit([commandBuffer]);
  }
```

请再次记住，`encoder` 和 `pass` 对象只是将命令编码到命令缓冲区中。因此，当 `render` 函数执行完毕时，我们实际上按以下顺序发出了这些*命令*：

```js
device.queue.writeBuffer(...) // update uniform buffer 0 with data for object 0
device.queue.writeBuffer(...) // update uniform buffer 1 with data for object 1
device.queue.writeBuffer(...) // update uniform buffer 2 with data for object 2
device.queue.writeBuffer(...) // update uniform buffer 3 with data for object 3
...
// execute commands that draw 100 things, each with their own uniform buffer.
device.queue.submit([commandBuffer]);
```

效果如下：

{{{example url="../webgpu-simple-triangle-uniforms-multiple.html"}}}

说到这里，还有一件事值得介绍。你可以在着色器中自由引用多个 uniform 缓冲区。在上面的例子中，每次绘制时我们都更新 `scale`，然后通过 `writeBuffer` 将该对象的整个 `uniformValues` 上传到相应的 uniform 缓冲区。但实际上只有 `scale` 在变化，颜色和偏移量并没有变化，所以我们在上传颜色和偏移量上浪费了时间。

我们可以将 uniforms 拆分为：只需设置一次的 uniforms 和每次绘制都要更新的 uniforms。

```js
const module = device.createShaderModule({
    code: /* wgsl */ `
      struct OurStruct {
        color: vec4f,
-        scale: vec2f,
        offset: vec2f,
      };

+      struct OtherStruct {
+        scale: vec2f,
+      };

      @group(0) @binding(0) var<uniform> ourStruct: OurStruct;
+      @group(0) @binding(1) var<uniform> otherStruct: OtherStruct;

      @vertex fn vs(
        @builtin(vertex_index) vertexIndex : u32
      ) -> @builtin(position) vec4f {
        let pos = array(
          vec2f( 0.0,  0.5),  // top center
          vec2f(-0.5, -0.5),  // bottom left
          vec2f( 0.5, -0.5)   // bottom right
        );

        return vec4f(
-          pos[vertexIndex] * ourStruct.scale + ourStruct.offset, 0.0, 1.0);
+          pos[vertexIndex] * otherStruct.scale + ourStruct.offset, 0.0, 1.0);
      }

      @fragment fn fs() -> @location(0) vec4f {
        return ourStruct.color;
      }
    `,
});
```

然后我们需要为每个要绘制的物体创建 2 个 uniform 缓冲区：

```js
-  // create a buffer for the uniform values
-  const uniformBufferSize =
-    4 * 4 + // color is 4 32bit floats (4bytes each)
-    2 * 4 + // scale is 2 32bit floats (4bytes each)
-    2 * 4;  // offset is 2 32bit floats (4bytes each)
-  // offsets to the various uniform values in float32 indices
-  const kColorOffset = 0;
-  const kScaleOffset = 4;
-  const kOffsetOffset = 6;
+  // create 2 buffers for the uniform values
+  const staticUniformBufferSize =
+    4 * 4 + // color is 4 32bit floats (4bytes each)
+    2 * 4 + // offset is 2 32bit floats (4bytes each)
+    2 * 4;  // padding
+  const uniformBufferSize =
+    2 * 4;  // scale is 2 32bit floats (4bytes each)
+
+  // offsets to the various uniform values in float32 indices
+  const kColorOffset = 0;
+  const kOffsetOffset = 4;
+
+  const kScaleOffset = 0;

  const kNumObjects = 100;
  const objectInfos = [];

  for (let i = 0; i < kNumObjects; ++i) {
+    const staticUniformBuffer = device.createBuffer({
+      label: `static uniforms for obj: ${i}`,
+      size: staticUniformBufferSize,
+      usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
+    });
+
+    // These are only set once so set them now
+    {
-      const uniformValues = new Float32Array(uniformBufferSize / 4);
+      const uniformValues = new Float32Array(staticUniformBufferSize / 4);
      uniformValues.set([rand(), rand(), rand(), 1], kColorOffset);        // set the color
      uniformValues.set([rand(-0.9, 0.9), rand(-0.9, 0.9)], kOffsetOffset);      // set the offset

+      // copy these values to the GPU
+      device.queue.writeBuffer(staticUniformBuffer, 0, uniformValues);
    }

+    // create a typedarray to hold the values for the uniforms in JavaScript
+    const uniformValues = new Float32Array(uniformBufferSize / 4);
+    const uniformBuffer = device.createBuffer({
+      label: `changing uniforms for obj: ${i}`,
+      size: uniformBufferSize,
+      usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
+    });

    const bindGroup = device.createBindGroup({
      label: `bind group for obj: ${i}`,
      layout: pipeline.getBindGroupLayout(0),
      entries: [
        { binding: 0, resource: staticUniformBuffer },
+        { binding: 1, resource: uniformBuffer },
      ],
    });

    objectInfos.push({
      scale: rand(0.2, 0.5),
      uniformBuffer,
      uniformValues,
      bindGroup,
    });
  }
```

渲染代码无需任何改动。每个物体的绑定组都包含对它自己两个 uniform 缓冲区的引用。和之前一样，我们仍在更新 `scale`。但现在调用 `device.queue.writeBuffer` 时只上传 scale 值，而不是像之前那样上传每个物体的 `color` + `offset` + `scale`。

{{{example url="../webgpu-simple-triangle-uniforms-split.html"}}}

在这个简单的例子中，拆分成多个 uniform 缓冲区可能有些多余，但根据数据的变化频率进行拆分是很常见的做法。例如，可以有一个包含共享矩阵的 uniform 缓冲区，比如[投影矩阵、视图矩阵、摄像机矩阵](webgpu-cameras.html)。由于要绘制的所有物体通常使用相同的矩阵，我们只需创建一个缓冲区，让所有物体共用同一个 uniform 缓冲区。

此外，着色器可能引用另一个 uniform 缓冲区，只包含该物体特有的内容，如它的[世界/模型矩阵](webgpu-cameras.html)和[法线矩阵](webgpu-lighting-directional.html)。

还可以有一个 uniform 缓冲区包含材质设置，这些设置可能被多个物体共享。

我们会在讲解 3D 绘制时详细介绍这些内容。

接下来是[存储缓冲区（storage buffers）](webgpu-storage-buffers.html)。
