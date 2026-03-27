Title: WebGPU 高效使用视频
Description: 如何在 WebGPU 中使用视频
TOC: 使用视频

在[上一篇文章](webgpu-importing-textures.html)中，我们介绍了
如何把图片、canvas 和视频加载到纹理中。
这篇文章会讲一种在 WebGPU 中使用视频的更高效方式。

在上一篇文章中，我们是通过调用 `copyExternalImageToTexture`
把视频数据加载到 WebGPU 纹理里的。这个函数会把视频当前帧
从视频本身复制到我们预先创建好的纹理中。

WebGPU 还提供了另一种使用视频的方法，叫做 `importExternalTexture`。
顾名思义，它会提供一个 `GPUExternalTexture`。这个外部纹理
直接表示视频中的数据本身，不需要做拷贝。[^no-copy]
你把一个视频传给 `importExternalTexture`，它就会返回一个可直接使用的纹理。

[^no-copy]: 实际上具体会发生什么，取决于浏览器实现。
WebGPU 规范设计时的目标，是让浏览器不需要
进行拷贝。

不过，使用 `importExternalTexture` 得到的纹理有几个比较大的注意事项。

* ## 这个纹理只在当前 JavaScript task 结束之前有效

  对大多数 WebGPU 应用来说，这意味着这个纹理只会在
  你的 `requestAnimationCallback` 函数结束之前存在。或者说，只在你当前触发渲染的事件内有效；
  比如 `requestVideoFrameCallback`、`setTimeout`、
  `mouseMove` 等等。一旦函数退出，这个纹理就过期了。
  如果你还想再次使用视频，就必须重新调用 `importExternalTexture`。

  这带来的一个影响是：每次调用 `importExternalTexture` 时，
  你都必须创建一个新的 bindgroup [^bindgroup-exception]，
  这样才能把新的纹理传给着色器。

  [^bindgroup-exception]: 规范实际上也说了，实现
  可以返回同一个纹理，但不是必须的。如果你
  想检查拿到的是否还是同一个纹理，可以和前一个纹理比较，
  例如 <pre><code>const newTexture = device.importExternalTexture(...);<br>const same = oldTexture === newTexture;</code></pre> 如果它确实
  是同一个纹理，那么你就可以复用已有的 bindgroup，
  继续引用 `oldTexture`。

* ## 你必须在着色器中使用 `texture_external`

  在前面所有纹理示例里，我们用的一直是 `texture_2d<f32>`，
  但通过 `importExternalTexture` 得到的纹理只能绑定到
  使用 `texture_external` 的绑定点上。

* ## 你必须在着色器中使用 `textureSampleBaseClampToEdge`

  在之前所有的纹理示例里，我们一直使用 `textureSample`，
  但来自 `importExternalTexture` 的纹理只能使用
  `textureSampleBaseClampToEdge`。[^textureLoad] 顾名思义，
  `textureSampleBaseClampToEdge` 只会采样基础纹理的 mip 级别
  （也就是 level 0）。换句话说，外部纹理不能有 mipmap。
  另外，这个函数会做边缘钳制，也就是说，即使把采样器设置成
  `addressModeU: 'repeat'` 也会被忽略。

  注意，你仍然可以自己通过 `fract` 来实现重复，例如：

  ```wgsl
  let color = textureSAmpleBaseClampToEdge(
     someExternalTexture,
     someSampler,
     fract(texcoord)
  );`
  ```

  [^textureLoad]: 你也可以对外部纹理使用 `textureLoad`。

如果这些限制不符合你的需求，那么你就需要像
[上一篇文章](webgpu-importing-textures.html)中那样继续使用
`copyExternalImageToTexture`。

下面我们做一个使用 `importExternalTexture` 的可运行示例。这里有一段视频

<div class="webgpu_center">
  <div>
     <video muted controls src="../resources/videos/pexels-anna-bondarenko-5534310 (540p).mp4" style="width: 320px";></video>
     <div class="copyright"><a href="https://www.pexels.com/video/dog-walking-outside-the-house-5534310/">by Anna Bondarenko</a></div>
  </div>
</div>

下面是从前一个示例改过来所需要的变动。

首先我们要更新着色器。

```wgsl
struct OurVertexShaderOutput {
  @builtin(position) position: vec4f,
  @location(0) texcoord: vec2f,
};

