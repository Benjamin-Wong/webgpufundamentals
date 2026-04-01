Title: WebGPU 天空盒
Description: 用天空盒来显示天空！
TOC: 天空盒


这篇文章承接自[环境贴图那篇文章](webgpu-environment-maps.html)。

*天空盒（skybox）* 本质上是一个贴了纹理的盒子，用来表现四面八方的天空，或者更准确地说，是用来表现非常遥远的景物，包括地平线。你可以想象自己站在一个房间里，四面墙上都贴着某个风景的大幅海报，再加上一张贴在天花板上的天空海报，以及一张贴在地板上的地面海报，这就是一个天空盒。

很多 3D 游戏的做法很直接，就是创建一个立方体，把它做得非常大，再给它贴上天空纹理。

这种方法能用，但也会带来一些问题。其中一个问题是：你有一个立方体，而且你需要从多个方向去看它，也就是无论摄像机朝向哪里都要正确显示。你希望所有内容都看起来非常遥远，但又不希望立方体的角超出裁剪平面。更麻烦的是，从性能角度考虑，你通常希望先画近处的东西，再画远处的东西，这样 GPU 借助[深度纹理](webgpu-orthographic.html)就能跳过那些注定无法通过深度测试的像素。

所以理想情况下，你应该最后再绘制天空盒，并开启深度测试；但如果你真的使用一个盒子，那么随着摄像机朝向不同方向，盒子的角会比盒子的面更远，从而引发问题。

<div class="webgpu_center"><img src="resources/skybox-issues.svg" style="width: 500px"></div>

从上图你可以看出，我们必须确保立方体最远的点仍然位于视锥体内部，但正因为如此，立方体的某些边又可能会遮挡住那些我们并不希望被遮挡的物体。

典型的解决方案是关闭深度测试，并在最开始先绘制天空盒。但这样一来，我们就失去了深度测试带来的性能好处，也就是 GPU 无法跳过那些稍后会被场景中其它内容覆盖掉的像素。

与其使用一个立方体，不如直接[绘制一个覆盖整个画布的三角形](webgpu-large-triangle-to-cover-clip-space.html)，再配合[a cubemap](webgpu-cube-maps.html) 来实现。通常我们会使用 view projection 矩阵把 3D 空间中的几何体投影到屏幕上，而这里我们反过来做。我们使用 view projection 矩阵的逆矩阵，反向推导出当前正在绘制的每个像素所对应的摄像机观察方向。这样我们就能得到去 cubemap 中采样所需的方向向量。

我们从[环境贴图示例](webgpu-environment-maps.html)开始，因为它已经会加载 cubemap 并为其生成 mip。现在我们改成使用一个写死的三角形。下面是 shader：

```wgsl
struct Uniforms {
  viewDirectionProjectionInverse: mat4x4f,
};

struct VSOutput {
  @builtin(position) position: vec4f,
  @location(0) pos: vec4f,
};

@group(0) @binding(0) var<uniform> uni: Uniforms;
@group(0) @binding(1) var ourSampler: sampler;
@group(0) @binding(2) var ourTexture: texture_cube<f32>;

@vertex fn vs(@builtin(vertex_index) vNdx: u32) -> VSOutput {
  let pos = array(
    vec2f(-1, 3),
    vec2f(-1,-1),
    vec2f( 3,-1),
  );
  var vsOut: VSOutput;
  vsOut.position = vec4f(pos[vNdx], 1, 1);
  vsOut.pos = vsOut.position;
  return vsOut;
}
```

上面可以看到，我们先通过 `vsOut.position` 设置了 `@builtin(position)` 为顶点位置，并且显式把 z 设成了 1，这样这个三角形就会被绘制在最远的 z 值处。我们还把这个顶点位置继续传给了片元着色器。

在片元着色器里，我们将这个位置与逆 view projection 矩阵相乘，然后再除以 w，从 4D 空间回到 3D 空间。这一步和顶点着色器里 `@builtin(position)` 会发生的那次除法是同样的事情，只不过这次我们自己手动做。

```glsl
@fragment fn fs(vsOut: VSOutput) -> @location(0) vec4f {
  let t = uni.viewDirectionProjectionInverse * vsOut.pos;
  return textureSample(ourTexture, ourSampler, normalize(t.xyz / t.w) * vec3f(1, 1, -1));
}
```

