Title: WebGPU 矩阵数学
Description: 矩阵数学让一切变得简单
TOC: 矩阵数学

本文是系列文章的第 4 篇，希望能帮助你学习 3D 数学知识。
每篇文章都建立在前一篇的基础上，建议按顺序阅读。

1. [平移](webgpu-translation.html)
2. [旋转](webgpu-rotation.html)
3. [缩放](webgpu-scale.html)
4. [矩阵数学](webgpu-matrix-math.html) ⬅ 你在这里
5. [正交投影](webgpu-orthographic-projection.html)
6. [透视投影](webgpu-perspective-projection.html)
7. [相机](webgpu-cameras.html)
8. [矩阵栈](webgpu-matrix-stacks.html)
9. [场景图](webgpu-scene-graphs.html)

在前三篇文章中，我们学习了如何对顶点位置进行[平移](webgpu-translation.html)、
[旋转](webgpu-rotation.html)和[缩放](webgpu-scale.html)。
平移、旋转和缩放各自被视为一种*变换（transformation）*。
每种变换都需要修改着色器，而且三种变换的应用顺序是有影响的。

在[上一个示例](webgpu-scale.html)中，我们先缩放、再旋转、最后平移。
如果改变顺序，结果就不一样了。

例如，先缩放 2, 1，然后旋转 30 度，再平移 100, 0：

<img src="resources/f-scale-rotation-translation.svg" class="webgpu_center" width="400" />

而这是先平移 100, 0，再旋转 30 度，再缩放 2, 1：

<img src="resources/f-translation-rotation-scale.svg" class="webgpu_center" width="400" />

结果完全不同。更糟糕的是，如果我们想要第二种效果，
就不得不写一个新的着色器来按新的顺序应用平移、旋转和缩放。

但有些聪明人想出了一种方法，
用矩阵数学来完成同样的事情。对于 2D，我们使用 3x3 矩阵。
3x3 矩阵就像一个有 9 个格子的网格：

<div class="glocal-center">
  <table class="glocal-center-content glocal-mat">
    <tr>
      <td class="m11">1</td>
      <td class="m12">4</td>
      <td class="m13">7</td>
    </tr>
    <tr>
      <td class="m21">2</td>
      <td class="m22">5</td>
      <td class="m23">8</td>
    </tr>
    <tr>
      <td class="m31">3</td>
      <td class="m32">6</td>
      <td class="m33">9</td>
    </tr>
  </table>
</div>

计算方式是将位置沿矩阵的行相乘，然后把结果加起来。

<div class="webgpu_center"><img src="resources/matrix-vector-math.svg" class="noinvertdark" style="width: 1000px;"></div>

我们的位置只有 x 和 y 两个值，
但为了做这个运算，我们需要 3 个值，所以第三个值用 1。

在这个例子中，结果将是：

<div class="glocal-center">
  <p>newX = x * <span class="m11">1</span> + y * <span class="m12">4</span> + 1 * <span class="m13">7</span></p>
  <p>newY = x * <span class="m21">2</span> + y * <span class="m22">5</span> + 1 * <span class="m23">8</span></p>
  <p>newZ = x * <span class="m31">3</span> + y * <span class="m32">6</span> + 1 * <span class="m33">9</span></p>
</div>

你可能会想"这有什么用？"好吧，
假设我们有一个平移操作，平移量为 tx 和 ty。
我们可以构造一个这样的矩阵：

<div class="glocal-center">
  <table class="glocal-center-content glocal-mat">
    <tr>
      <td class="m11">1</td>
      <td class="m12">0</td>
      <td class="m13">tx</td>
    </tr>
    <tr>
      <td class="m21">0</td>
      <td class="m22">1</td>
      <td class="m23">ty</td>
    </tr>
    <tr>
      <td class="m31">0</td>
      <td class="m32">0</td>
      <td class="m33">1</td>
    </tr>
  </table>
</div>

现在来看看：

<div class="glocal-center">
  <div class="eq">
    <div>newX = x * <span class="m11">1</span> + y * <span class="m12">0</span> + 1 * <span class="m13">tx</span></div>
    <div>newY = x * <span class="m21">0</span> + y * <span class="m22">1</span> + 1 * <span class="m23">ty</span></div>
    <div>newZ = x * <span class="m31">0</span> + y * <span class="m32">0</span> + 1 * <span class="m33">1</span></div>
  </div>
</div>

如果你还记得代数知识，任何乘以零的地方都可以去掉，
乘以 1 等于不变，所以让我们化简看看发生了什么：

<div class="glocal-center">
  <div class="eq">
    <div>newX = x <div class="blk">* <span class="m11">1</span></div> + <div class="blk">y * <span class="m12">0</span> + 1 * </div><span class="m13">tx</span></div>
    <div>newY = <div class="blk">x * <span class="m21">0</span> +</div> y <div class="blk">* <span class="m22">1</span></div> + <div class="blk">1 * </div><span class="m23">ty</span></div>
    <div>newZ = <div class="blk">x * <span class="m31">0</span> + y * <span class="m32">0</span> +</div> 1 <div class="blk">* <span class="m33">1</span></div></div>
  </div>
</div>

更简洁地说就是：

<div class="webgpu_center"><pre class="webgpu_math">
newX = x + tx;
newY = y + ty;
</pre></div>

newZ 我们并不关心。

这看起来跟我们[平移示例中的代码](webgpu-translation.html)非常像。

类似地，来做旋转。
正如我们在旋转那篇文章中指出的，我们只需要旋转角度的正弦和余弦值：

<div class="webgpu_center"><pre class="webgpu_math">
s = Math.sin(angleToRotateInRadians);
c = Math.cos(angleToRotateInRadians);
</pre></div>

然后构造这样的矩阵：

