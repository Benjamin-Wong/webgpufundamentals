Title: WebGPU 高亮显示
Description: 高亮选中的对象
TOC: 高亮显示

这篇文章是一个短系列中的第 1 篇，
主题是如何实现 3D 编辑器中的一些功能部件。
每一篇都会建立在上一篇的基础上，
所以按顺序阅读会更容易理解。
这些文章默认你已经读过
[场景图那篇文章](webgpu-scene-graphs.html)
以及[后处理那篇文章](webgpu-post-processing.html)。

1. [高亮显示](webgpu-highlighting.html) ⬅ 你现在在这里
2. [相机控制](webgpu-camera-controls.html)
3. [拾取](webgpu-picking.html)

假设我们想做一个比较简单的 3D 编辑器，灵感来源可能类似 Blender、Maya、Unity 或 Unreal。我们希望它能让我们在 3D 场景中选中并操作对象。
其实我们在[场景图那篇文章](webgpu-scene-graphics.html)里已经在这个方向上迈出了一步：那里我们有一组节点，可以通过 UI 中的按钮选中其中一个节点，并编辑这个节点的平移、旋转和缩放。

如果我们还能在画面里直观地看出当前选中的是哪一个，就更好了。那就来实现它。

我们从[第一次加入节点选择功能的那个示例](webgpu-scene-graphs.html#a-gui)开始，当时的场景大概是这样：

<div class="webgpu_center center">
  <div data-diagram="standardPass" style="width: 600px"></div>
</div>

为了高亮当前选中的对象，我们可以把“仅被选中的对象”单独渲染到另一张纹理里。

<div class="webgpu_center center">
  <div data-diagram="selectedPass" style="width: 600px"></div>
</div>

这些 alpha 值实际上会形成被选中对象的一个剪影（silhouette）。

<div class="webgpu_center center">
  <div data-diagram="alpha" style="width: 600px"></div>
</div>

然后我们就可以把这个 alpha 蒙版作为输入，交给一个类似后处理的 pass：当当前像素的 alpha 为 0，但附近存在非 0 值时，就绘制高亮颜色。这样就能得到一个描边效果。

<div class="webgpu_center center">
  <div data-diagram="outline" style="width: 600px"></div>
</div>

下面是一个类似后处理的 shader。给它 alpha 蒙版后，它就会画出轮廓线：

```wgsl
struct VSOutput {
  @builtin(position) position: vec4f,
  @location(0) texcoord: vec2f,
};

@vertex fn vs(
  @builtin(vertex_index) vertexIndex : u32,
) -> VSOutput {
  var pos = array(
    vec2f(-1.0, -1.0),
    vec2f(-1.0,  3.0),
    vec2f( 3.0, -1.0),
  );

  var vsOutput: VSOutput;
  let xy = pos[vertexIndex];
  vsOutput.position = vec4f(xy, 0.0, 1.0);
  vsOutput.texcoord = xy * vec2f(0.5, -0.5) + vec2f(0.5);
  return vsOutput;
}

@group(0) @binding(0) var mask: texture_2d<f32>;

fn isOnEdge(pos: vec2i) -> bool {
  // Note: we need to make sure we don't use out of bounds
  // texel coordinates with textureLoad as that returns
  // different results on different GPUs
  let size = vec2i(textureDimensions(mask, 0));
  let start = max(pos - 2, vec2i(0));
  let end = min(pos + 2, size);

  for (var y = start.y; y <= end.y; y++) {
    for (var x = start.x; x <= end.x; x++) {
      let s = textureLoad(mask, vec2i(x, y), 0).a;
      if (s > 0) {
        return true;
      }
    }
  }
  return false;
};

@fragment fn fs2d(fsInput: VSOutput) -> @location(0) vec4f {
  let pos = vec2i(fsInput.position.xy);

  // Get the current texel.
  // If it's not 0 we're inside the selected objects
  let s = textureLoad(mask, pos, 0).a;
  if (s > 0) {
    discard;
  }

  let hit = isOnEdge(pos);
  if (!hit) {
    discard;
  }
  return vec4f(1, 0.5, 0, 1); // orange
}
```

这个 shader 首先检查蒙版中当前像素是否大于 0。
如果大于 0，
说明它位于蒙版内部，
也就是位于被选中的对象内部，
因此我们不想在这里绘制任何东西，于是直接 `discard`。

否则，它会调用 `isOnEdge` 去检查周围的像素。
如果周围没有任何一个像素大于 0，
那它就不在边缘上，于是同样通过 `discard` 不绘制任何内容。

否则就说明当前像素位于边缘，于是画成橙色。

现在既然已经有了 shader，我们就需要用到[后处理那篇文章](webgpu-post-processing.html)里的后处理初始化代码。

```js
  const postProcessModule = device.createShaderModule({
    code: /* wgsl */ `
      struct VSOutput {
        @builtin(position) position: vec4f,
        @location(0) texcoord: vec2f,
      };

      @vertex fn vs(
        @builtin(vertex_index) vertexIndex : u32,
      ) -> VSOutput {
        var pos = array(
          vec2f(-1.0, -1.0),
          vec2f(-1.0,  3.0),
          vec2f( 3.0, -1.0),
        );

        var vsOutput: VSOutput;
        let xy = pos[vertexIndex];
        vsOutput.position = vec4f(xy, 0.0, 1.0);
        vsOutput.texcoord = xy * vec2f(0.5, -0.5) + vec2f(0.5);
        return vsOutput;
      }

      @group(0) @binding(0) var mask: texture_2d<f32>;

      fn isOnEdge(pos: vec2i) -> bool {
        // Note: we need to make sure we don't use out of bounds
        // texel coordinates with textureLoad as that returns
        // different results on different GPUs
        let size = vec2i(textureDimensions(mask, 0));
        let start = max(pos - 2, vec2i(0));
        let end = min(pos + 2, size);

        for (var y = start.y; y <= end.y; y++) {
          for (var x = start.x; x <= end.x; x++) {
            let s = textureLoad(mask, vec2i(x, y), 0).a;
            if (s > 0) {
              return true;
            }
          }
        }
        return false;
      };

      @fragment fn fs2d(fsInput: VSOutput) -> @location(0) vec4f {
        let pos = vec2i(fsInput.position.xy);

        // get the current. If it's not 0 we're inside the selected objects
        let s = textureLoad(mask, pos, 0).a;
        if (s > 0) {
          discard;
        }

        let hit = isOnEdge(pos);
        if (!hit) {
          discard;
        }
        return vec4f(1, 0.5, 0, 1);
      }
    `,
  });

  const postProcessPipeline = device.createRenderPipeline({
    layout: 'auto',
    vertex: { module: postProcessModule },
    fragment: {
      module: postProcessModule,
      targets: [ { format: presentationFormat }],
    },
  });

-  const postProcessSampler = device.createSampler({
-    minFilter: 'linear',
-    magFilter: 'linear',
-  });

  const postProcessRenderPassDescriptor = {
    label: 'post process render pass',
    colorAttachments: [
-      { loadOp: 'clear', storeOp: 'store' },
+      { loadOp: 'load', storeOp: 'store' },
    ],
  };

-  let renderTarget;
  let postProcessBindGroup;
+  let lastPostProcessTexture;

  function setupPostProcess(texture) {
-    if (renderTarget?.width === canvasTexture.width &&
-        renderTarget?.height === canvasTexture.height) {
-      return;
-    }
-
-    renderTarget?.destroy();
-    renderTarget = device.createTexture({
-      size: canvasTexture,
-      format: 'rgba8unorm',
-      usage: GPUTextureUsage.RENDER_ATTACHMENT | GPUTextureUsage.TEXTURE_BINDING,
-    });
-    const renderTargetView = renderTarget.createView();
-    renderPassDescriptor.colorAttachments[0].view = renderTargetView;

+    if (!postProcessBindGroup || texture !== lastPostProcessTexture) {
+      lastPostProcessTexture = texture;
*      postProcessBindGroup = device.createBindGroup({
*        layout: postProcessPipeline.getBindGroupLayout(0),
*        entries: [
-          { binding: 0, resource: renderTargetView },
-          { binding: 1, resource: postProcessSampler },
-          { binding: 2, resource: postProcessUniformBuffer },
+          { binding: 0, resource: texture },
*        ],
*      });
+    }
  }

  function postProcess(encoder, srcTexture, dstTexture) {
-    device.queue.writeBuffer(
-      postProcessUniformBuffer,
-      0,
-      new Float32Array([
-        settings.affectAmount,
-        settings.bandMult,
-        settings.cellMult,
-        settings.cellBright,
-      ]),
-    );

    postProcessRenderPassDescriptor.colorAttachments[0].view = dstTexture.createView();
    const pass = encoder.beginRenderPass(postProcessRenderPassDescriptor);
    pass.setPipeline(postProcessPipeline);
    pass.setBindGroup(0, postProcessBindGroup);
    pass.draw(3);
    pass.end();
  }
```

渲染时我们也需要把这些后处理对象用起来。

```js
+  let selectedMeshes = [];

  function render() {

    ...

-    const encoder = device.createCommandEncoder();
-    const pass = encoder.beginRenderPass(renderPassDescriptor);
-    pass.setPipeline(pipeline);

    const aspect = canvas.clientWidth / canvas.clientHeight;
    const projection = mat4.perspective(
        degToRad(60), // fieldOfView,
        aspect,
        1,      // zNear
        2000,   // zFar
    );

    // Get the camera's position from the matrix we computed
    const cameraMatrix = mat4.identity();
    mat4.translate(cameraMatrix, [120, 100, 0], cameraMatrix);
    mat4.rotateY(cameraMatrix, settings.cameraRotation, cameraMatrix);
    mat4.translate(cameraMatrix, [60, 0, 300], cameraMatrix);

    // Compute a view matrix
    const viewMatrix = mat4.inverse(cameraMatrix);

    // combine the view and projection matrixes
    const viewProjectionMatrix = mat4.multiply(projection, viewMatrix);

+    const encoder = device.createCommandEncoder();
+    {
+      const pass = encoder.beginRenderPass(renderPassDescriptor);
+      pass.setPipeline(pipeline);

*      const ctx = { pass, viewProjectionMatrix };
*      root.updateWorldMatrix();
*      for (const mesh of meshes) {
*        drawMesh(ctx, mesh);
*      }
*
*      pass.end();
+    }

+    // draw selected objects to postTexture
+    {
+       if (!postTexture ||
+            postTexture.width !== canvasTexture.width)
+            postTexture.height !== canvasTexture.height) {
+         postTexture?.destroy();
+         postTexture = device.createTexture({
+          format: canvasTexture.format,
+          canvasTexture, // for size,
+          usage: GPUTextureUsage.RENDER_ATTACHMENT |
+                 GPUTextureUsage.TEXTURE_BINDING,
+         });
+       }
+      setupPostProcess(postTexture);
+
+      renderPassDescriptor.colorAttachments[0].view = postTexture.createView();
+      const pass = encoder.beginRenderPass(renderPassDescriptor);
+      pass.setPipeline(pipeline);
+
+      const ctx = { pass, viewProjectionMatrix };
+      for (const mesh of selectedMeshes) {
+        drawMesh(ctx, mesh);
+      }
+
+      pass.end();
+
+      // Draw outline based on alpha of postTexture
+      // on to the canvasTexture
+      postProcess(encoder, undefined, canvasTexture);
+    }

    const commandBuffer = encoder.finish();
    device.queue.submit([commandBuffer]);
  }
```

上面的代码先绘制原始场景。然后再把 `selectedMeshes` 绘制到 `postTexture`。接着我们把这个 `postTexture` 传给后处理代码，让它把描边画到 `canvasTexture` 上。

由于我们有两段代码都在做同一件事：当某张纹理所依赖的尺寸发生变化时，重新创建它。所以我们可以加一个辅助函数，让代码更简洁一点。

```js
+  function makeNewTextureIfSizeDifferent(texture, size, format, usage) {
+    if (!texture ||
+        texture.width !== size.width ||
+        texture.height !== size.height) {
+      texture?.destroy();
+      texture = device.createTexture({
+        format,
+        size,
+        usage,
+      });
+    }
+    return texture;
+  }

...

  function render() {
    ...

    // If we don't have a depth texture OR if its size is different
    // from the canvasTexture when make a new depth texture
-    if (!depthTexture ||
-        depthTexture.width !== canvasTexture.width ||
-        depthTexture.height !== canvasTexture.height) {
-      if (depthTexture) {
-        depthTexture.destroy();
-      }
-      depthTexture = device.createTexture({
-        size: [canvasTexture.width, canvasTexture.height],
-        format: 'depth24plus',
-        usage: GPUTextureUsage.RENDER_ATTACHMENT,
-      });
-    }
+    depthTexture = makeNewTextureIfSizeDifferent(
+      depthTexture,
+      canvasTexture, // for size
+      'depth24plus',
+      GPUTextureUsage.RENDER_ATTACHMENT,
+    );

...

    // draw selected objects to postTexture
    {
-      if (!postTexture ||
-           postTexture.width !== canvasTexture.width)
-           postTexture.height !== canvasTexture.height) {
-        postTexture?.destroy();
-        postTexture = device.createTexture({
-         format: canvasTexture.format,
-         canvasTexture, // for size,
-         usage: GPUTextureUsage.RENDER_ATTACHMENT |
-                GPUTextureUsage.TEXTURE_BINDING,
-        });
-      }
+      postTexture = makeNewTextureIfSizeDifferent(
+        postTexture,
+        canvasTexture, // for size
+        canvasTexture.format,
+        GPUTextureUsage.RENDER_ATTACHMENT |
+        GPUTextureUsage.TEXTURE_BINDING,
+      );
      setupPostProcess(postTexture);
```

最后还差一件事：我们需要一种方式来填充 `selectedMeshes`。
这件事稍微有点复杂，因为我们把所有东西都用立方体拼出来了，而且默认情况下其中一些节点是隐藏的。
在设置 `selectedMeshes` 时，我们会把这种隐藏关系也考虑进去，做法是检查一个节点的所有子节点里是否还包含更多 mesh。

```js
+  function meshUsesNode(mesh, node) {
+    if (!node) {
+      return false;
+    }
+    if (mesh.node === node) {
+      return true;
+    }
+    for (const child of node.children) {
+      if (meshUsesNode(mesh, child)) {
+        return true;
+      }
+    }
+    return false;
+  }

  const kUnelected = '\u3000'; // full-width space
  const kSelected = '➡️';
  const prefixRE = new RegExp(`^(?:${kUnelected}|${kSelected})`);

  function setCurrentSceneGraphNode(node) {
    trsUIHelper.setTRS(node.source);
    trsFolder.name(`orientation: ${node.name}`);
    trsFolder.updateDisplay();

    // Mark which node is selected.
    for (const b of nodeButtons) {
      const name = b.button.getName().replace(prefixRE, '');
      b.button.name(`${b.node === node ? kSelected : kUnelected}${name}`);
    }

+    selectedMeshes = meshes.filter(mesh => meshUsesNode(mesh, node));

+    render();
  }
```

这样一来，被选中的对象就会被高亮显示了。

{{{example url="../webgpu-highlighting.html"}}}

现在既然我们已经能高亮选中了，下一步就来实现：
[通过拖拽来移动相机](webgpu-camera-controls.html)，
而不是只能依赖 UI 里的按钮。

<!-- keep this at the bottom of the article -->
<link href="webgpu-highlighting.css" rel="stylesheet">
<script type="module" src="webgpu-highlighting.js"></script>