注意：这里我们把 z 方向乘了 `-1`，原因和[上一篇文章中讲过的一样](webgpu-environment-maps.html#a-flipped)。

这个 pipeline 的 vertex 阶段不需要任何 buffer：

```js
  const pipeline = device.createRenderPipeline({
    label: 'no attributes',
    layout: 'auto',
    vertex: {
      module,
    },
    fragment: {
      module,
      targets: [{ format: presentationFormat }],
    },
    depthStencil: {
      depthWriteEnabled: true,
      depthCompare: 'less-equal',
      format: 'depth24plus',
    },
  });
```

注意这里我们把 `depthCompare` 设成了 `less-equal`，而不是 `less`。原因是我们会把深度纹理清成 1.0，而当前绘制的内容也在 1.0 上。如果不改成 `less-equal`，那么 1.0 并不小于 1.0，最终就什么也画不出来。

同样地，我们还需要设置一个 uniform buffer：

```js
  // viewDirectionProjectionInverse
  const uniformBufferSize = (16) * 4;
  const uniformBuffer = device.createBuffer({
    label: 'uniforms',
    size: uniformBufferSize,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
  });

  const uniformValues = new Float32Array(uniformBufferSize / 4);

  // offsets to the various uniform values in float32 indices
  const kViewDirectionProjectionInverseOffset = 0;

  const viewDirectionProjectionInverseValue = uniformValues.subarray(
      kViewDirectionProjectionInverseOffset,
      kViewDirectionProjectionInverseOffset + 16);
```

并在渲染时为它赋值：

```js
    const aspect = canvas.clientWidth / canvas.clientHeight;
    const projection = mat4.perspective(
        60 * Math.PI / 180,
        aspect,
        0.1,      // zNear
        10,      // zFar
    );
    // Camera going in circle from origin looking at origin
    const cameraPosition = [Math.cos(time * .1), 0, Math.sin(time * .1)];
    const view = mat4.lookAt(
      cameraPosition,
      [0, 0, 0],  // target
      [0, 1, 0],  // up
    );
    // We only care about direction so remove the translation
    view[12] = 0;
    view[13] = 0;
    view[14] = 0;

    const viewProjection = mat4.multiply(projection, view);
    mat4.inverse(viewProjection, viewDirectionProjectionInverseValue);

    // upload the uniform values to the uniform buffer
    device.queue.writeBuffer(uniformBuffer, 0, uniformValues);
```

注意上面我们在计算 `cameraPosition` 时，让摄像机围绕原点旋转。然后在得到 `view` 矩阵之后，我们把其中的平移部分清零，因为我们只关心摄像机朝向哪个方向，而不关心它具体位于哪里。

在此基础上，我们再把它和 projection 矩阵相乘，取其逆矩阵，然后把结果设置进去。

{{{example url="../webgpu-skybox.html" }}}

现在让我们把环境映射的立方体重新合并回这个示例中。
首先，先把一批变量重命名。

来自 skybox 示例的变量：

```
module -> skyBoxModule
pipeline -> skyBoxPipeline
uniformBuffer -> skyBoxUniformBuffer
uniformValues -> skyBoxUniformValues
bindGroup -> skyBoxBindGroup
```

类似地，来自环境贴图示例的变量：

```
module -> envMapModule
pipeline -> envMapPipeline
uniformBuffer -> envMapUniformBuffer
uniformValues -> envMapUniformValues
bindGroup -> envMapBindGroup
```

把这些名字改好之后，我们只需要更新渲染代码即可。首先，要同时更新两边的 uniform 值：

```js
    const aspect = canvas.clientWidth / canvas.clientHeight;
    mat4.perspective(
        60 * Math.PI / 180,
        aspect,
        0.1,      // zNear
        10,      // zFar
        projectionValue,
    );
    // Camera going in circle from origin looking at origin
    cameraPositionValue.set([Math.cos(time * .1) * 5, 0, Math.sin(time * .1) * 5]);
    const view = mat4.lookAt(
      cameraPositionValue,
      [0, 0, 0],  // target
      [0, 1, 0],  // up
    );
    // Copy the view into the viewValue since we're going
    // to zero out the view's translation
    viewValue.set(view);

    // We only care about direction so remove the translation
    view[12] = 0;
    view[13] = 0;
    view[14] = 0;
    const viewProjection = mat4.multiply(projectionValue, view);
    mat4.inverse(viewProjection, viewDirectionProjectionInverseValue);

    // Rotate the cube
    mat4.identity(worldValue);
    mat4.rotateX(worldValue, time * -0.1, worldValue);
    mat4.rotateY(worldValue, time * -0.2, worldValue);

    // upload the uniform values to the uniform buffers
    device.queue.writeBuffer(envMapUniformBuffer, 0, envMapUniformValues);
    device.queue.writeBuffer(skyBoxUniformBuffer, 0, skyBoxUniformValues);
```

然后再把两者都绘制出来。这里先画环境映射立方体，再画天空盒，用来说明“把天空盒放到最后绘制”也是可行的。

```js
    // Draw the cube
    pass.setPipeline(envMapPipeline);
    pass.setVertexBuffer(0, vertexBuffer);
    pass.setIndexBuffer(indexBuffer, 'uint16');
    pass.setBindGroup(0, envMapBindGroup);
    pass.drawIndexed(numVertices);

    // Draw the skyBox
    pass.setPipeline(skyBoxPipeline);
    pass.setBindGroup(0, skyBoxBindGroup);
    pass.draw(3);
```

{{{example url="../webgpu-skybox-plus-environment-map.html" }}}