<div class="glocal-center">
  <table class="glocal-center-content glocal-mat">
    <tr>
      <td class="m11">c</td>
      <td class="m12">-s</td>
      <td class="m13">0</td>
    </tr>
    <tr>
      <td class="m21">s</td>
      <td class="m22">c</td>
      <td class="m23">0</td>
    </tr>
    <tr>
      <td class="m31">0</td>
      <td class="m32">0</td>
      <td class="m33">1</td>
    </tr>
  </table>
</div>

应用矩阵后得到：

<div class="glocal-center">
  <div class="eq">
    <div>newX = x * <span class="m11">c</span> + y * <span class="m12">-s</span> + 1 * <span class="m13">0</span></div>
    <div>newY = x * <span class="m21">s</span> + y * <span class="m22">c</span> + 1 * <span class="m23">0</span></div>
    <div>newZ = x * <span class="m31">0</span> + y * <span class="m32">0</span> + 1 * <span class="m33">1</span></div>
  </div>
</div>

将所有乘以 0 和 1 的部分标灰后得到：

<div class="glocal-center">
  <div class="eq">
    <div>newX = x * <span class="m11">c</span> + y * <span class="m12">-s</span><div class="blk"> + 1 * <span class="m13">0</span></div></div>
    <div>newY = x * <span class="m21">s</span> + y * <span class="m22">c</span><div class="blk"> + 1 * <span class="m23">0</span></div></div>
    <div>newZ = <div class="blk">x * <span class="m31">0</span> + y * <span class="m32">0</span> +</div> 1 <div class="blk">* <span class="m33">1</span></div></div>
  </div>
</div>

化简后得到：

<div class="webgpu_center">
<pre class="webgpu_math">
newX = x * c - y * s;
newY = x * s + y * c;
</pre>
</div>

这和我们在[旋转示例](webgpu-rotation.html)中的结果完全一样。

最后是缩放。两个缩放因子分别记为 sx 和 sy。

构造这样的矩阵：

<div class="glocal-center">
  <table class="glocal-center-content glocal-mat">
    <tr>
      <td class="m11">sx</td>
      <td class="m12">0</td>
      <td class="m13">0</td>
    </tr>
    <tr>
      <td class="m21">0</td>
      <td class="m22">sy</td>
      <td class="m23">0</td>
    </tr>
    <tr>
      <td class="m31">0</td>
      <td class="m32">0</td>
      <td class="m33">1</td>
    </tr>
  </table>
</div>

应用矩阵后得到：

<div class="glocal-center">
  <div class="eq">
    <div>newX = x * <span class="m11">sx</span> + y * <span class="m12">0</span> + 1 * <span class="m13">0</span></div>
    <div>newY = x * <span class="m21">0</span> + y * <span class="m22">sy</span> + 1 * <span class="m23">0</span></div>
    <div>newZ = x * <span class="m31">0</span> + y * <span class="m32">0</span> + 1 * <span class="m33">1</span></div>
  </div>
</div>

实际上就是：

<div class="glocal-center">
  <div class="eq">
    <div>newX = x * <span class="m11">sx</span><div class="blk"> + y * <span class="m12">0</span> + 1 * <span class="m13">0</span></div></div>
    <div>newY = <div class="blk">x * <span class="m21">0</span> +</div> y * <span class="m22">sy</span><div class="blk"> + 1 * <span class="m23">0</span></div></div>
    <div>newZ = <div class="blk">x * <span class="m31">0</span> + y * <span class="m32">0</span> +</div> 1 <div class="blk">* <span class="m33">1</span></div></div>
  </div>
</div>

化简后就是：

<div class="webgpu_center">
<pre class="webgpu_math">
newX = x * sx;
newY = y * sy;
</pre>
</div>

这和我们的[缩放示例](webgpu-scale.html)完全一样。

现在你可能还在想"那又怎样？有什么意义呢？"
看起来做了一堆额外的工作，结果和之前一样。

神奇的地方来了。我们可以把矩阵相乘，
一次性应用所有变换。假设我们有一个函数 `mat3.multiply`，
它接收两个矩阵、将它们相乘并返回结果。

```js
const mat3 = {
  multiply: function(a, b) {
    const a00 = a[0 * 3 + 0];
    const a01 = a[0 * 3 + 1];
    const a02 = a[0 * 3 + 2];
    const a10 = a[1 * 3 + 0];
    const a11 = a[1 * 3 + 1];
    const a12 = a[1 * 3 + 2];
    const a20 = a[2 * 3 + 0];
    const a21 = a[2 * 3 + 1];
    const a22 = a[2 * 3 + 2];
    const b00 = b[0 * 3 + 0];
    const b01 = b[0 * 3 + 1];
    const b02 = b[0 * 3 + 2];
    const b10 = b[1 * 3 + 0];
    const b11 = b[1 * 3 + 1];
    const b12 = b[1 * 3 + 2];
    const b20 = b[2 * 3 + 0];
    const b21 = b[2 * 3 + 1];
    const b22 = b[2 * 3 + 2];

    return [
      b00 * a00 + b01 * a10 + b02 * a20,
      b00 * a01 + b01 * a11 + b02 * a21,
      b00 * a02 + b01 * a12 + b02 * a22,
      b10 * a00 + b11 * a10 + b12 * a20,
      b10 * a01 + b11 * a11 + b12 * a21,
      b10 * a02 + b11 * a12 + b12 * a22,
      b20 * a00 + b21 * a10 + b22 * a20,
      b20 * a01 + b21 * a11 + b22 * a21,
      b20 * a02 + b21 * a12 + b22 * a22,
    ];
  }
}
```

为了让事情更清晰，让我们编写生成平移、旋转和缩放矩阵的函数。

```js
const mat3 = {
  multiply(a, b) {
    ...
  },
  translation([tx, ty]) {
    return [
      1, 0, 0,
      0, 1, 0,
      tx, ty, 1,
    ];
  },

  rotation(angleInRadians) {
    const c = Math.cos(angleInRadians);
    const s = Math.sin(angleInRadians);
    return [
      c, s, 0,
      -s, c, 0,
      0, 0, 1,
    ];
  },

  scaling([sx, sy]) {
    return [
      sx, 0, 0,
      0, sy, 0,
      0, 0, 1,
    ];
  },
};
```

