Title: WebGPU 缩放
Description: 缩放一个对象
TOC: 缩放

本文是系列文章的第 3 篇，希望能帮助你学习 3D 数学知识。
每篇文章都建立在前一篇的基础上，建议按顺序阅读。

1. [平移](webgpu-translation.html)
2. [旋转](webgpu-rotation.html)
3. [缩放](webgpu-scale.html) ⬅ 你在这里
4. [矩阵数学](webgpu-matrix-math.html)
5. [正交投影](webgpu-orthographic-projection.html)
6. [透视投影](webgpu-perspective-projection.html)
7. [相机](webgpu-cameras.html)
8. [矩阵栈](webgpu-matrix-stacks.html)
9. [场景图](webgpu-scene-graphs.html)

缩放和[平移](webgpu-translation.html)一样简单。

我们将顶点位置乘以期望的缩放值。
以下是相比[上一个示例](webgpu-rotation.html)对着色器的修改：

```wgsl
struct Uniforms {
  color: vec4f,
  resolution: vec2f,
  translation: vec2f,
  rotation: vec2f,
  scale: vec2f,
};

struct Vertex {
  @location(0) position: vec2f,
};

struct VSOutput {
  @builtin(position) position: vec4f,
};

@group(0) @binding(0) var<uniform> uni: Uniforms;

@vertex fn vs(vert: Vertex) -> VSOutput {
  var vsOut: VSOutput;

+  // Scale the position
+  let scaledPosition = vert.position * uni.scale;

  // Rotate the position
  let rotatedPosition = vec2f(
-    vert.position.x * uni.rotation.y - vert.position.y * uni.rotation.x,
-    vert.position.x * uni.rotation.x + vert.position.y * uni.rotation.y
+    scaledPosition.x * uni.rotation.y - scaledPosition.y * uni.rotation.x,
+    scaledPosition.x * uni.rotation.x + scaledPosition.y * uni.rotation.y
  );

  // Add in the translation
  let position = rotatedPosition + uni.translation;

  // convert the position from pixels to a 0.0 to 1.0 value
  let zeroToOne = position / uni.resolution;

  // convert from 0 <-> 1 to 0 <-> 2
  let zeroToTwo = zeroToOne * 2.0;

  // covert from 0 <-> 2 to -1 <-> +1 (clip space)
  let flippedClipSpace = zeroToTwo - 1.0;

  // flip Y
  let clipSpace = flippedClipSpace * vec2f(1, -1);

  vsOut.position = vec4f(clipSpace, 0.0, 1.0);
  return vsOut;
}
```

和之前一样，我们需要更新 uniform buffer，为缩放值腾出空间。

```js
-  // color, resolution, translation, rotation, padding
-  const uniformBufferSize = (4 + 2 + 2 + 2) * 4 + 8;
+  // color, resolution, translation, rotation, scale
+  const uniformBufferSize = (4 + 2 + 2 + 2 + 2) * 4;
  const uniformBuffer = device.createBuffer({
    label: 'uniforms',
    size: uniformBufferSize,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
  });

  const uniformValues = new Float32Array(uniformBufferSize / 4);

  // offsets to the various uniform values in float32 indices
  const kColorOffset = 0;
  const kResolutionOffset = 4;
  const kTranslationOffset = 6;
  const kRotationOffset = 8;
+  const kScaleOffset = 10;

  const colorValue = uniformValues.subarray(kColorOffset, kColorOffset + 4);
  const resolutionValue = uniformValues.subarray(kResolutionOffset, kResolutionOffset + 2);
  const translationValue = uniformValues.subarray(kTranslationOffset, kTranslationOffset + 2);
  const rotationValue = uniformValues.subarray(kRotationOffset, kRotationOffset + 2);
+  const scaleValue = uniformValues.subarray(kScaleOffset, kScaleOffset + 2);
```

在渲染时，我们还需要更新缩放值：

```js
  const settings = {
    translation: [150, 100],
    rotation: degToRad(30),
+    scale: [1, 1],
  };

  const radToDegOptions = { min: -360, max: 360, step: 1, converters: GUI.converters.radToDeg };

  const gui = new GUI();
  gui.onChange(render);
  gui.add(settings.translation, '0', 0, 1000).name('translation.x');
  gui.add(settings.translation, '1', 0, 1000).name('translation.y');
  gui.add(settings, 'rotation', radToDegOptions);
+  gui.add(settings.scale, '0', -5, 5).name('scale.x');
+  gui.add(settings.scale, '1', -5, 5).name('scale.y');

  function render() {
    ...

    // Set the uniform values in our JavaScript side Float32Array
    resolutionValue.set([canvas.width, canvas.height]);
    translationValue.set(settings.translation);
    rotationValue.set([
        Math.cos(settings.rotation),
        Math.sin(settings.rotation),
    ]);
+    scaleValue.set(settings.scale);

    // upload the uniform values to the uniform buffer
    device.queue.writeBuffer(uniformBuffer, 0, uniformValues);
```

现在我们有了缩放功能。拖动滑块试试看。

{{{example url="../webgpu-scale.html" }}}

有一点需要注意：将缩放设为负值会让几何体翻转。

另一点需要注意的是，缩放是以 0, 0 为原点进行的，
对于我们的 F 来说，就是左上角。
这是合理的，因为我们是将顶点位置乘以缩放值，
所以它们会相对于 0, 0 移动。
你大概能想到一些解决方法，
比如在缩放之前先做一次平移（称为*预缩放平移*），
或者直接修改 F 的顶点数据。我们很快还会介绍另一种方式。

希望这三篇文章能帮助你理解
[平移](webgpu-translation.html)、[旋转](webgpu-rotation.html)和缩放。
下一篇我们将介绍[矩阵的神奇之处](webgpu-matrix-math.html)——
它将这三者结合成一种**简单得多**、也往往更实用的形式。
