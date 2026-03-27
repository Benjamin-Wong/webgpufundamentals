Title: WebGPU 旋转
Description: 旋转一个对象
TOC: 旋转

本文是系列文章的第 2 篇，希望能帮助你学习 3D 数学知识。
每篇文章都建立在前一篇的基础上，建议按顺序阅读。

1. [平移](webgpu-translation.html)
2. [旋转](webgpu-rotation.html) ⬅ 你在这里
3. [缩放](webgpu-scale.html)
4. [矩阵数学](webgpu-matrix-math.html)
5. [正交投影](webgpu-orthographic-projection.html)
6. [透视投影](webgpu-perspective-projection.html)
7. [相机](webgpu-cameras.html)
8. [矩阵栈](webgpu-matrix-stacks.html)
9. [场景图](webgpu-scene-graphs.html)

我先坦白，我不确定用这种方式解释是否能讲清楚，不过没关系，试试看吧。

首先我想介绍一样叫"单位圆"的东西。
如果你还记得初中数学（别睡着！），
圆有一个半径，圆的半径是从圆心到圆边缘的距离。
单位圆就是半径为 1.0 的圆。

下面是一个单位圆。[^ydown]

[^ydown]: 这个单位圆的 +Y 向下，与我们的像素空间一致（像素空间也是 Y 向下的）。WebGPU 的标准裁剪空间是 +Y 向上。如我们在前一篇文章中介绍的，我们在着色器中翻转了 Y。

<div class="webgpu_center"><div data-diagram="unit-circle" style="display: inline-block; width: 500px;"></div></div>

注意，当你拖动圆上的蓝色手柄时，X 和 Y 的值会随之改变。
这两个值代表圆上该点的位置。
在顶部，Y 为 1，X 为 0；在右侧，X 为 1，Y 为 0。

如果你还记得小学数学：一个数乘以 1 结果不变，123 * 1 = 123。很基础，对吧？
单位圆，也就是半径为 1.0 的圆，也是一种形式的 1。它是一个"旋转的 1"。
所以你可以将某个东西乘以这个单位圆，
从某种意义上说就像乘以 1，只不过神奇之处在于：东西会旋转。

我们将从单位圆上某个点取 X 和 Y 值，
然后与[上一个示例](webgpu-translation.html)中的顶点位置相乘。

以下是对着色器的修改：

```wgsl
struct Uniforms {
  color: vec4f,
  resolution: vec2f,
  translation: vec2f,
+  rotation: vec2f,
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

+  // Rotate the position
+  let rotatedPosition = vec2f(
+    vert.position.x * uni.rotation.x - vert.position.y * uni.rotation.y,
+    vert.position.x * uni.rotation.y + vert.position.y * uni.rotation.x
+  );

  // Add in the translation
-  let position = vert.position + uni.translation;
+  let position = rotatedPosition + uni.translation;

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

然后更新 JavaScript，为新的 uniform 值分配空间。

```js
-  // color, resolution, translation
-  const uniformBufferSize = (4 + 2 + 2) * 4;
+  // color, resolution, translation, rotation, padding
+  const uniformBufferSize = (4 + 2 + 2 + 2) * 4 + 8;
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
+  const kRotationOffset = 8;

  const colorValue = uniformValues.subarray(kColorOffset, kColorOffset + 4);
  const resolutionValue = uniformValues.subarray(kResolutionOffset, kResolutionOffset + 2);
  const translationValue = uniformValues.subarray(kTranslationOffset, kTranslationOffset + 2);
+  const rotationValue = uniformValues.subarray(kRotationOffset, kRotationOffset + 2);
```

我们还需要一个 UI 界面。这不是一篇关于如何制作 UI 的教程，
所以我直接用现成的。先写一段 HTML 给它留一个位置：

```html
  <body>
    <canvas></canvas>
+    <div id="circle"></div>
  </body>
```

再写一些 CSS 来定位它：

```css
#circle {
  position: fixed;
  right: 0;
  bottom: 0;
  width: 300px;
  background-color: var(--bg-color);
}
```

最后是使用它的 JavaScript：

```js
+import UnitCircle from './resources/js/unit-circle.js';

...

  const gui = new GUI();
  gui.onChange(render);
  gui.add(settings.translation, '0', 0, 1000).name('translation.x');
  gui.add(settings.translation, '1', 0, 1000).name('translation.y');