现在让我们修改着色器来使用矩阵：

```wgsl
struct Uniforms {
  color: vec4f,
  resolution: vec2f,
-  translation: vec2f,
-  rotation: vec2f,
-  scale: vec2f,
+  matrix: mat3x3f,
};

...

@vertex fn vs(vert: Vertex) -> VSOutput {
  var vsOut: VSOutput;

-  // Scale the position
-  let scaledPosition = vert.position * uni.scale;
-
-  // Rotate the position
-  let rotatedPosition = vec2f(
-    scaledPosition.x * uni.rotation.x - scaledPosition.y * uni.rotation.y,
-    scaledPosition.x * uni.rotation.y + scaledPosition.y * uni.rotation.x
-  );
-
-  // Add in the translation
-  let position = rotatedPosition + uni.translation;
+  // Multiply by a matrix
+  let position = (uni.matrix * vec3f(vert.position, 1)).xy;

  ...
```

如你所见，我们将 z 传入 1，用矩阵乘以位置，
然后只保留结果中的 x 和 y。

同样需要更新 uniform buffer 的大小和偏移量：

```js
-  // color, resolution, translation, rotation, scale
-  const uniformBufferSize = (4 + 2 + 2 + 2 + 2) * 4;
+  // color, resolution, padding, matrix
+  const uniformBufferSize = (4 + 2 + 2 + 12) * 4;
  const uniformBuffer = device.createBuffer({
    label: 'uniforms',
    size: uniformBufferSize,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
  });

  const uniformValues = new Float32Array(uniformBufferSize / 4);

  // offsets to the various uniform values in float32 indices
  const kColorOffset = 0;
  const kResolutionOffset = 4;
  const kMatrixOffset = 8;

  const colorValue = uniformValues.subarray(kColorOffset, kColorOffset + 4);
  const resolutionValue = uniformValues.subarray(kResolutionOffset, kResolutionOffset + 2);
-  const translationValue = uniformValues.subarray(kTranslationOffset, kTranslationOffset + 2);
-  const rotationValue = uniformValues.subarray(kRotationOffset, kRotationOffset + 2);
-  const scaleValue = uniformValues.subarray(kScaleOffset, kScaleOffset + 2);
+  const matrixValue = uniformValues.subarray(kMatrixOffset, kMatrixOffset + 12);
```

最后，我们需要在渲染时做一些*矩阵数学*运算：

```js
  function render() {
    ...
+    const translationMatrix = mat3.translation(settings.translation);
+    const rotationMatrix = mat3.rotation(settings.rotation);
+    const scaleMatrix = mat3.scaling(settings.scale);
+
+    let matrix = mat3.multiply(translationMatrix, rotationMatrix);
+    matrix = mat3.multiply(matrix, scaleMatrix);

    // Set the uniform values in our JavaScript side Float32Array
    resolutionValue.set([canvas.width, canvas.height]);
-    translationValue.set(settings.translation);
-    rotationValue.set([
-        Math.cos(settings.rotation),
-        Math.sin(settings.rotation),
-    ]);
-    scaleValue.set(settings.scale);
+    matrixValue.set([
+      ...matrix.slice(0, 3), 0,
+      ...matrix.slice(3, 6), 0,
+      ...matrix.slice(6, 9), 0,
+    ]);
```

下面就是使用新代码的效果。
滑块还是一样的——平移、旋转和缩放——
但它们在着色器中的使用方式简单多了。

{{{example url="../webgpu-matrix-math-transform-trs-3x3.html"}}}

## <a id="a-columns-are-rows"></a> 列即是行

在介绍矩阵工作原理时，我们提到了按列相乘。
比如我们展示了这个平移矩阵作为例子：

<div class="glocal-center">
  <table class="glocal-center-content glocal-mat">
    <tr>
      <td class="m11">1</td>
      <td class="m12">0</td>
      <td class="m13">tx</td>
    </tr>
    <tr>
      <td class="m21">0</td>
      <td class="m22">1</td>
      <td class="m23">ty</td>
    </tr>
    <tr>
      <td class="m31">0</td>
      <td class="m32">0</td>
      <td class="m33">1</td>
    </tr>
  </table>
</div>

但我们在代码中实际构建矩阵时是这样做的：

```js
  translation([tx, ty]) {
    return [
      1, 0, 0,
      0, 1, 0,
      tx, ty, 1,
    ];
  },
```

`tx, ty, 1` 部分在底部的行中，而不是最后一列。

```js
  translation([tx, ty]) {
    return [
      1, 0, 0,   // <-- 1st column
      0, 1, 0,   // <-- 2nd column
      tx, ty, 1, // <-- 3rd column
    ];
  },
```

一些图形学专家解决这个问题的方式是把这些叫做"列"。
遗憾的是，这只是你需要习惯的事情。
数学书和网上的数学文章会像上面的图示那样展示矩阵，
其中 `tx, ty, 1` 在最后一列，
但在代码中（至少在 WebGPU 中），我们是按上面的方式来指定的。

## 矩阵数学的灵活性

你可能还在问，那又怎样呢？这看起来没什么好处嘛。
好处在于：现在如果我们想改变操作顺序，
不需要写新的着色器，只需在 JavaScript 中改变数学运算即可。

```js
-    let matrix = mat3.multiply(translationMatrix, rotationMatrix);
-    matrix = mat3.multiply(matrix, scaleMatrix);
+    let matrix = mat3.multiply(scaleMatrix, rotationMatrix);
+    matrix = mat3.multiply(matrix, translationMatrix);
```

上面我们把顺序从 平移→旋转→缩放 改成了 缩放→旋转→平移。

{{{example url="../webgpu-matrix-math-transform-srt-3x3.html"}}}

拖动滑块，你会看到现在表现不同了，
因为我们以不同的顺序组合矩阵。
例如，平移现在发生在旋转之后。

