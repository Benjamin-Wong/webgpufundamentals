Title: WebGPU 兼容模式
Description: 在旧设备上运行
TOC: 兼容模式

WebGPU 兼容模式是一个在一定限制条件下可以在旧设备上运行的 WebGPU 版本。
其核心思想是：如果你能让应用在一些额外的限制和约束内运行，
你就可以请求一个 WebGPU 兼容模式适配器，从而让应用在更多平台上运行。

> 注意：兼容模式将在 Chrome 146（2026-02-23）中正式发布。它目前可能在你的浏览器中以实验功能的形式提供。在 [Chrome Canary](https://www.google.com/chrome/canary/) 中，从版本 136.0.7063.0（2025-03-11）起，
> 你可以通过启用 `chrome://flags/#enable-unsafe-webgpu` 中的 `"enable-unsafe-webgpu"` 标志来开启兼容模式。

要了解兼容模式能做什么，可以这样理解：
实际上*几乎*所有 WebGL2 程序都可以转换后在兼容模式下运行。

方法如下：

```js
const adapter = await navigator.gpu.requestAdapter({
  featureLevel: 'compatibility',
});
const device = await adapter.requestDevice();
```

很简单！需要注意的是，所有遵循兼容模式限制的应用，
本身也是合法的"核心（core）" WebGPU 应用，
可以在任何已经支持 WebGPU 的地方运行。

# 主要限制与约束

## 顶点着色器中可能不支持 Storage Buffers

最可能影响 WebGPU 应用的主要限制是：
约有 45% 的旧设备不支持在顶点着色器中使用 storage buffers。

我们在[storage buffers 那篇文章](webgpu-storage-buffers.html)中使用了这个特性，
那是本站的第三篇文章。在那之后我们
[改用了顶点缓冲区（vertex buffers）](webgpu-vertex-buffers.html)。
使用顶点缓冲区是常见且通用的做法，但某些场景用 storage buffers 会更简单。
例如[这个绘制线框的示例](https://webgpu.github.io/webgpu-samples/?sample=wireframe)，
它使用 storage buffers 从顶点数据生成三角形。

将顶点数据存储在 storage buffers 中，我们可以随机访问顶点数据；
而将顶点数据存储在顶点缓冲区中则无法做到这一点。
当然，总是有其他解决方案的。

## 中等级别的限制与约束

## 纹理作为 `TEXTURE_BINDING` 时只允许使用单一视图维度

在普通 WebGPU 中，你可以像这样创建一个 2d 纹理：

```js
const myTexture = device.createTexture({
  size: [width, height, 6],
  usage: ...
  format: ...
});
```

然后可以用 3 种不同的视图维度来查看它：

```js
// a view of myTexture as a 2d array with 6 layers
const as2DArray = myTexture.createView();

// view layer 3 of myTexture as a 2d texture
const as2D = myTexture.createView({
  dimension: '2d',
  baseArrayLayer: 3,
  arrayLayerCount: 1,
});

// view of myTexture as a cubemap
const asCube = myTexture.createView({
  dimension: 'cube',
});
```

在兼容模式下，你只能使用一种视图维度，
并且必须在创建纹理时就决定使用哪种视图维度。
只有 1 层的 2D 纹理默认只能用作 `'2d'` 视图，
超过 1 层的 2D 纹理默认只能用作 `'2d-array'` 视图。
如果你想使用非默认的视图，必须提前告知 WebGPU。
例如，如果你想要一个 cubemap，就必须在创建纹理时声明。

```js
const cubeTexture = device.createTexture({
  size: [width, height, 6],
  usage: ...
  format: ...
  textureBindingViewDimension: 'cube', 
});
```

注意，这个额外参数叫做 `textureBindingViewDimension`，
因为它与纹理在 `TEXTURE_BINDING` 用途下的使用有关。
你仍然可以将 cubemap 或 2d-array 的单个层作为 `RENDER_ATTACHMENT` 使用 `'2d'` 维度视图。

换句话说，在 bind group 中使用纹理时，
必须使用与之相同的视图维度。
即使 `textureBindingViewDimension` 是 `2d-array` 或 `cube`，
你仍然可以在将纹理用作渲染目标时使用 `2d` 维度。

在兼容模式下，如果在 bind group 中使用其他类型的视图访问纹理，会产生验证错误。

```js
// a view of cubeTexture as a 2d array with 6 layers
const bindGroup = device.createBindGroup({
  ...
  entries: [
    {
      binding,
      // ERROR in compatibility mode: texture is a cubemap not a 2d-array
      // (the default for a texture with more than 1 layer)
      resource: cubeTexture,
    },
  ],
})
```

```js
// view layer 3 of cubeTexture as a 2d texture
const bindGroup = device.createBindGroup({
  ...
  entries: [
    {
      binding,
      // ERROR in compatibility mode: texture is a cubemap not 2d
      resource: cubeTexture.createView({
        viewDimension: '2d',
        baseArrayLayer: 3,
        arrayLayerCount: 1,
      }),
    },
  ]
});
```

```js
// view of cubeTexture as a cubemap
const bindGroup = device.createBindGroup({
  ...
  entries: [
    {
      binding,
      // GOOD!
      resource: cubeTexture.createView({
        viewDimension: 'cube',
      }),
    },
  ],
});
```

这个限制其实影响不大。很少有程序需要对同一个纹理使用不同类型的视图。

## 调用 `texture.createView` 时，在 bindGroup 中不能选择层的子集

在核心 WebGPU 中，我们可以创建一个多层纹理：

```js
const texture = device.createTexture({
  size: [64, 128, 8],   // 8 layers,
  ...
});
```

然后选择层的子集：

```js
const bindGroup = device.createBindGroup({
  ...
  entries: [
    {
      binding,
      // ERROR  in compatibility mode - select layers 3 and 4
      resource: cubeTexture.createView({
        baseArrayLayer: 3,
        arrayLayerCount: 2,
      }),
    },
  ],
});
```

这个限制同样影响不大。很少有程序需要从纹理中选择层的子集。

## <a id="a-generating-mipmaps"></a> 在兼容模式下生成 Mipmaps

不过，上述两个限制在生成 mipmaps 时会同时遇到，
而这是一个非常常见的使用场景。

回想一下，我们在[导入图像到纹理那篇文章](webgpu-importing-textures.html#a-generating-mips-on-the-gpu)中
创建了一个基于 GPU 的 mipmap 生成器。
我们在[cube maps 那篇文章](webgpu-cube-maps.html#a-texture-helpers)中将其修改为
也能为 2d-array 和 cubemap 生成 mipmaps。
在那个版本中，我们总是用 `'2d'` 维度来查看纹理的每一层，
以便只引用纹理的某一层。
由于上述原因，这在兼容模式下行不通——
我们不能对 `'2d-array'` 或 `'cube'` 纹理使用 `'2d'` 视图，
也不能在 bind group 中选择单个层来指定要从哪一层读取数据。

为了让代码在兼容模式下工作，我们必须使用与创建纹理时相同的视图维度，
并且需要将整个纹理（包含所有层）传入，
然后在着色器中自行选择需要的层，
而不是通过 `createView` 来选择层。

那么我们来做吧！我们从[cube maps 那篇文章](webgpu-cube-maps.html#a-texture-helpers)中
`generateMips` 的代码开始。

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

                vec2f( 0.0,  0.0),  // center
                vec2f( 1.0,  0.0),  // right, center
                vec2f( 0.0,  1.0),  // center, top

                // 2st triangle
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
        for (let layer = 0; layer < texture.depthOrArrayLayers; ++layer) {
          const bindGroup = device.createBindGroup({
            layout: pipeline.getBindGroupLayout(0),
            entries: [
              { binding: 0, resource: sampler },
              {
                binding: 1,
                resource: texture.createView({
                  dimension: '2d',
                  baseMipLevel: baseMipLevel - 1,
                  mipLevelCount: 1,
                  baseArrayLayer: layer,
                  arrayLayerCount: 1,
                }),
              },
            ],
          });

          const renderPassDescriptor = {
            label: 'our basic canvas renderPass',
            colorAttachments: [
              {
                view: texture.createView({
                  dimension: '2d',
                  baseMipLevel: baseMipLevel,
                  mipLevelCount: 1,
                  baseArrayLayer: layer,
                  arrayLayerCount: 1,
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
      }

      const commandBuffer = encoder.finish();
      device.queue.submit([commandBuffer]);
    };
  })();
```

我们需要修改 WGSL，
针对每种纹理类型（`'2d'`、`'2d-array'`、`'cube'` 等）使用不同的片元着色器，
并且需要能够传入一个层索引来指定读取哪一层。

```wgsl
+const faceMat = array(
+  mat3x3f( 0,  0,  -2,  0, -2,   0,  1,  1,   1),   // pos-x
+  mat3x3f( 0,  0,   2,  0, -2,   0, -1,  1,  -1),   // neg-x
+  mat3x3f( 2,  0,   0,  0,  0,   2, -1,  1,  -1),   // pos-y
+  mat3x3f( 2,  0,   0,  0,  0,  -2, -1, -1,   1),   // neg-y
+  mat3x3f( 2,  0,   0,  0, -2,   0, -1,  1,   1),   // pos-z
+  mat3x3f(-2,  0,   0,  0, -2,   0,  1,  1,  -1));  // neg-z

struct VSOutput {
  @builtin(position) position: vec4f,
  @location(0) texcoord: vec2f,
+  @location(1) @interpolate(flat, either) baseArrayLayer: u32,
};

@vertex fn vs(
  @builtin(vertex_index) vertexIndex : u32,
+  @builtin(instance_index) baseArrayLayer: u32,
) -> VSOutput {
  var pos = array<vec2f, 3>(
    vec2f(-1.0, -1.0),
    vec2f(-1.0,  3.0),
    vec2f( 3.0, -1.0),
  );

  var vsOutput: VSOutput;
  let xy = pos[vertexIndex];
  vsOutput.position = vec4f(xy, 0.0, 1.0);
  vsOutput.texcoord = xy * vec2f(0.5, -0.5) + vec2f(0.5);
+  vsOutput.baseArrayLayer = baseArrayLayer;
  return vsOutput;
}

@group(0) @binding(0) var ourSampler: sampler;
-@group(0) @binding(1) var ourTexture: texture_2d<f32>;

+@group(0) @binding(1) var ourTexture2d: texture_2d<f32>;
@fragment fn fs2d(fsInput: VSOutput) -> @location(0) vec4f {
-  return textureSample(ourTexture, ourSampler, fsInput.texcoord);
+  return textureSample(ourTexture2d, ourSampler, fsInput.texcoord);
}

+@group(0) @binding(1) var ourTexture2dArray: texture_2d_array<f32>;
+@fragment fn fs2darray(fsInput: VSOutput) -> @location(0) vec4f {
+  return textureSample(
+    ourTexture2dArray,
+    ourSampler,
+    fsInput.texcoord,
+    fsInput.baseArrayLayer);
+}
+
+@group(0) @binding(1) var ourTextureCube: texture_cube<f32>;
+@fragment fn fscube(fsInput: VSOutput) -> @location(0) vec4f {
+  return textureSample(
+    ourTextureCube,
+    ourSampler,
+    faceMat[fsInput.baseArrayLayer] * vec3f(fract(fsInput.texcoord), 1));
+}
```

这段代码包含 3 个片元着色器，分别对应 `'2d'`、`'2d-array'`、`'cube'` 三种纹理类型。
它使用了[大三角形覆盖裁剪空间](webgpu-large-triangle-to-cover-clip-space.html)这一技术来绘制。
它还利用 `@builtin(instance_index)` 来选择层索引。
这是一种向着色器传入单个整数值的简洁方式，无需使用 uniform buffer。
当我们调用 `draw` 时，第 4 个参数是第一个实例，
该值会作为 `@builtin(instance_index)` 传递给着色器。
我们通过 `VSOutput.baseArrayLayer` 将其从顶点着色器传递到片元着色器，
在片元着色器中可以通过 `fsInput.baseArrayLayer` 引用。

cubemap 的代码将 2d-array 的层索引和归一化 UV 坐标转换为 cubemap 的 3D 坐标。
这是必要的，因为在兼容模式下，cubemap 只能以 cubemap 的方式查看。

回到 JavaScript 部分，我们需要从纹理上读取 `textureBindingViewDimension` 属性。
注意，如果**不**在兼容模式下，该值会是 undefined。
但在这种情况下，我们可以直接假定是 `'2d-array'`，
因为在普通的"核心"WebGPU 中，`'2d-array'` 应该始终有效。

```js
  const generateMips = (() => {
    let sampler;
    let module;
    const pipelineByFormat = {};

    return function generateMips(device, texture) {
+      // If the texture doesn't have a textureBindingViewDimension then use '2d-array'
+      const textureBindingViewDimension = texture.textureBindingViewDimension ?? '2d-array';
      if (!module) {
        module = device.createShaderModule({
          label: 'textured quad shaders for mip level generation',
          code: /* wgsl */ `
            const faceMat = array(
              mat3x3f( 0,  0,  -2,  0, -2,   0,  1,  1,   1),   // pos-x
              mat3x3f( 0,  0,   2,  0, -2,   0, -1,  1,  -1),   // neg-x
              mat3x3f( 2,  0,   0,  0,  0,   2, -1,  1,  -1),   // pos-y
              mat3x3f( 2,  0,   0,  0,  0,  -2, -1, -1,   1),   // neg-y
              mat3x3f( 2,  0,   0,  0, -2,   0, -1,  1,   1),   // pos-z
              mat3x3f(-2,  0,   0,  0, -2,   0,  1,  1,  -1));  // neg-z

            struct VSOutput {
              @builtin(position) position: vec4f,
              @location(0) texcoord: vec2f,
              @location(1) @interpolate(flat, either) baseArrayLayer: u32,
            };

            @vertex fn vs(
              @builtin(vertex_index) vertexIndex : u32,
              @builtin(instance_index) baseArrayLayer: u32,
            ) -> VSOutput {
              var pos = array<vec2f, 3>(
                vec2f(-1.0, -1.0),
                vec2f(-1.0,  3.0),
                vec2f( 3.0, -1.0),
              );

              var vsOutput: VSOutput;
              let xy = pos[vertexIndex];
              vsOutput.position = vec4f(xy, 0.0, 1.0);
              vsOutput.texcoord = xy * vec2f(0.5, -0.5) + vec2f(0.5);
              vsOutput.baseArrayLayer = baseArrayLayer;
              return vsOutput;
            }

            @group(0) @binding(0) var ourSampler: sampler;

            @group(0) @binding(1) var ourTexture2d: texture_2d<f32>;
            @fragment fn fs2d(fsInput: VSOutput) -> @location(0) vec4f {
              return textureSample(ourTexture2d, ourSampler, fsInput.texcoord);
            }

            @group(0) @binding(1) var ourTexture2dArray: texture_2d_array<f32>;
            @fragment fn fs2darray(fsInput: VSOutput) -> @location(0) vec4f {
              return textureSample(
                ourTexture2dArray,
                ourSampler,
                fsInput.texcoord,
                fsInput.baseArrayLayer);
            }

            @group(0) @binding(1) var ourTextureCube: texture_cube<f32>;
            @fragment fn fscube(fsInput: VSOutput) -> @location(0) vec4f {
              return textureSample(
                ourTextureCube,
                ourSampler,
                faceMat[fsInput.baseArrayLayer] * vec3f(fract(fsInput.texcoord), 1));
            }
          `,
        });

        sampler = device.createSampler({
          minFilter: 'linear',
          magFilter: 'linear',
        });
      }

    ...
```

之前我们按格式跟踪 pipeline，以便对相同格式的纹理复用 pipeline。
现在需要更新为按格式和视图维度同时跟踪。

```js
  const generateMips = (() => {
    let sampler;
    let module;
-    const pipelineByFormat = {};
+    const pipelineByFormatAndView = {};

    return function generateMips(device, texture, textureBindingViewDimension) {
      // If the texture doesn't have a textureBindingViewDimension then use '2d-array'.
      // This will be true in core webgpu mode.
      const textureBindingViewDimension = texture.textureBindingViewDimension ?? '2d-array';
      let module = moduleByViewDimension[textureBindingViewDimension];
      if (!module) {
        ...
      }

+      const id = `${texture.format}.${textureBindingViewDimension}`;

-      if (!pipelineByFormat[texture.format]) {
-        pipelineByFormat[texture.format] = device.createRenderPipeline({
-          label: 'mip level generator pipeline',
+      if (!pipelineByFormatAndView[id]) {
+        // chose an fragment shader based on the viewDimension (removes the '-' from 2d-array and cube-array)
+        const entryPoint = `fs${textureBindingViewDimension.replace(/[\W]/, '')}`;
+        pipelineByFormatAndView[id] = device.createRenderPipeline({
+          label: `mip level generator pipeline for ${textureBindingViewDimension}, format: ${texture.format}`,
          layout: 'auto',
          vertex: {
            module,
          },
          fragment: {
            module,
            entryPoint,
            targets: [{ format: texture.format }],
          },
        });
      }
-      const pipeline = pipelineByFormat[texture.format];
+      const pipeline = pipelineByFormatAndView[id];

      ...
}
```

然后，生成 mipmap 的循环需要改为使用完整的层范围，
因为兼容模式不允许选择层的子集。
我们还需要通过 draw 调用中的 instance index 来选择要读取的层。

```js
  const generateMips = (() => {

      ...

      const pipeline = pipelineByFormatAndView[id];

      for (let baseMipLevel = 1; baseMipLevel < texture.mipLevelCount; ++baseMipLevel) {
        for (let layer = 0; layer < texture.depthOrArrayLayers; ++layer) {
          const bindGroup = device.createBindGroup({
            layout: pipeline.getBindGroupLayout(0),
            entries: [
              { binding: 0, resource: sampler },
              {
                binding: 1,
                resource: texture.createView({
-                  dimension: '2d',
+                  dimension: textureBindingViewDimension,
                  baseMipLevel: baseMipLevel - 1,
                  mipLevelCount: 1,
-                  baseArrayLayer: layer,
-                  arrayLayerCount: 1,
                }),
              },
            ],
          });

          const renderPassDescriptor = {
            label: 'our basic canvas renderPass',
            colorAttachments: [
              {
                view: texture.createView({
                  dimension: '2d',
                  baseMipLevel,
                  mipLevelCount: 1,
                  baseArrayLayer: layer,
                  arrayLayerCount: 1,
                }),
                loadOp: 'clear',
                storeOp: 'store',
              },
            ],
          };

          const pass = encoder.beginRenderPass(renderPassDescriptor);
          pass.setPipeline(pipeline);
          pass.setBindGroup(0, bindGroup);
-          pass.draw(6);
+          // draw 3 vertices, 1 instance, first instance (instance_index) = layer
+          pass.draw(3, 1, 0, layer);
          pass.end();
        }
      }

      const commandBuffer = encoder.finish();
      device.queue.submit([commandBuffer]);
    };
  })();
```

至此，我们的 mipmap 生成代码可以在兼容模式下正常工作，
同时也继续兼容核心 WebGPU。

为了让示例完整运行，我们还需要做一些其他更新。

我们有一个 `createTextureFromSources` 函数，可以接收数据源并创建纹理。
它之前总是创建 `'2d'` 纹理，
因为在核心模式下可以将一个有 6 层的 `'2d'` 纹理作为 cubemap 查看。
现在我们需要让它支持传入 `textureBindingViewDimension` 和/或 `dimension`，
以便在创建纹理时告知兼容模式我们打算如何查看它。

```js
+  function textureViewDimensionToDimension(viewDimension) {
+   switch (viewDimension) {
+      case '1d': return '1d';
+      case '3d': return '3d';
+      default: return '2d';
+    }
+  }

  function createTextureFromSources(device, sources, options = {}) {
+    const viewDimension = options.dimension ??
+      getDefaultViewDimensionForTexture(options.textureBindingViewDimension);
+    const dimension = options.dimension ?? textureViewDimensionToDimension(viewDimension);
    // Assume are sources all the same size so just use the first one for width and height
    const source = sources[0];
    const texture = device.createTexture({
      format: 'rgba8unorm',
      mipLevelCount: options.mips ? numMipLevels(source.width, source.height) : 1,
      size: [source.width, source.height, sources.length],
      usage: GPUTextureUsage.TEXTURE_BINDING |
             GPUTextureUsage.COPY_DST |
             GPUTextureUsage.RENDER_ATTACHMENT,
+      dimension,
+      textureBindingViewDimension: options.textureBindingViewDimension,
    });
    copySourcesToTexture(device, texture, sources, options);
    return texture;
  }
```

另外，我们需要更新对 `createTextureFromSources` 的调用，
提前告知它我们希望创建 cubemap。

```js
  const texture = await createTextureFromSources(
-      device, faceCanvases, {mips: true, flipY: false});
+      device, faceCanvases, {mips: true, flipY: false, textureBindingViewDimension: 'cube'});
```

为了让示例在兼容模式下运行，我们需要像本文开头讲的那样请求兼容模式。

```js
async function main() {
-  const adapter = await navigator.gpu?.requestAdapter()
+  const adapter = await navigator.gpu?.requestAdapter({
+    featureLevel: 'compatibility',
+  });
  const device = await adapter?.requestDevice();

  ...
```

这样，我们的 cube map 示例就可以在兼容模式下正常运行了。

{{{example url="../webgpu-compatibility-mode-generatemips.html"}}}

现在你有了一个兼容模式友好的 `generateMips`，
可以在本站的任何示例中使用。它同时兼容核心模式和兼容模式。
在兼容模式下，如果你想要 cubemap 或者 1 层的 2d-array，
必须传入 `textureBindingViewDimension`。
在核心 WebGPU 下，传不传都可以，没有影响。

# 次要限制与约束

以下限制与约束是大多数程序*不太可能*遇到的：

* ## 所有颜色目标的混合设置必须一致

  在核心模式下，创建渲染管线时，每个颜色目标可以指定不同的混合设置。
  我们在[混合与透明那篇文章](webgpu-transparency.html)中使用过混合设置。
  在兼容模式下，单个管线中所有颜色目标的混合设置必须相同。

* ## `copyTextureToBuffer` 和 `copyTextureToTexture` 不支持压缩纹理

* ## `copyTextureToTexture` 不支持多重采样纹理

* ## 不支持 `cube-array`

* ## 同一次 draw/dispatch 调用中，纹理的视图不能在 aspect 或 mip level 上有所区别

  在核心 WebGPU 中，你可以为同一纹理创建指向不同 mip level 的多个视图，
  并在同一 draw 调用中使用它们。这种用法并不常见。
  注意，该限制针对的是 `TEXTURE_BINDING` 用途，即通过 bindGroup 使用纹理。
  如我们在上面的 mipmap 生成代码中所做的，
  你仍然可以将不同视图用作 `RENDER_ATTACHMENT`。

* ## 不支持 `@builtin(sample_mask)` 和 `@builtin(sample_index)`

* ## `rg32uint`、`rg32sint` 和 `rg32float` 纹理格式不能用作 storage texture

* ## `depthClampBias` 必须为 0

  这是创建渲染管线时的一个设置项。

* ## 不支持 `@interpolation(linear)` 和 `@interpolation(..., sample)`

  这些在[跨阶段变量那篇文章](webgpu-inter-stage-variables.html#a-interpolate)中有简要介绍。

* ## <a id="flat"></a> 不支持 `@interpolate(flat)` 和 `@interpolate(flat, first)`

  在兼容模式下，当你需要平坦插值时，必须使用 `@interpolate(flat, either)`。
  `either` 表示传递给片元着色器的值可以来自被绘制三角形或线段的第一个或最后一个顶点，
  由具体实现决定。

  大多数情况下这不会有影响。
  将平坦插值数据从顶点着色器传递到片元着色器，
  最常见的用途通常是每个模型、每个材质或每个实例的值。
  例如，上面的 mipmap 生成代码就使用平坦插值将 `instance_index` 传递给片元着色器。
  对于三角形的所有顶点来说该值都相同，
  因此使用 `@interpolate(flat, either)` 完全没问题。

* ## 纹理格式不能被重新解释

  在核心 WebGPU 中，你可以创建一个 `'rgba8unorm'` 纹理，
  然后将其作为 `'rgba8unorm-srgb'` 纹理查看，反之亦然，
  以及其他 `'-srgb'` 格式与对应的非 `'-srgb'` 格式之间的互转。
  兼容模式不允许这样做。纹理创建时使用的格式，就是它唯一可以被使用的格式。

* ## 不支持 `bgra8unorm-srgb`

* ## `rgba16float` 和 `r32float` 纹理不能进行多重采样

* ## 所有整型纹理格式都不能进行多重采样

* ## `depthOrArrayLayers` 必须与 `textureBindingViewDimension` 兼容

  这意味着标记了 `textureBindingViewDimension: '2d'` 的纹理必须有 `depthOrArrayLayers: 1`（默认值）。
  标记了 `textureBindingViewDimension: 'cube'` 的纹理必须有 `depthOrArrayLayers: 6`。

* ## `textureLoad` 不支持深度纹理

  "深度纹理"是指在 WGSL 中用 `texture_depth`、
  `texture_depth_2d_array` 或 `texture_depth_cube` 引用的纹理。
  在兼容模式下，这些类型不能与 `textureLoad` 一起使用。

  另一方面，`textureLoad` 可以与 `texture_2d<f32>`、`texture_2d_array<f32>` 和
  `texture_cube<f32>` 一起使用，并且使用深度格式的纹理可以绑定到这些绑定上。

* ## 深度纹理不能与非比较采样器一起使用

  同样，"深度纹理"是指在 WGSL 中用 `texture_depth`、
  `texture_depth_2d_array` 或 `texture_depth_cube` 引用的纹理。
  在兼容模式下，这些类型不能与非比较采样器一起使用。

  这实际上意味着在兼容模式下，`texture_depth`、`texture_depth_2d_array`
  和 `texture_depth_cube` 只能与 `textureSampleCompare`、
  `textureSampleCompareLevel` 和 `textureGatherCompare` 配合使用。

  另一方面，你可以将使用深度格式的纹理绑定到 `texture_2d<f32>`、
  `texture_2d_array<f32>` 和 `texture_cube<f32>` 绑定，
  但要遵守只能使用非过滤采样器的普通限制。

* ## 不支持精细导数（fine derivatives）

  WGSL 函数 `dpdxFine`、`dpdyFine` 和 `fwidthFine` 在兼容模式下不受支持。
  你仍然可以使用 `dpdx`、`dpdxCoarse`、`dpdy`、`dpdyCoarse`、`fwidth` 和 `fwidthCoarse`。

* ## 纹理与采样器的组合更加受限

  在核心模式下，你可以绑定 16 个以上的纹理和 16 个以上的采样器，
  然后在着色器中使用 256 种以上的组合。

  在兼容模式下，单个着色器阶段中只能使用 16 种组合。

  实际规则略微复杂，以下是伪代码形式的详细说明：

  ```
  maxCombinationsPerStage =
     min(device.limits.maxSampledTexturesPerShaderStage, device.limits.maxSamplersPerShaderStage)
  for each stage of the pipeline:
    sum = 0
    for each texture binding in the pipeline layout which is visible to that stage:
      sum += max(1, number of texture sampler combos for that texture binding)
    for each external texture binding in the pipeline layout which is visible to that stage:
      sum += 1 // for LUT texture + LUT sampler
      sum += 3 * max(1, number of external_texture sampler combos) // for Y+U+V
    if sum > maxCombinationsPerStage
      generate a validation error.
  ```

* ## 兼容模式下部分默认限制更低

  | 限制项                              | 兼容模式 | 核心模式  |
  | :---------------------------------- | ------: | --------: |
  | `maxColorAttachments`               |       4 |         8 |
  | `maxComputeInvocationsPerWorkgroup` |     128 |       256 |
  | `maxComputeWorkgroupSizeX`          |     128 |       256 |
  | `maxComputeWorkgroupSizeY`          |     128 |       256 |
  | `maxInterStageShaderVariables`      |      15 |        16 |
  | `maxTextureDimension1D`             |    4096 |      8192 |
  | `maxTextureDimension2D`             |    4096 |      8192 |
  | `maxUniformBufferBindingSize`       |   16384 |     65536 |
  | `maxVertexAttributes`        | 16<sup>a</sup> |        16 |

  (a) 在兼容模式下，使用 `@builtin(vertex_index)` 和/或 `@builtin(instance_index)` 各算作一个 attribute。

  当然，适配器可能对上述任何限制支持更高的值。

* ## 新增 4 个限制项

  * `maxStorageBuffersInVertexStage`（默认值 0）
  * `maxStorageTexturesInVertexStage`（默认值 0）
  * `maxStorageBuffersInFragmentStage`（默认值 4）
  * `maxStorageTexturesInFragmentStage`（默认值 4）

  与其他限制一样，你可以在请求适配器时查看适配器所支持的值，
  并在需要时请求高于默认值的限制。

  如前所述，约 45% 的设备在顶点着色器中支持 `0` 个
  storage buffers 和 storage textures。

# 从兼容模式升级到核心模式

兼容模式被设计为一种可选的模式。
如果你能让应用在上述限制内工作，就可以请求兼容模式。
如果不行，可以请求核心模式（默认），
如果设备不支持核心模式，则不会返回适配器。

另一方面，你也可以将应用设计为：
在兼容模式下也能正常工作，
但如果用户的设备支持核心 WebGPU，就充分利用所有核心特性。

要实现这一点，先请求兼容模式适配器，然后检查并启用 `core-features-and-limits` 特性。
如果适配器上存在该特性，并且你在请求设备时要求了它，
则该设备将是核心设备，上述所有限制都不再适用。

示例：

```js
const adapter = await navigator.gpu.requestAdapter({
  featureLevel: 'compatibility',
});
const hasCore = adapter.features.has('core-features-and-limits');
const device = await adapter.requestDevice({
  requiredFeatures: [
    ...(hasCore ? ['core-features-and-limits'] : []),
  ],
});
```

如果 `hasCore` 为 true，则以上所有限制和约束均不适用。

注意，其他希望检查设备是核心模式还是兼容模式的代码，
应当检查设备的 features。

```js
const isCore = device.features.has('core-features-and-limits');
```

在核心设备上，这将始终为 true。

# 测试兼容模式

在支持兼容模式的浏览器上，你可以通过不请求 `'core-features-and-limits'`（如本文开头所示）
来测试应用是否遵循了相关限制。
你可能还想确认自己确实处于兼容模式，
以便知道限制和约束正在被强制执行。

```js
const adapter = await navigator.gpu.requestAdapter({
  featureLevel: 'compatibility',
});
const device = await adapter.requestDevice();

const isCompatibilityMode = !device.features.has('core-features-and-limits');
```

这是一种很好的测试方式，可以验证你的应用是否能在这些旧设备上运行。

# 通过 webgpu-dev-extension 快速测试

使用 [webgpu-dev-extension](https://github.com/greggman/webgpu-dev-extension)，
你可以强制应用使用兼容模式进行快速测试，而无需修改应用代码。
你也可以测试一个自动升级到核心 WebGPU 的应用，
验证它在获得兼容模式时是否仍能正常工作。

步骤：

1. 打开 DevTools 并运行你的应用
2. 在 DevTools 中，打开设置

   <div class="webgpu_left"><img src="resources/images/webgpu-devtools-settings.png" style="width: 554px"></div>

3. 开启"Custom Formatters"

   <div class="webgpu_left"><img src="resources/images/webgpu-devtools-custom-formatters.png" style="width: 554px"></div>

4. 在 WebGPU-Dev-Extension 中，选择以下选项：

   <div class="webgpu_left"><img src="resources/images/webgpu-dev-extension-compat.png" style="width: 274px"></div>

    * ### Force Mode: 'compatibility-mode'

      这会让应用执行 `navigator.gpu.requestAdapter({ featureLevel: 'compatibility' });`

      如果你的应用已经支持兼容模式，则保持默认即可。

    * ### Block Features 'core-features-and-limits'

      这会阻止应用请求核心模式。

    * ### DevTools Custom Formatters

      这使得在 DevTools 中检查设备时，
      `device.features` 会显示为字符串数组。
      否则 DevTools 会显示一个不透明对象，你无法看到具体的 features。

    * ### Show Adapter Info

      这个选项会在每次创建新适配器或设备时执行 `console.log(adapter)` 和 `console.log(device)`，
      让你可以验证设备是否处于兼容模式。
      你可以查看 `device.features`，确认其中不包含 `'core-features-and-limits'`。

5. 刷新页面
6. 验证应用是否在兼容模式下运行

   在 JavaScript 控制台中，你应该看到类似下面的内容：

<div class="webgpu_center"><img src="resources/images/webgpu-compat-verification.png" style="width: 1100px" class="nobg"></div>

   在顶部附近寻找 `webgpu-dev-extension: custom-formatters`，确认格式化器已注入页面。

   然后，找到 `GPUDevice` 并展开 `features`。确保你**看不到** `"core-features-and-limits"`。

# 示例

截至 2026-02-01，[webgpu-samples](https://webgpu.github.io/webgpu-samples) 上的所有本地示例，
以及 [threejs.org/examples](https://threejs.org/examples/) 上 193 个 WebGPU 示例中的 185 个，
均可在兼容模式下运行。
剩余的 8 个可能在未来经过少量调整后也能支持兼容模式。
