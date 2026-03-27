Title: WebGPU 立方体贴图
Description: 如何在 WebGPU 中使用立方体贴图
TOC: Cube Maps

这篇文章默认你已经读过[纹理那篇文章](webgpu-textures.html)以及[将图像导入纹理那篇文章](webgpu-importing-textures.html)。
本文还会用到[方向光那篇文章](webgpu-lighting-directional.html)里介绍过的一些概念。
如果你还没读过这些文章，建议先看一下。

在[之前的一篇文章](webgpu-textures.html)中，我们讲过如何使用纹理，
讲过如何通过从左到右、从下到上都在 0 到 1 之间变化的纹理坐标来引用纹理，
也讲过纹理如何通过 mip 进行可选过滤。

另一种纹理类型叫做 *cubemap*（立方体贴图）。一个 cubemap 由 6 个面组成，
分别对应立方体的 6 个面。和传统只有 2 个维度的纹理坐标不同，
cubemap 使用的是法线，换句话说，是一个 3D 方向。
根据这个法线指向的方向，会先选中立方体的 6 个面之一，
然后在那个面内部采样像素，最终得到颜色。

让我们做一个简单示例。我们会用 2D canvas
来生成这 6 个面的图像。

下面这段代码会用某种颜色填充一个 canvas，并在中心绘制一段文字

```js
function generateFace(size, {faceColor, textColor, text}) {
  const canvas = document.createElement('canvas');
  canvas.width = size;
  canvas.height = size;
  const ctx = canvas.getContext('2d');
  ctx.fillStyle = faceColor;
  ctx.fillRect(0, 0, size, size);
  ctx.font = `${size * 0.7}px sans-serif`;
  ctx.fillStyle = textColor;
  ctx.textAlign = 'left';
  ctx.textBaseline = 'top';
  const m = ctx.measureText(text);
  ctx.fillText(
    text,
    (size - m.actualBoundingBoxRight + m.actualBoundingBoxLeft) / 2,
    (size - m.actualBoundingBoxDescent + m.actualBoundingBoxAscent) / 2
  );
  return canvas;
}
```

下面这段代码会调用它，生成 6 张图片

```js
const faceSize = 128;
const faceCanvases = [
  { faceColor: '#F00', textColor: '#0FF', text: '+X' },
  { faceColor: '#FF0', textColor: '#00F', text: '-X' },
  { faceColor: '#0F0', textColor: '#F0F', text: '+Y' },
  { faceColor: '#0FF', textColor: '#F00', text: '-Y' },
  { faceColor: '#00F', textColor: '#FF0', text: '+Z' },
  { faceColor: '#F0F', textColor: '#0F0', text: '-Z' },
].map(faceInfo => generateFace(faceSize, faceInfo));

// show the results
for (const canvas of faceCanvases) {
  document.body.appendChild(canvas);
}
```

{{{example url="../webgpu-cube-faces.html" }}}