<div class="webgpu_center compare" style="justify-content: space-evenly;">
  <div style="flex: 0 0 auto;">
    <div>平移→旋转→缩放</div>
    <div><div data-diagram="trs"></div></div>
  </div>
  <div style="flex: 0 0 auto;">
    <div>缩放→旋转→平移</div>
    <div><div data-diagram="srt"></div></div>
  </div>
</div>

左边的可以描述为：一个经过缩放和旋转的 F，左右平移。
而右边的则可以更好地描述为：平移本身被旋转和缩放了。
移动方向不是左↔右，而是沿对角线的。
并且右边的 F 移动距离没那么远，因为平移本身也被缩放了。

这种灵活性就是为什么矩阵数学是几乎所有计算机图形学的核心组件。

能够这样应用矩阵对于层次动画尤为重要，
比如身体上的手臂和腿、围绕太阳转的行星上的月亮、
或树上的树枝等。
举一个简单的层次矩阵应用的例子，
让我们画 5 个 'F'，每次都从上一个 'F' 的矩阵开始。

为此我们需要 5 个 uniform buffer、5 组 uniform 值和 5 个 bindGroup：

```js
+  const numObjects = 5;
+  const objectInfos = [];
+  for (let i = 0; i < numObjects; ++i) {
    // color, resolution, padding, matrix
    const uniformBufferSize = (4 + 2 + 2 + 12) * 4;
    const uniformBuffer = device.createBuffer({
      label: 'uniforms',
      size: uniformBufferSize,
      usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
    });

    const uniformValues = new Float32Array(uniformBufferSize / 4);

    // offsets to the various uniform values in float32 indices
    const kColorOffset = 0;
    const kResolutionOffset = 4;
    const kMatrixOffset = 8;

    const colorValue = uniformValues.subarray(kColorOffset, kColorOffset + 4);
    const resolutionValue = uniformValues.subarray(kResolutionOffset, kResolutionOffset + 2);
    const matrixValue = uniformValues.subarray(kMatrixOffset, kMatrixOffset + 12);

    // The color will not change so let's set it once at init time
    colorValue.set([Math.random(), Math.random(), Math.random(), 1]);

    const bindGroup = device.createBindGroup({
      label: 'bind group for object',
      layout: pipeline.getBindGroupLayout(0),
      entries: [
        { binding: 0, resource: uniformBuffer },
      ],
    });

+    objectInfos.push({
+      uniformBuffer,
+      uniformValues,
+      resolutionValue,
+      matrixValue,
+      bindGroup,
+    });
+  }
```

在渲染时，我们遍历它们，
将前一个矩阵与我们的平移、旋转和缩放矩阵相乘：

```js
function render() {
  ...

  const translationMatrix = mat3.translation(settings.translation);
  const rotationMatrix = mat3.rotation(settings.rotation);
  const scaleMatrix = mat3.scaling(settings.scale);

-  let matrix = mat3.multiply(translationMatrix, rotationMatrix);
-  matrix = mat3.multiply(matrix, scaleMatrix);

+  // Starting Matrix.
+  let matrix = mat3.identity();
+
+  for (const {
+    uniformBuffer,
+    uniformValues,
+    resolutionValue,
+    matrixValue,
+    bindGroup,
+  } of objectInfos) {
+    matrix = mat3.multiply(matrix, translationMatrix)
+    matrix = mat3.multiply(matrix, rotationMatrix);
+    matrix = mat3.multiply(matrix, scaleMatrix);

    // Set the uniform values in our JavaScript side Float32Array
    resolutionValue.set([canvas.width, canvas.height]);
    matrixValue.set([
      ...matrix.slice(0, 3), 0,
      ...matrix.slice(3, 6), 0,
      ...matrix.slice(6, 9), 0,
    ]);

    // upload the uniform values to the uniform buffer
    device.queue.writeBuffer(uniformBuffer, 0, uniformValues);

    pass.setBindGroup(0, bindGroup);
    pass.drawIndexed(numVertices);
+  }

  pass.end();
```

为了实现这一点，我们引入了 `mat3.identity` 函数来创建单位矩阵。
单位矩阵实际上代表 1.0，
用单位矩阵相乘不改变任何东西。就像

<div class="webgpu_center"><div class="webgpu_math">X * 1 = X</div></div>

同样地

<div class="webgpu_center"><div class="webgpu_math">matrixX * identity = matrixX</div></div>

以下是创建单位矩阵的代码：

```js
const mat3 = {
  ...
  identity() {
    return [
      1, 0, 0,
      0, 1, 0,
      0, 0, 1,
    ];
  },

  ...
```

这是 5 个 F 的效果：

{{{example url="../webgpu-matrix-math-transform-five-fs-3x3.html"}}}

拖动滑块，看看每个后续的 'F' 如何相对于前一个 'F' 的大小和方向来绘制。
这就是 CG 人物手臂的工作原理——手臂的旋转影响前臂，
前臂的旋转影响手掌，手掌的旋转影响手指，以此类推……

## 改变旋转或缩放的中心点

再来看一个例子。在目前的所有示例中，
我们的 'F' 都围绕其左上角旋转
（上面反转顺序的示例除外）。
这是因为我们的数学运算总是围绕原点旋转，
而 'F' 的左上角正好在原点 (0, 0)。

但现在因为我们可以做矩阵运算，而且可以选择变换的应用顺序，
我们就可以移动原点。

```js
    const translationMatrix = mat3.translation(settings.translation);
    const rotationMatrix = mat3.rotation(settings.rotation);
    const scaleMatrix = mat3.scaling(settings.scale);
+    // make a matrix that will move the origin of the 'F' to its center.
+    const moveOriginMatrix = mat3.translation([-50, -75]);

    let matrix = mat3.multiply(translationMatrix, rotationMatrix);
    matrix = mat3.multiply(matrix, scaleMatrix);
+    matrix = mat3.multiply(matrix, moveOriginMatrix);
```