struct Uniforms {
  matrix: mat4x4f,
};

@group(0) @binding(2) var<uniform> uni: Uniforms;

@vertex fn vs(
  @builtin(vertex_index) vertexIndex : u32
) -> OurVertexShaderOutput {
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

  var vsOutput: OurVertexShaderOutput;
  let xy = pos[vertexIndex];
  vsOutput.position = uni.matrix * vec4f(xy, 0.0, 1.0);
-  vsOutput.texcoord = xy * vec2f(1, 50);
+  vsOutput.texcoord = xy;
  return vsOutput;
}

@group(0) @binding(0) var ourSampler: sampler;
-@group(0) @binding(1) var ourTexture: texture_2d<f32>;
+@group(0) @binding(1) var ourTexture: texture_external;

@fragment fn fs(fsInput: OurVertexShaderOutput) -> @location(0) vec4f {
-  return textureSample(ourTexture, ourSampler, fsInput.texcoord);
+  return textureSampleBaseClampToEdge(
+      ourTexture,
+      ourSampler,
+      fsInput.texcoord,
+  );
}
```

上面我们不再把纹理坐标乘以 50，
因为那个操作只是为了演示重复采样，而外部纹理并不支持重复。

我们也做了前面提到的必需修改。`texture_2d<f32>`
改成 `texture_external`，而 `textureSample`
改成 `textureSampleBaseClampToEdge`。

我们还移除了所有与创建纹理以及生成 mips
相关的代码。

当然，我们也需要把视频地址换掉

```js
-  video.src = 'resources/videos/Golden_retriever_swimming_the_doggy_paddle-360-no-audio.webm';
+  video.src = 'resources/videos/pexels-anna-bondarenko-5534310 (540p).mp4';
```

既然不能有 mip 级别，也就没必要再创建会用到它们的采样器了。

```js
  const objectInfos = [];