现在我们来把这些图像通过 cubemap 应用到一个立方体上。我们将从
[将图像导入纹理那篇文章](webgpu-importing-textures.html#a-texture-atlases)
里的纹理图集示例开始改。

首先我们要把着色器改成使用 cube map

```wgsl
struct Uniforms {
  matrix: mat4x4f,
};

struct Vertex {
  @location(0) position: vec4f,
-  @location(1) texcoord: vec2f,
};

struct VSOutput {
  @builtin(position) position: vec4f,
-  @location(0) texcoord: vec2f,
+  @location(0) normal: vec3f,
};

...

@vertex fn vs(vert: Vertex) -> VSOutput {
  var vsOut: VSOutput;
  vsOut.position = uni.matrix * vert.position;
-  vsOut.texcoord = vert.texcoord;
+  vsOut.normal = normalize(vert.position.xyz);
  return vsOut;
}
```

我们去掉了着色器中的纹理坐标，
并把跨阶段变量改成向片元着色器传递法线。
由于这个立方体的顶点位置正好都是以原点为中心对称分布的，
所以我们可以直接把这些位置当作法线来用。

回忆一下在[光照那篇文章](webgpu-lighting-directional.html)里提到的内容：
法线表示的是方向，通常用来指定某个顶点所在表面的方向。
因为这里我们直接使用了归一化后的顶点位置作为法线，
所以如果现在给它加光照，立方体表面会呈现出平滑光照，
而不是硬边立方体那种面与面之间明显分开的效果。

{{{diagram url="resources/cube-normals.html" caption="标准立方体法线 vs 这个立方体的法线" width="700" height="400"}}}

既然我们不再使用纹理坐标，
那就可以删掉所有与设置纹理坐标相关的代码。

```js
  const vertexData = new Float32Array([
-     // front face     select the top left image
-    -1,  1,  1,        0   , 0  ,
-    -1, -1,  1,        0   , 0.5,
-     1,  1,  1,        0.25, 0  ,
-     1, -1,  1,        0.25, 0.5,
-     // right face     select the top middle image
-     1,  1, -1,        0.25, 0  ,
-     1,  1,  1,        0.5 , 0  ,
-     1, -1, -1,        0.25, 0.5,
-     1, -1,  1,        0.5 , 0.5,
-     // back face      select to top right image
-     1,  1, -1,        0.5 , 0  ,
-     1, -1, -1,        0.5 , 0.5,
-    -1,  1, -1,        0.75, 0  ,
-    -1, -1, -1,        0.75, 0.5,
-    // left face       select the bottom left image
-    -1,  1,  1,        0   , 0.5,
-    -1,  1, -1,        0.25, 0.5,
-    -1, -1,  1,        0   , 1  ,
-    -1, -1, -1,        0.25, 1  ,
-    // bottom face     select the bottom middle image
-     1, -1,  1,        0.25, 0.5,
-    -1, -1,  1,        0.5 , 0.5,
-     1, -1, -1,        0.25, 1  ,
-    -1, -1, -1,        0.5 , 1  ,
-    // top face        select the bottom right image
-    -1,  1,  1,        0.5 , 0.5,
-     1,  1,  1,        0.75, 0.5,
-    -1,  1, -1,        0.5 , 1  ,
-     1,  1, -1,        0.75, 1  ,
+     // front face
+    -1,  1,  1,
+    -1, -1,  1,
+     1,  1,  1,
+     1, -1,  1,
+     // right face
+     1,  1, -1,
+     1,  1,  1,
+     1, -1, -1,
+     1, -1,  1,
+     // back face
+     1,  1, -1,
+     1, -1, -1,
+    -1,  1, -1,
+    -1, -1, -1,
+    // left face
+    -1,  1,  1,
+    -1,  1, -1,
+    -1, -1,  1,
+    -1, -1, -1,
+    // bottom face
+     1, -1,  1,
+    -1, -1,  1,
+     1, -1, -1,
+    -1, -1, -1,
+    // top face
+    -1,  1,  1,
+     1,  1,  1,
+    -1,  1, -1,
+     1,  1, -1,
  ]);

  ...

  const pipeline = device.createRenderPipeline({
    label: '2 attributes',
    layout: 'auto',
    vertex: {
      module,
      buffers: [
        {
-          arrayStride: (3 + 2) * 4, // (3+2) floats 4 bytes each
+          arrayStride: (3) * 4, // (3) floats 4 bytes each
          attributes: [
            {shaderLocation: 0, offset: 0, format: 'float32x3'},  // position
-            {shaderLocation: 1, offset: 12, format: 'float32x2'},  // texcoord
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

在片元着色器中，我们需要使用 `texture_cube`
而不是 `texture_2d`。
并且当 `textureSample` 用在 `texture_cube` 上时，
它接收的是一个 `vec3f` 方向，
所以我们要传入法线。由于法线是一个跨阶段变量、会发生插值，
因此还需要再做一次归一化。

```wgsl
@group(0) @binding(0) var<uniform> uni: Uniforms;
@group(0) @binding(1) var ourSampler: sampler;
-@group(0) @binding(2) var ourTexture: texture_2d<f32>;
+@group(0) @binding(2) var ourTexture: texture_cube<f32>;

@fragment fn fs(vsOut: VSOutput) -> @location(0) vec4f {
-  return textureSample(ourTexture, ourSampler, vsOut.texcoord);
+  return textureSample(ourTexture, ourSampler, normalize(vsOut.normal));
}
```

要真正创建一个 cube map，我们需要创建一个带有 6 层的 2D 纹理。
下面把所有辅助函数都改一下，让它们可以处理多个 source。

## <a id="a-texture-helpers"></a>让纹理辅助函数支持多层

首先，把 `createTextureFromSource` 改成 `createTextureFromSources`，
让它接收一个 source 数组

```js
-  function createTextureFromSource(device, source, options = {}) {
+  function createTextureFromSources(device, sources, options = {}) {
+    // Assume are sources all the same size so just use the first one for width and height
+    const source = sources[0];
    const texture = device.createTexture({
      format: 'rgba8unorm',
      mipLevelCount: options.mips ? numMipLevels(source.width, source.height) : 1,
-      size: [source.width, source.height],
+      size: [source.width, source.height, sources.length],
      usage: GPUTextureUsage.TEXTURE_BINDING |
             GPUTextureUsage.COPY_DST |
             GPUTextureUsage.RENDER_ATTACHMENT,
    });
-    copySourceToTexture(device, texture, source, options);
+    copySourcesToTexture(device, texture, sources, options);
    return texture;
  }
```

上面的代码会创建一个带多层的纹理，每个 source 对应一层。
它还假设所有 source 的尺寸都相同。这个假设通常是合理的，
因为同一张纹理的不同层尺寸不一致的情况其实非常少见。

接下来，我们需要更新 `copySourceToTexture`，让它也支持多个 source。

```js
-  function copySourceToTexture(device, texture, source, {flipY} = {}) {
+  function copySourcesToTexture(device, texture, sources, {flipY} = {}) {
+    sources.forEach((source, layer) => {
*      device.queue.copyExternalImageToTexture(
*        { source, flipY, },
-        { texture },
+        { texture, origin: [0, 0, layer] },
*        { width: source.width, height: source.height },
*      );
+  });

    if (texture.mipLevelCount > 1) {
      generateMips(device, texture);
    }
  }
```

上面最大的变化只有一个：我们添加了一个循环来遍历所有 source，
并设置了 `origin`，指定每个 source 在纹理中的复制位置，
从而把每个 source 复制到它对应的那一层。

接下来还要更新 `generateMips`，让它支持多层 source。

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
          magFilter: 'linear',
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
+        for (let layer = 0; layer < texture.depthOrArrayLayers; ++layer) {
*          const bindGroup = device.createBindGroup({
*            layout: pipeline.getBindGroupLayout(0),
*            entries: [
*              { binding: 0, resource: sampler },
-              { binding: 1, resource: texture.createView({baseMipLevel, mipLevelCount: 1}) },
+              {
+                binding: 1,
+                resource: texture.createView({
+                  dimension: '2d',
+                  baseMipLevel: baseMipLevel - 1,
+                  mipLevelCount: 1,
+                  baseArrayLayer: layer,
+                  arrayLayerCount: 1,
+                }),
*              },
*            ],
*          });
*
*          const renderPassDescriptor = {
*            label: 'our basic canvas renderPass',
*            colorAttachments: [
*              {
-                view: texture.createView({baseMipLevel, mipLevelCount: 1}),
+                view: texture.createView({
+                  dimension: '2d',
+                  baseMipLevel: baseMipLevel,
+                  mipLevelCount: 1,
+                  baseArrayLayer: layer,
+                  arrayLayerCount: 1,
+                }),
*                loadOp: 'clear',
*                storeOp: 'store',
*              },
*            ],
*          };
*
*          const pass = encoder.beginRenderPass(renderPassDescriptor);
*          pass.setPipeline(pipeline);
*          pass.setBindGroup(0, bindGroup);
*          pass.draw(6);  // call our vertex shader 6 times
*          pass.end();
+        }
+      }

      const commandBuffer = encoder.finish();
      device.queue.submit([commandBuffer]);
    };
  })();
```

我们添加了一层循环，用来处理纹理的每一层。
我们还修改了 view，让它们只选择单独一层。
另外，我们必须显式指定 `dimension: '2d'`，
因为默认情况下，对于一个有多层的 2D 纹理，
`createView` 得到的 view 维度会是 `2d-array`。
而在生成 mipmap 这个场景下，这并不是我们想要的。

> 注意：[兼容模式那篇文章](webgpu-compatibility-mode.html)
> 提供了一个能在 compatibility mode 下工作的 `generateMips` 版本。

虽然我们这里不会再用到它们，但原来的 `createTextureFromSource`
和 `copySourceToTexture` 实际上很容易改写成

```js
  function copySourceToTexture(device, texture, source, options = {}) {
    copySourcesToTexture(device, texture, [source], options);
  }

  function createTextureFromSource(device, source, options = {}) {
    return createTextureFromSources(device, [source], options);
  }
```

现在这些辅助函数准备好了，我们就可以使用文章开头创建的那些面图像了

```js
  const texture = await createTextureFromSources(
      device, faceCanvases, {mips: true, flipY: false});
```

剩下要做的，就是在 bindGroup 里修改纹理的 view

```js
  const bindGroup = device.createBindGroup({
    label: 'bind group for object',
    layout: pipeline.getBindGroupLayout(0),
    entries: [
      { binding: 0, resource: uniformBuffer },
      { binding: 1, resource: sampler },
-      { binding: 2, resource: texture },
+      { binding: 2, resource: texture.createView({dimension: 'cube'}) },
    ],
  });
```

然后，啪，就好了

{{{example url="../webgpu-cube-map.html" }}}

注意纹理各层对应的面顺序

* layer 0 => positive x
* layer 1 => negative x
* layer 2 => positive y
* layer 3 => negative y
* layer 4 => positive z
* layer 5 => negative z

换个角度理解就是：如果你调用 `textureSample`，
并传入对应的方向，它就会返回纹理那一层中心像素的颜色。

* `textureSample(tex, sampler, vec3f( 1, 0, 0))` => 第 0 层的中心
* `textureSample(tex, sampler, vec3f(-1, 0, 0))` => 第 1 层的中心
* `textureSample(tex, sampler, vec3f( 0, 1, 0))` => 第 2 层的中心
* `textureSample(tex, sampler, vec3f( 0,-1, 0))` => 第 3 层的中心
* `textureSample(tex, sampler, vec3f( 0, 0, 1))` => 第 4 层的中心
* `textureSample(tex, sampler, vec3f( 0, 0,-1))` => 第 5 层的中心

把 cubemap 用来给立方体贴图，**并不是** cubemap 的典型用途。
给立方体贴图更“正确”或者说更标准的方法，
是像我们[前面提到的](webgpu-importing-textures.html#a-texture-atlases)那样使用纹理图集。
这篇文章的重点是介绍 cube map 这个概念，
并演示你如何给它传入方向（法线），然后它会返回立方体在那个方向上的颜色。

现在我们已经知道 cubemap 是什么、以及如何设置它，
那么 cubemap 通常是用来做什么的呢？
它最常见的用途大概就是作为
[*环境贴图*](webgpu-environment-maps.html)。