上面我们添加了一个平移，将 F 移动 -50, -75。
这会把它的所有点移动，使得 0, 0 位于 F 的中心。
拖动滑块，注意 F 现在围绕其中心旋转和缩放。

{{{example url="../webgpu-matrix-math-transform-move-origin-3x3.html" }}}

利用这种技术，你可以从任何点旋转或缩放。
现在你知道图像编辑软件是怎么让你移动旋转中心点的了。

## 加入投影

让我们再疯狂一点。你可能还记得着色器中有一段
将像素坐标转换为裁剪空间的代码：

```wgsl
// convert the position from pixels to a 0.0 to 1.0 value
let zeroToOne = position / uni.resolution;

// convert from 0 <-> 1 to 0 <-> 2
let zeroToTwo = zeroToOne * 2.0;

// covert from 0 <-> 2 to -1 <-> +1 (clip space)
let flippedClipSpace = zeroToTwo - 1.0;

// flip Y
let clipSpace = flippedClipSpace * vec2f(1, -1);

vsOut.position = vec4f(clipSpace, 0.0, 1.0);
```

逐步来看这些步骤：

第一步："将位置从像素转换为 0.0 到 1.0 的值"，
实际上是一个缩放操作。`zeroToOne = position / uni.resolution`
等同于 `zeroToOne = position * (1 / uni.resolution)`，即缩放。

第二步，`let zeroToTwo = zeroToOne * 2.0;` 也是缩放操作，缩放 2 倍。

第三步，`flippedClipSpace = zeroToTwo - 1.0;` 是平移。

第四步，`clipSpace = flippedClipSpace * vec2f(1, -1);` 是缩放。

所以我们可以把这些加到矩阵运算中：

```js
+  const scaleBy1OverResolutionMatrix = mat3.scaling([1 / canvas.width, 1 / canvas.height]);
+  const scaleBy2Matrix = mat3.scaling([2, 2]);
+  const translateByMinus1 = mat3.translation([-1, -1]);
+  const scaleBy1Minus1 = mat3.scaling([1, -1]);

  const translationMatrix = mat3.translation(settings.translation);
  const rotationMatrix = mat3.rotation(settings.rotation);
  const scaleMatrix = mat3.scaling(settings.scale);

-  let matrix = mat3.multiply(translationMatrix, rotationMatrix);
+  let matrix = mat3.multiply(scaleBy1Minus1, translateByMinus1);
+  matrix = mat3.multiply(matrix, scaleBy2Matrix);
+  matrix = mat3.multiply(matrix, scaleBy1OverResolutionMatrix);
+  matrix = mat3.multiply(matrix, translationMatrix);
+  matrix = mat3.multiply(matrix, rotationMatrix);
  matrix = mat3.multiply(matrix, scaleMatrix);
```

然后着色器可以改成这样：

```wgsl
struct Uniforms {
  color: vec4f,
-  resolution: vec2f,
  matrix: mat3x3f,
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

-  let position = (uni.matrix * vec3f(vert.position, 1)).xy;
-
-  // convert the position from pixels to a 0.0 to 1.0 value
-  let zeroToOne = position / uni.resolution;
-
-  // convert from 0 <-> 1 to 0 <-> 2
-  let zeroToTwo = zeroToOne * 2.0;
-
-  // covert from 0 <-> 2 to -1 <-> +1 (clip space)
-  let flippedClipSpace = zeroToTwo - 1.0;
-
-  // flip Y
-  let clipSpace = flippedClipSpace * vec2f(1, -1);
-
-  vsOut.position = vec4f(clipSpace, 0.0, 1.0);
+  let clipSpace = (uni.matrix * vec3f(vert.position, 1)).xy;
+
+  vsOut.position = vec4f(clipSpace, 0.0, 1.0);
  return vsOut;
}

@fragment fn fs(vsOut: VSOutput) -> @location(0) vec4f {
  return uni.color;
}
```

着色器现在变得超级简单，而且功能丝毫未减。
实际上它变得更灵活了！
我们不再硬编码为像素单位，
可以从着色器外部选择不同的单位系统。
这都是因为我们使用了矩阵数学。

与其创建那 4 个额外的矩阵，不如直接写一个生成相同结果的函数：

```js
const mat3 = {
  projection(width, height) {
    // Note: This matrix flips the Y axis so that 0 is at the top.
    return [
      2 / width, 0, 0,
      0, -2 / height, 0,
      -1, 1, 1,
    ];
  },

  ...
```

JavaScript 代码变成这样：

```js
-  const scaleBy1OverResolutionMatrix = mat3.scaling([1 / canvas.width, 1 / canvas.height]);
-  const scaleBy2Matrix = mat3.scaling([2, 2]);
-  const translateByMinus1 = mat3.translation([-1, -1]);
-  const scaleBy1Minus1 = mat3.scaling([1, -1]);
  const projectionMatrix = mat3.projection(canvas.clientWidth, canvas.clientHeight);
  const translationMatrix = mat3.translation(settings.translation);
  const rotationMatrix = mat3.rotation(settings.rotation);
  const scaleMatrix = mat3.scaling(settings.scale);

-  let matrix = mat3.multiply(scaleBy1Minus1, translateByMinus1);
-  matrix = mat3.multiply(matrix, scaleBy2Matrix);
-  matrix = mat3.multiply(matrix, scaleBy1OverResolutionMatrix);
-  matrix = mat3.multiply(matrix, translationMatrix);
  let matrix = mat3.multiply(projectionMatrix, translationMatrix);
  matrix = mat3.multiply(matrix, rotationMatrix);
  matrix = mat3.multiply(matrix, scaleMatrix);
  matrix = mat3.multiply(matrix, moveOriginMatrix);
```

我们还去掉了 uniform buffer 中为 resolution 预留空间和设置它的代码。

通过这最后一步，我们把一个有 6-7 个步骤的复杂着色器，
变成了只有 1 个步骤的非常简单的着色器，
而且更加灵活——全靠矩阵数学的魔力。