-  for (let i = 0; i < 8; ++i) {
+  for (let i = 0; i < 4; ++i) {
    const sampler = device.createSampler({
      addressModeU: 'repeat',
      addressModeV: 'repeat',
      magFilter: (i & 1) ? 'linear' : 'nearest',
      minFilter: (i & 2) ? 'linear' : 'nearest',
-      mipmapFilter: (i & 4) ? 'linear' : 'nearest',
    });

  ...
```

由于只有在调用 `importExternalTexture` 之后我们才能拿到纹理，
所以不能提前创建 bindgroup。我们会把稍后创建 bindgroup 所需的信息
先保存下来。[^bindgroups-in-advance]

[^bindgroups-in-advance]: 我们也可以把 bind group 拆开，
其中一个提前创建，用来保存 sampler 和 uniformBuffer，
另一个只引用外部纹理，在渲染时创建。
值不值得这样做，要看你的具体需求。

```js
  const objectInfos = [];
  for (let i = 0; i < 4; ++i) {

    ...

-    const bindGroups = textures.map(texture =>
-      device.createBindGroup({
-        layout: pipeline.getBindGroupLayout(0),
-        entries: [
-          { binding: 0, resource: sampler },
-          { binding: 1, resource: texture },
-          { binding: 2, resource: uniformBuffer },
-        ],
-      }));

    // Save the data we need to render this object.
    objectInfos.push({
-      bindGroups,
+     sampler,
      matrix,
      uniformValues,
      uniformBuffer,
    });
```

到了渲染阶段，我们再调用 `importExternalTexture`
并创建 bindgroup

```js
  function render() {
-    copySourceToTexture(device, texture, video);
    ...

    const encoder = device.createCommandEncoder({
      label: 'render quad encoder',
    });
    const pass = encoder.beginRenderPass(renderPassDescriptor);
    pass.setPipeline(pipeline);

+    const texture = device.importExternalTexture({source: video});

    objectInfos.forEach(({sampler, matrix, uniformBuffer, uniformValues}, i) => {
+      const bindGroup = device.createBindGroup({
+        layout: pipeline.getBindGroupLayout(0),
+        entries: [
+          { binding: 0, resource: sampler },
+          { binding: 1, resource: texture },
+          { binding: 2, resource: uniformBuffer },
+        ],
+      });

      ...

      pass.setBindGroup(0, bindGroup);
      pass.draw(6);  // call our vertex shader 6 times
    });
```

另外，既然纹理不能重复，我们也来调整一下
矩阵运算，让绘制出来的四边形更容易看清，
而不是像之前那样在 50:1 的比例下被拉得很长。

```js
  function render() {
    ...
    objectInfos.forEach(({bindGroups, matrix, uniformBuffer, uniformValues}, i) => {
      const bindGroup = bindGroups[texNdx];

      const xSpacing = 1.2;
-      const ySpacing = 0.7;
-      const zDepth = 50;
+      const ySpacing = 0.5;
+      const zDepth = 1;

-      const x = i % 4 - 1.5;
-      const y = i < 4 ? 1 : -1;
+      const x = i % 2 - .5;
+      const y = i < 2 ? 1 : -1;

      mat4.translate(viewProjectionMatrix, [x * xSpacing, y * ySpacing, -zDepth * 0.5], matrix);
-      mat4.rotateX(matrix, 0.5 * Math.PI, matrix);
-      mat4.scale(matrix, [1, zDepth * 2, 1], matrix);
+      mat4.rotateX(matrix, 0.25 * Math.PI * Math.sign(y), matrix);
+      mat4.scale(matrix, [1, -1, 1], matrix);
      mat4.translate(matrix, [-0.5, -0.5, 0], matrix);

      // copy the values from JavaScript to the GPU
      device.queue.writeBuffer(uniformBuffer, 0, uniformValues);

      pass.setBindGroup(0, bindGroup);
      pass.draw(6);  // call our vertex shader 6 times
    });

```

这样我们就得到了一个在 WebGPU 中零拷贝的视频纹理。

{{{example url="../webgpu-simple-textured-quad-external-video.html"}}}

## 为什么是 `texture_external`？

有些人可能已经注意到了：这种使用视频的方法采用的是
`texture_external`，而不是像 `texture_2d<f32>` 这样更常见的类型；
同时它使用的是 `textureSampleBaseClampToEdge`，而不是普通的 `textureSample`。
这意味着，如果你想以这种方式使用视频纹理，
又想把它和渲染流程里的其他部分混用，
你就需要不同的着色器。使用静态纹理时要用
`texture_2d<f32>` 的着色器，而想使用视频时则要用
`texture_external` 的着色器。

我觉得理解这里底层到底发生了什么很重要。

视频通常会把亮度信息（每个像素的明暗）
和色度信息（每个像素的颜色）分开传输。
而且颜色部分的分辨率通常会低于亮度部分。
一种常见的分离和编码方式是
[YUV](https://en.wikipedia.org/wiki/Y%E2%80%B2UV)，其中数据被分成
亮度（Y）和颜色信息（UV）。
这种表示方式通常压缩率也更高。

WebGPU 对外部纹理的目标，是直接使用视频原本提供的格式。
为了做到这一点，它会*假装*那里有一张视频纹理，但在实际实现中，
背后可能对应着多张纹理。比如，一张纹理保存亮度值（Y），
另一张纹理保存 UV 值。而且这些 UV 值本身也可能采用特殊的分离方式。
它们不一定是那种每个像素交错存储 2 个值的纹理，
例如

    uvuvuvuvuvuvuvuv
    uvuvuvuvuvuvuvuv
    uvuvuvuvuvuvuvuv
    uvuvuvuvuvuvuvuv
    uvuvuvuvuvuvuvuv
    uvuvuvuvuvuvuvuv

它们也可能像这样排列

    uuuuuuuu
    uuuuuuuu
    uuuuuuuu
    uuuuuuuu
    uuuuuuuu
    uuuuuuuu
    vvvvvvvv
    vvvvvvvv
    vvvvvvvv
    vvvvvvvv
    vvvvvvvv
    vvvvvvvv

也就是纹理的一部分区域里每个像素存一个 (u) 值，
另一部分区域里每个像素存一个 (v) 值。
同样地，这种方式通常也是因为更利于压缩。

当你在着色器里使用 `texture_external` 和 `textureSampleBaseClampToEdge` 时，
WebGPU 会在幕后向你的着色器注入代码，把这些
视频数据转换后返回一个 RGBA 值。它可能需要从多张纹理中采样，
也可能需要做一些纹理坐标运算，
从 2 个、3 个甚至更多位置中取出正确的数据，再转换成 RGB。

下面是上面那段视频的 Y、U 和 V 通道

<div class="webgpu_center">
  <div class="side-by-side">
    <div class="separate">
      <img src="../resources/videos/pexels-anna-bordarenko-5534310-y-channel.png" style="width: 300px;">
      <div>Y 通道（亮度）</div>
    </div>
    <div class="separate">
      <div class="side-by-side">
        <div class="separate">
          <img src="../resources/videos/pexels-anna-bordarenko-5534310-u-channel.png" style="width: 150px;">
          <div>U 通道<br>(red 鈫?yellow)</div>
        </div>
        <div class="separate">
          <img src="../resources/videos/pexels-anna-bordarenko-5534310-v-channel.png" style="width: 150px;">
          <div>V 通道<br>( blue 鈫?yellow)</div>
        </div>
      </div>
    </div>
  </div>
</div>

从效果上看，WebGPU 在这里实际上提供了一种优化。
在传统图形库里，这部分通常要你自己处理。
要么你自己写代码把 YUV 转成 RGB，
要么让操作系统替你做。然后你把数据复制到一张 RGBA 纹理里，
再把那张 RGBA 纹理当作 `texture_2d<f32>` 来使用。
这种方式更灵活。你不需要为了视频和静态纹理
分别写不同的着色器。
但它更慢，因为必须先把 YUV 纹理转换成 RGBA 纹理。

这种更慢但更灵活的方法，在 WebGPU 中仍然可用，
我们在[上一篇文章](webgpu-importing-textures.html#a-loading-video)里已经介绍过。
如果你需要这种灵活性，如果你希望视频在任何地方都能像普通纹理一样使用，
而不用为视频和静态图像分别写不同的着色器，那么就用那种方法。

WebGPU 之所以为 `texture_external` 提供这种优化，
其中一个原因是这里是 Web 平台。浏览器支持哪些视频格式，
会随着时间发生变化。
WebGPU 会替你处理这些事情；而如果你得自己写着色器把 YUV 转成 RGB，
那你还得知道视频格式不会变化，
但这并不是 Web 可以保证的事情。

这篇文章里介绍的 `texture_external` 方法，
最明显的应用场景就是各种和视频相关的功能，
比如 meet、zoom、FB messenger 一类的功能，
例如做人脸识别、添加视觉特效或者背景分离。
另一个可能的场景是，等 WebGPU 在 WebXR 中可用以后，
拿它来处理 VR 视频。

## <a id="a-web-camera"></a>使用摄像头

事实上，我们就来直接用一下摄像头。
改动非常小。

首先，我们不再指定要播放的视频。

```js
  const video = document.createElement('video');
-  video.muted = true;
-  video.loop = true;
-  video.preload = 'auto';
-  video.src = 'resources/videos/pexels-anna-bondarenko-5534310 (540p).mp4'; /* webgpufundamentals: url */
  await waitForClick();
  await startPlayingAndWaitForVideo(video);
```

然后，当用户点击播放时，我们调用 `getUserMedia` 去请求摄像头。
得到的 stream 会被设置到 video 上。
WebGPU 相关的代码完全不需要修改。

```js
  function waitForClick() {
    return new Promise(resolve => {
      window.addEventListener(
        'click',
-        () => {
+        async() => {
          document.querySelector('#start').style.display = 'none';
-          resolve();
+          try {
+            const stream = await navigator.mediaDevices.getUserMedia({
+              video: true,
+            });
+            video.srcObject = stream;
+            resolve();
+          } catch (e) {
+            fail(`could not access camera: ${e.message ?? ''}`);
+          }
        },
        { once: true });
    });
  }
```

完成！

{{{example url="../webgpu-simple-textured-quad-external-video-camera.html"}}}

如果你想把摄像头画面作为更灵活的 `texture<f32>` 类型纹理来使用，
而不是作为更高效的 `texture_external` 类型纹理，
那么也可以对
[上一篇文章中的视频示例](webgpu-importing-textures.html#a-loading-video)
做类似修改。