+  const unitCircle = new UnitCircle();
+  document.querySelector('#circle').appendChild(unitCircle.domElement);
+  unitCircle.onChange(render);

  function render() {
    ...

    // Set the uniform values in our JavaScript side Float32Array
    resolutionValue.set([canvas.width, canvas.height]);
    translationValue.set(settings.translation);
+    rotationValue.set([unitCircle.x, unitCircle.y]);

    // upload the uniform values to the uniform buffer
    device.queue.writeBuffer(uniformBuffer, 0, uniformValues);
```

结果如下。拖动圆上的手柄来旋转，拖动滑块来平移。

{{{example url="../webgpu-rotation-via-unit-circle.html"}}}

为什么这样能工作呢？来看看这个数学公式：

<div class="webgpu_center">
<pre class="webgpu_math">
rotatedX = a_position.x * u_rotation.x - a_position.y * u_rotation.y;
rotatedY = a_position.x * u_rotation.y + a_position.y * u_rotation.x;
</pre>
</div>

假设你有一个矩形并想旋转它。
在旋转之前，右上角在 3.0、-9.0 的位置。
在单位圆上取一个从 3 点钟方向顺时针旋转 30 度的点：

<div class="webgpu_center"><div data-diagram="static-circle-30" style="display: inline-block; width: 400px;"></div></div>

该点在圆上的位置是 x = 0.87，y = 0.50：

<div class="webgpu_center">
<pre class="webgpu_math">
 3.0 * 0.87 - -9.0 * 0.50 =  7.1
 3.0 * 0.50 + -9.0 * 0.87 = -6.3
</pre>
</div>

这正好是我们需要的位置：

<img src="resources/rotation-drawing.svg" width="500" class="webgpu_center" style="width: 1000px"/>

顺时针旋转 60 度也同理：

<div class="webgpu_center"><div data-diagram="static-circle-60" style="display: inline-block; width: 400px;"></div></div>

该点在圆上的位置是 0.87 和 0.50：

<div class="webgpu_center">
<pre class="webgpu_math">
 3.0 * 0.50 - -9.0 * 0.87 =  9.3
 3.0 * 0.87 + -9.0 * 0.50 = -1.9
</pre>
</div>

你可以看到，随着该点顺时针旋转，X 值越来越大，Y 值越来越小。
如果继续旋转超过 90 度，X 会开始变小，Y 会开始变大。
这种规律就产生了旋转效果。

单位圆上的点还有另一个名字——它们叫做正弦和余弦。
对于任意给定的角度，我们可以像下面这样查找其正弦和余弦：

    function printSineAndCosineForAnAngle(angleInDegrees) {
      const angleInRadians = angleInDegrees * Math.PI / 180;
      const s = Math.sin(angleInRadians);
      const c = Math.cos(angleInRadians);
      console.log('s =', s, 'c =', c);
    }

如果你将上面的代码粘贴到 JavaScript 控制台并输入
`printSineAndCosignForAngle(30)`，你会看到它输出
`s = 0.50 c = 0.87`（注意：我对数字做了四舍五入）

把这些结合起来，你就可以将顶点位置旋转到任意角度。
只需将 rotation 设置为目标角度的正弦和余弦值：

      ...
      const angleInRadians = angleInDegrees * Math.PI / 180;
      rotation[0] = Math.cos(angleInRadians);
      rotation[1] = Math.sin(angleInRadians);

让我们把 UI 改成直接输入旋转角度：

```js
+  const degToRad = d => d * Math.PI / 180;

  const settings = {
    translation: [150, 100],
+    rotation: degToRad(30),
  };

  const radToDegOptions = { min: -360, max: 360, step: 1, converters: GUI.converters.radToDeg };

  const gui = new GUI();
  gui.onChange(render);
  gui.add(settings.translation, '0', 0, 1000).name('translation.x');
  gui.add(settings.translation, '1', 0, 1000).name('translation.y');
+  gui.add(settings, 'rotation', radToDegOptions);

-  const unitCircle = new UnitCircle();
-  document.querySelector('#circle').appendChild(unitCircle.domElement);
-  unitCircle.onChange(render);

  function render() {
    ...

    // Set the uniform values in our JavaScript side Float32Array
    resolutionValue.set([canvas.width, canvas.height]);
    translationValue.set(settings.translation);
-    rotationValue.set([unitCircle.x, unitCircle.y]);
+    rotationValue.set([
+        Math.cos(settings.rotation),
+        Math.sin(settings.rotation),
+    ]);
```

拖动滑块来平移或旋转。

{{{example url="../webgpu-rotation.html"}}}

希望这些内容对你有所帮助。[下一篇更简单：缩放](webgpu-scale.html)。

<div class="webgpu_bottombar"><h3>什么是弧度？</h3>
<p>
弧度是用于圆、旋转和角度的度量单位。
就像我们可以用英寸、码、米等来测量距离一样，
我们可以用度或弧度来测量角度。
</p>
<p>
你大概知道，公制计算比英制计算容易。
从英寸换算到英尺要除以 12，从英寸换算到码要除以 36。
我反正不能心算除以 36。而公制就简单多了，
从毫米换算到厘米除以 10，从毫米换算到米除以 1000。
我**能**心算除以 1000。
</p>
<p>
弧度和度数的情况类似。度数会让计算变复杂，弧度则让计算变简单。
一个圆有 360 度，但只有 2π 弧度。
所以转一圈是 2π 弧度，转半圈是 1π 弧度，
转 1/4 圈（也就是 90 度）是 1/2π 弧度。
因此，如果你想旋转 90 度，只需使用
<code>Math.PI * 0.5</code>；旋转 45 度就用
<code>Math.PI * 0.25</code>，以此类推。
</p>
<p>
几乎所有涉及角度、圆或旋转的数学，
一旦你开始用弧度思考就会变得非常简单。
试试看吧，除了 UI 显示之外，请使用弧度而非度数。
</p>
</div>

<!-- keep this at the bottom of the article -->
<script type="module" src="webgpu-rotation.js"></script>