{{{example url="../webgpu-matrix-math-transform-just-matrix-3x3.html" }}}

## 边乘边算

在继续之前，让我们稍作简化。虽然生成各个矩阵然后分别相乘是常见的做法，
但直接边乘边算也很常见。我们可以编写这样的函数：

```js
const mat3 = {

  ...

  translate: function(m, translation) {
    return mat3.multiply(m, mat3.translation(translation));
  },

  rotate: function(m, angleInRadians) {
    return mat3.multiply(m, mat3.rotation(angleInRadians));
  },

  scale: function(m, scale) {
    return mat3.multiply(m, mat3.scaling(scale));
  },

  ...

};
```

这样就可以把上面 7 行矩阵代码简化成 4 行：

```js
const projectionMatrix = mat3.projection(canvas.clientWidth, canvas.clientHeight);
-const translationMatrix = mat3.translation(settings.translation);
-const rotationMatrix = mat3.rotation(settings.rotation);
-const scaleMatrix = mat3.scaling(settings.scale);
-
-let matrix = mat3.multiply(projectionMatrix, translationMatrix);
-matrix = mat3.multiply(matrix, rotationMatrix);
-matrix = mat3.multiply(matrix, scaleMatrix);
+let matrix = mat3.translate(projectionMatrix, settings.translation);
+matrix = mat3.rotate(matrix, settings.rotation);
+matrix = mat3.scale(matrix, settings.scale);
```

## mat3x3 是 3 个带填充的 vec3f

正如[内存布局那篇文章](webgpu-memory-layout.md)中指出的，
`vec3f` 通常占用 4 个浮点数的空间，而不是 3 个。

以下是 `mat3x3f` 在内存中的样子：

<div class="webgpu_center" data-diagram="mat3x3f"></div>

这就是为什么我们需要这段代码来复制矩阵到 uniform 值中：

```js
    matrixValue.set([
      ...matrix.slice(0, 3), 0,
      ...matrix.slice(3, 6), 0,
      ...matrix.slice(6, 9), 0,
    ]);
```

我们可以通过修改矩阵函数来处理填充来解决这个问题。

```js
const mat3 = {
  projection(width, height) {
    // Note: This matrix flips the Y axis so that 0 is at the top.
    return [
-      2 / width, 0, 0,
-      0, -2 / height, 0,
-      -1, 1, 1,
+      2 / width, 0, 0, 0,
+      0, -2 / height, 0, 0,
+      -1, 1, 1, 0,
    ];
  },
  identity() {
    return [
-      1, 0, 0,
-      0, 1, 0,
-      0, 0, 1,
+      1, 0, 0, 0,
+      0, 1, 0, 0,
+      0, 0, 1, 0,
    ];
  },
  multiply(a, b) {
-    const a00 = a[0 * 3 + 0];
-    const a01 = a[0 * 3 + 1];
-    const a02 = a[0 * 3 + 2];
-    const a10 = a[1 * 3 + 0];
-    const a11 = a[1 * 3 + 1];
-    const a12 = a[1 * 3 + 2];
-    const a20 = a[2 * 3 + 0];
-    const a21 = a[2 * 3 + 1];
-    const a22 = a[2 * 3 + 2];
-    const b00 = b[0 * 3 + 0];
-    const b01 = b[0 * 3 + 1];
-    const b02 = b[0 * 3 + 2];
-    const b10 = b[1 * 3 + 0];
-    const b11 = b[1 * 3 + 1];
-    const b12 = b[1 * 3 + 2];
-    const b20 = b[2 * 3 + 0];
-    const b21 = b[2 * 3 + 1];
-    const b22 = b[2 * 3 + 2];
+    const a00 = a[0 * 4 + 0];
+    const a01 = a[0 * 4 + 1];
+    const a02 = a[0 * 4 + 2];
+    const a10 = a[1 * 4 + 0];
+    const a11 = a[1 * 4 + 1];
+    const a12 = a[1 * 4 + 2];
+    const a20 = a[2 * 4 + 0];
+    const a21 = a[2 * 4 + 1];
+    const a22 = a[2 * 4 + 2];
+    const b00 = b[0 * 4 + 0];
+    const b01 = b[0 * 4 + 1];
+    const b02 = b[0 * 4 + 2];
+    const b10 = b[1 * 4 + 0];
+    const b11 = b[1 * 4 + 1];
+    const b12 = b[1 * 4 + 2];
+    const b20 = b[2 * 4 + 0];
+    const b21 = b[2 * 4 + 1];
+    const b22 = b[2 * 4 + 2];

    return [
      b00 * a00 + b01 * a10 + b02 * a20,
      b00 * a01 + b01 * a11 + b02 * a21,
      b00 * a02 + b01 * a12 + b02 * a22,
+      0,
      b10 * a00 + b11 * a10 + b12 * a20,
      b10 * a01 + b11 * a11 + b12 * a21,
      b10 * a02 + b11 * a12 + b12 * a22,
+      0,
      b20 * a00 + b21 * a10 + b22 * a20,
      b20 * a01 + b21 * a11 + b22 * a21,
      b20 * a02 + b21 * a12 + b22 * a22,
+      0,
    ];
  },
  translation([tx, ty]) {
    return [
-      1, 0, 0,
-      0, 1, 0,
-      tx, ty, 1,
+      1, 0, 0, 0,
+      0, 1, 0, 0, 
+      tx, ty, 1, 0,
    ];
  },

  rotation(angleInRadians) {
    const c = Math.cos(angleInRadians);
    const s = Math.sin(angleInRadians);
    return [
-      c, s, 0,
-      -s, c, 0,
-      0, 0, 1,
+      c, s, 0, 0,
+      -s, c, 0, 0,
+      0, 0, 1, 0,
    ];
  },

  scaling([sx, sy]) {
    return [
-      sx, 0, 0,
-      0, sy, 0,
-      0, 0, 1,
+      sx, 0, 0, 0, 
+      0, sy, 0, 0,
+      0, 0, 1, 0,
    ];
  },
};
```

现在设置矩阵的代码可以简化为：

```js
-    matrixValue.set([
-      ...matrix.slice(0, 3), 0,
-      ...matrix.slice(3, 6), 0,
-      ...matrix.slice(6, 9), 0,
-    ]);
+    matrixValue.set(matrix);
```

## 原地更新矩阵

我们还可以做的是允许向矩阵函数传入一个目标矩阵，
这样就可以原地更新矩阵，而不需要复制。
两种方式都有用，所以我们让函数在没有传入目标矩阵时创建新矩阵，
否则使用传入的那个。

以 3 个函数为例：

```
const mat3 = {
-  multiply(a, b) {
+  multiply(a, b, dst) {
+    dst = dst || new Float32Array(12);
    const a00 = a[0 * 4 + 0];
    const a01 = a[0 * 4 + 1];
    const a02 = a[0 * 4 + 2];
    const a10 = a[1 * 4 + 0];
    const a11 = a[1 * 4 + 1];
    const a12 = a[1 * 4 + 2];
    const a20 = a[2 * 4 + 0];
    const a21 = a[2 * 4 + 1];
    const a22 = a[2 * 4 + 2];
    const b00 = b[0 * 4 + 0];
    const b01 = b[0 * 4 + 1];
    const b02 = b[0 * 4 + 2];
    const b10 = b[1 * 4 + 0];
    const b11 = b[1 * 4 + 1];
    const b12 = b[1 * 4 + 2];
    const b20 = b[2 * 4 + 0];
    const b21 = b[2 * 4 + 1];
    const b22 = b[2 * 4 + 2];

-    return [
-      b00 * a00 + b01 * a10 + b02 * a20,
-      b00 * a01 + b01 * a11 + b02 * a21,
-      b00 * a02 + b01 * a12 + b02 * a22,
-      0,
-      b10 * a00 + b11 * a10 + b12 * a20,
-      b10 * a01 + b11 * a11 + b12 * a21,
-      b10 * a02 + b11 * a12 + b12 * a22,
-      0,
-      b20 * a00 + b21 * a10 + b22 * a20,
-      b20 * a01 + b21 * a11 + b22 * a21,
-      b20 * a02 + b21 * a12 + b22 * a22,
-      0,
-    ];
+    dst[ 0] = b00 * a00 + b01 * a10 + b02 * a20;
+    dst[ 1] = b00 * a01 + b01 * a11 + b02 * a21;
+    dst[ 2] = b00 * a02 + b01 * a12 + b02 * a22;
+
+    dst[ 4] = b10 * a00 + b11 * a10 + b12 * a20;
+    dst[ 5] = b10 * a01 + b11 * a11 + b12 * a21;
+    dst[ 6] = b10 * a02 + b11 * a12 + b12 * a22;
+
+    dst[ 7] = b20 * a00 + b21 * a10 + b22 * a20;
+    dst[ 8] = b20 * a01 + b21 * a11 + b22 * a21;
+    dst[ 9] = b20 * a02 + b21 * a12 + b22 * a22;
+    return dst;
  },
-  translation([tx, ty]) {
+  translation([tx, ty], dst) {
+    dst = dst || new Float32Array(12);
-    return [
-      1, 0, 0, 0,
-      0, 1, 0, 0,
-      tx, ty, 1, 0,
-    ];
+    dst[0] = 1;   dst[1] = 0;   dst[ 2] = 0;
+    dst[4] = 0;   dst[5] = 1;   dst[ 6] = 0;
+    dst[8] = tx;  dst[9] = ty;  dst[10] = 1;
+    return dst;
  },
-  translate(m, translation) {
-    return mat3.multiply(m, mat3.translation(m));
+  translate(m, translation, dst) {
+    return mat3.multiply(m, mat3.translation(m), dst);
  }

  ...
```

对其他函数做同样的处理，现在代码可以变成这样：

```js
-    const projectionMatrix = mat3.projection(canvas.clientWidth, canvas.clientHeight);
-    let matrix = mat3.translate(projectionMatrix, settings.translation);
-    matrix = mat3.rotate(matrix, settings.rotation);
-    matrix = mat3.scale(matrix, settings.scale);
-    matrixValue.set(matrix);
+    mat3.projection(canvas.clientWidth, canvas.clientHeight, matrixValue);
+    mat3.translate(matrixValue, settings.translation, matrixValue);
+    mat3.rotate(matrixValue, settings.rotation, matrixValue);
+    mat3.scale(matrixValue, settings.scale, matrixValue);
```

我们不再需要把矩阵复制到 `matrixValue` 中，
而是可以直接在它上面操作。

{{{example url="../webgpu-matrix-math-transform-trs.html"}}}

## 变换点 vs 变换空间

最后一件事。前面我们看到顺序很重要。第一个例子中：

    translation * rotation * scale

第二个例子中：

    scale * rotation * translation

结果完全不同。

有两种方式来理解矩阵。给定表达式：

    projectionMat * translationMat * rotationMat * scaleMat * position

第一种方式（很多人觉得自然）是从右往左读。

首先将位置乘以缩放矩阵得到缩放后的位置：

    scaledPosition = scaleMat * position

然后将缩放后的位置乘以旋转矩阵得到旋转缩放后的位置：

    rotatedScaledPosition = rotationMat * scaledPosition

然后乘以平移矩阵得到平移旋转缩放后的位置：

    translatedRotatedScaledPosition = translationMat * rotatedScaledPosition

最后乘以投影矩阵得到裁剪空间坐标：

    clipSpacePosition = projectionMatrix * translatedRotatedScaledPosition

第二种理解方式是从左往右读。
在这种情况下，每个矩阵改变了我们绘制到的纹理所代表的*空间*。
纹理一开始代表裁剪空间（每个方向 -1 到 +1）。
从左到右应用的每个矩阵都会改变 canvas 所代表的空间。

第 1 步：无矩阵（或单位矩阵）

> <div data-diagram="space-change-0" data-caption="裁剪空间"></div>
>
> 白色区域是纹理，蓝色在纹理外。我们在裁剪空间中。
> 传入的位置需要在裁剪空间中。右上角的绿色区域
> 是 F 的左上角。它是上下颠倒的，因为在裁剪空间中 +Y 向上，
> 但 F 是在像素空间（+Y 向下）中设计的。
> 此外裁剪空间只显示 2x2 个单位，但 F 有 100x150 个单位大，所以我们只看到一个单位的量。

第 2 步：`mat3.projection(canvas.clientWidth, canvas.clientHeight, matrixValue);`

> <div data-diagram="space-change-1" data-caption="从裁剪空间到像素空间"></div>
>
> 现在我们在像素空间中。X = 0 到 textureWidth，Y = 0 到 textureHeight，0,0 在左上角。
> 使用此矩阵传入的位置需要在像素空间中。
> 你看到的闪烁是空间从正 Y = 上翻转到正 Y = 下的瞬间。

第 3 步：`mat3.translate(matrixValue, settings.translation, matrixValue);`

> <div data-diagram="space-change-2" data-caption="将原点移动到 tx, ty"></div>
>
> 空间的原点已被移动到 tx, ty (150, 100)。

第 4 步：`mat3.rotate(matrixValue, settings.rotation, matrixValue);`

> <div data-diagram="space-change-3" data-caption="旋转 33 度"></div>
>
> 空间围绕 tx, ty 旋转了。

第 5 步：`mat3.scale(matrixValue, settings.scale, matrixValue);`

> <div data-diagram="space-change-4" data-caption="缩放空间"></div>
>
> 之前围绕 tx, ty 旋转过的空间，在 x 方向缩放了 2 倍，y 方向缩放了 1.5 倍。

在着色器中我们执行 `clipSpace = uni.matrix * vert.position;`。
`vert.position` 的值实际上是在这个最终空间中被应用的。

选择你觉得更容易理解的方式就好。

希望这些文章帮你揭开了矩阵数学的神秘面纱。
下一篇[我们将进入 3D](webgpu-orthographic-projection.html)。
在 3D 中，矩阵数学遵循同样的原理和用法。
我们从 2D 开始是为了让内容更容易理解。

另外，如果你想真正成为矩阵数学的专家，
可以看看[这个很棒的视频](https://www.youtube.com/watch?v=kjBOesZCoqc&list=PLZHQObOWTQDPD3MizzM2xVFitgF8hE_ab)。

<div class="webgpu_bottombar">
<h3><code>clientWidth</code> 和 <code>clientHeight</code> 是什么？</h3>
<p>到目前为止，每当我们引用 canvas 尺寸时都用的是 <code>canvas.width</code> 和 <code>canvas.height</code>，
但上面调用 <code>mat3.projection</code> 时我们用的却是 <code>canvas.clientWidth</code> 和 <code>canvas.clientHeight</code>。为什么？</p>
<p>投影矩阵关注的是如何把裁剪空间（每个维度 -1 到 +1）转换回像素。
但在浏览器中，我们要处理两种像素。一种是 canvas 本身的像素数量。
例如一个这样定义的 canvas：</p>
<pre class="prettyprint">
  &lt;canvas width="400" height="300"&gt;&lt;/canvas&gt;
</pre>
<p>或者这样定义的：</p>
<pre class="prettyprint">
  const canvas = document.createElement("canvas");
  canvas.width = 400;
  canvas.height = 300;
</pre>
<p>这两种都包含一个 400 像素宽 300 像素高的图像。
但这个尺寸和浏览器实际显示这个 400x300 canvas 的大小是分开的。
CSS 定义了 canvas 的显示大小。例如：</p>
<pre class="prettyprint">
  &lt;style&gt;
    canvas {
      width: 100%;
      height: 100%;
    }
  &lt;/style&gt;
  ...
  &lt;canvas width="400" height="300">&lt;/canvas&gt;
</pre>
<p>这个 canvas 会按其容器的大小来显示，很可能不是 400x300。</p>
<p>下面有两个例子，都将 canvas 的 CSS 显示大小设为 100%，使 canvas 拉伸到充满页面。
第一个在调用 <code>mat3.projection</code> 时使用 <code>canvas.width</code> 和 <code>canvas.height</code>。
在新窗口中打开并调整窗口大小，注意 'F' 的宽高比不正确、会变形，
位置也不对。代码说左上角应该在 150, 25，但随着 canvas 拉伸和收缩，
我们希望出现在 150, 25 的位置会发生移动。</p>
{{{example url="../webgpu-canvas-width-height.html" width="500" height="150" }}}
<p>第二个例子在调用 <code>mat3.projection</code> 时使用 <code>canvas.clientWidth</code> 和 <code>canvas.clientHeight</code>。
<code>canvas.clientWidth</code> 和 <code>canvas.clientHeight</code> 报告的是浏览器实际显示 canvas 的大小。
在这种情况下，即使 canvas 内部仍然只有 400x300 像素，
由于我们根据 canvas 的显示大小来定义宽高比，
<code>F</code> 看起来始终是正确的，位置也是对的。</p>
{{{example url="../webgpu-canvas-clientwidth-clientheight.html" width="500" height="150" }}}
<p>大多数允许调整 canvas 大小的应用都会尝试使 <code>canvas.width</code> 和 <code>canvas.height</code>
与 <code>canvas.clientWidth</code> 和 <code>canvas.clientHeight</code> 匹配，
因为它们希望 canvas 中的每个像素对应浏览器显示的每个像素。
但正如我们上面看到的，这不是唯一的选择。
这意味着在几乎所有情况下，使用 <code>canvas.clientHeight</code>
和 <code>canvas.clientWidth</code> 来计算投影矩阵的宽高比在技术上更加正确。
</p>
</div>

<!-- keep this at the bottom of the article -->
<link href="webgpu-matrix-math.css" rel="stylesheet">
<script type="module" src="webgpu-matrix-math.js"></script>
