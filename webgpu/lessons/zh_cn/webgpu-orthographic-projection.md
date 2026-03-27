Title: WebGPU 正交投影
Description: 正交投影（无透视）
TOC: 正交投影

这篇文章是 3D 数学系列文章中的第 5 篇。
希望这一系列能帮助你理解 3D 数学。
每一篇都会建立在上一篇的基础上，
所以按顺序阅读通常会更容易理解。

1. [平移](webgpu-translation.html)
2. [旋转](webgpu-rotation.html)
3. [缩放](webgpu-scale.html)
4. [矩阵数学](webgpu-matrix-math.html)
5. [正交投影](webgpu-orthographic-projection.html) 你现在在这里
6. [透视投影](webgpu-perspective-projection.html)
7. [相机](webgpu-cameras.html)
8. [矩阵栈](webgpu-matrix-stacks.html)
9. [场景图](webgpu-scene-graphs.html)

上一篇里我们讲了矩阵是如何工作的。
我们提到，
平移、旋转、缩放，
甚至从像素坐标投影到 clip space，
都可以通过 1 个矩阵
和一些“神奇”的矩阵数学来完成。
从那里迈向 3D，
其实只是很小的一步。

在之前的 2D 示例里，
我们处理的是 2D 点 `(x, y)`，
并将它们乘以一个 3x3 矩阵。
要做 3D，
我们需要 3D 点 `(x, y, z)`
以及一个 4x4 矩阵。

让我们基于上一个示例，
把它改成 3D。
我们仍然使用字母 F，
但这次是一个 3D 的 “F”。

第一件事就是修改 vertex shader，
让它支持 3D。
下面是之前的 vertex shader。

```wgsl
struct Uniforms {
  color: vec4f,
-  matrix: mat3x3f,
+  matrix: mat4x4f,
};

struct Vertex {
-  @location(0) position: vec2f,
+  @location(0) position: vec4f,
};

struct VSOutput {
  @builtin(position) position: vec4f,
};

@group(0) @binding(0) var<uniform> uni: Uniforms;

@vertex fn vs(vert: Vertex) -> VSOutput {
  var vsOut: VSOutput;
-
-  let clipSpace = (uni.matrix * vec3f(vert.position, 1)).xy;
-  vsOut.position = vec4f(clipSpace, 0.0, 1.0);
  vsOut.position = uni.matrix * vert.position;
  return vsOut;
}

@fragment fn fs(vsOut: VSOutput) -> @location(0) vec4f {
  return uni.color;
}
```

它甚至变得更简单了！
就像在 2D 里，
我们提供 `x` 和 `y`，
再把 `z` 设成 1；
在 3D 里，
我们提供 `x`、`y` 和 `z`，
而 `w` 需要是 1，
不过这里我们可以利用一个事实：
对于 attribute 来说，
`w` 的默认值就是 1。

接着我们需要提供 3D 数据。

```js
function createFVertices() {
  const vertexData = new Float32Array([
    // left column
*    0, 0, 0,
*    30, 0, 0,
*    0, 150, 0,
*    30, 150, 0,

    // top rung
*    30, 0, 0,
*    100, 0, 0,
*    30, 30, 0,
*    100, 30, 0,

    // middle rung
*    30, 60, 0,
*    70, 60, 0,
*    30, 90, 0,
*    70, 90, 0,
  ]);

  const indexData = new Uint32Array([
    0,  1,  2,    2,  1,  3,  // left column
    4,  5,  6,    6,  5,  7,  // top run
    8,  9, 10,   10,  9, 11,  // middle run
  ]);

  return {
    vertexData,
    indexData,
    numVertices: indexData.length,
  };
}
```

上面其实只是给每一行末尾都加了一个 `0,`

```js
  const pipeline = device.createRenderPipeline({
    label: '2 attributes',
    layout: 'auto',
    vertex: {
      module,
      buffers: [
        {
-          arrayStride: (2) * 4, // (2) floats, 4 bytes each
+          arrayStride: (3) * 4, // (3) floats, 4 bytes each
          attributes: [
-            {shaderLocation: 0, offset: 0, format: 'float32x2'},  // position
+            {shaderLocation: 0, offset: 0, format: 'float32x3'},  // position
          ],
        },
      ],
    },
    fragment: {
      module,
      targets: [{ format: presentationFormat }],
    },
  });
```

接下来，
我们需要把所有矩阵数学从 2D
升级为 3D。

<div class="webgpu_center compare" style="align-items: end;">
  <div>
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
    <div>2D 平移矩阵</div>
  </div>
  <div>
    <div class="glocal-center">
      <table class="glocal-center-content glocal-mat">
        <tr>
          <td class="m11">1</td>
          <td class="m12">0</td>
          <td class="m13">0</td>
          <td class="m14">tx</td>
        </tr>
        <tr>
          <td class="m21">0</td>
          <td class="m22">1</td>
          <td class="m23">0</td>
          <td class="m24">ty</td>
        </tr>
        <tr>
          <td class="m31">0</td>
          <td class="m32">0</td>
          <td class="m33">1</td>
          <td class="m34">tz</td>
        </tr>
        <tr>
          <td class="m41">0</td>
          <td class="m42">0</td>
          <td class="m43">0</td>
          <td class="m44">1</td>
        </tr>
      </table>
    </div>
    <div>3D 平移矩阵</div>
  </div>
</div>

<div class="webgpu_center compare" style="align-items: end;">
  <div>
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
    <div>2D 旋转矩阵</div>
  </div>
  <div>
    <div class="glocal-center">
      <table class="glocal-center-content glocal-mat">
        <tr>
          <td class="m11">c</td>
          <td class="m12">-s</td>
          <td class="m13">0</td>
          <td class="m14">0</td>
        </tr>
        <tr>
          <td class="m21">s</td>
          <td class="m22">c</td>
          <td class="m23">0</td>
          <td class="m24">0</td>
        </tr>
        <tr>
          <td class="m31">0</td>
          <td class="m32">0</td>
          <td class="m33">1</td>
          <td class="m34">0</td>
        </tr>
        <tr>
          <td class="m41">0</td>
          <td class="m42">0</td>
          <td class="m43">0</td>
          <td class="m44">1</td>
        </tr>
      </table>
    </div>
    <div>3D Z 轴旋转矩阵</div>
  </div>
</div>

<div class="webgpu_center compare" style="align-items: end;">
  <div>
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
    <div>2D 缩放矩阵</div>
  </div>
  <div>
    <div class="glocal-center">
      <table class="glocal-center-content glocal-mat">
        <tr>
          <td class="m11">sx</td>
          <td class="m12">0</td>
          <td class="m13">0</td>
          <td class="m14">0</td>
        </tr>
        <tr>
          <td class="m21">0</td>
          <td class="m22">sy</td>
          <td class="m23">0</td>
          <td class="m24">0</td>
        </tr>
        <tr>
          <td class="m31">0</td>
          <td class="m32">0</td>
          <td class="m33">sz</td>
          <td class="m34">0</td>
        </tr>
        <tr>
          <td class="m41">0</td>
          <td class="m42">0</td>
          <td class="m43">0</td>
          <td class="m44">1</td>
        </tr>
      </table>
    </div>
    <div>3D 缩放矩阵</div>
  </div>
</div>

我们还可以构造 X 轴和 Y 轴的旋转矩阵

<div class="webgpu_center compare" style="align-items: end;">
  <div>
    <div class="glocal-center">
      <table class="glocal-center-content glocal-mat">
        <tr>
          <td class="m11">1</td>
          <td class="m12">0</td>
          <td class="m13">0</td>
          <td class="m14">0</td>
        </tr>
        <tr>
          <td class="m21">0</td>
          <td class="m22">c</td>
          <td class="m23">-s</td>
          <td class="m24">0</td>
        </tr>
        <tr>
          <td class="m31">0</td>
          <td class="m32">s</td>
          <td class="m33">c</td>
          <td class="m34">0</td>
        </tr>
        <tr>
          <td class="m41">0</td>
          <td class="m42">0</td>
          <td class="m43">0</td>
          <td class="m44">1</td>
        </tr>
      </table>
    </div>
    <div>3D X 轴旋转矩阵</div>
  </div>
  <div>
    <div class="glocal-center">
      <table class="glocal-center-content glocal-mat">
        <tr>
          <td class="m11">c</td>
          <td class="m12">0</td>
          <td class="m13">s</td>
          <td class="m14">0</td>
        </tr>
        <tr>
          <td class="m21">0</td>
          <td class="m22">1</td>
          <td class="m23">0</td>
          <td class="m24">0</td>
        </tr>
        <tr>
          <td class="m31">-s</td>
          <td class="m32">0</td>
          <td class="m33">c</td>
          <td class="m34">0</td>
        </tr>
        <tr>
          <td class="m41">0</td>
          <td class="m42">0</td>
          <td class="m43">0</td>
          <td class="m44">1</td>
        </tr>
      </table>
    </div>
    <div>3D Y 轴旋转矩阵</div>
  </div>
</div>

现在我们已经有了 3 个旋转矩阵。
在 2D 中，
我们只需要一个旋转矩阵，
因为那实际上等价于围绕 Z 轴旋转。
但在 3D 中，
我们还希望能够围绕 X 轴和 Y 轴旋转。
你从这些矩阵的形式上也能看出来，
它们彼此都非常相似。
如果像之前那样把它们展开，
你会发现它们也会被化简成类似的形式。

Z 轴旋转

<div class="webgpu_center"><pre class="webgpu_math">
newX = x * c + y * -s;
newY = x * s + y *  c;
</pre></div>

Y 轴旋转

<div class="webgpu_center"><pre class="webgpu_math">
newX = x *  c + z * s;
newZ = x * -s + z * c;
</pre></div>

X 轴旋转

<div class="webgpu_center"><pre class="webgpu_math">
newY = y * c + z * -s;
newZ = y * s + z *  c;
</pre></div>

它们对应的旋转效果如下。

<iframe class="external_diagram" src="resources/axis-diagram.html" style="width: 540px; height: 280px;"></iframe>

下面是 2D 版本（也就是之前的）
`mat3.translation`、
`mat3.rotation`
和 `mat3.scaling`

```js
const mat3 = {
  ...
  translation([tx, ty], dst) {
    dst = dst || new Float32Array(12);
    dst[0] = 1;   dst[1] = 0;   dst[2] = 0;
    dst[4] = 0;   dst[5] = 1;   dst[6] = 0;
    dst[8] = tx;  dst[9] = ty;  dst[10] = 1;
    return dst;
  },

  rotation(angleInRadians, dst) {
    const c = Math.cos(angleInRadians);
    const s = Math.sin(angleInRadians);
    dst = dst || new Float32Array(12);
    dst[0] = c;   dst[1] = s;  dst[2] = 0;
    dst[4] = -s;  dst[5] = c;  dst[6] = 0;
    dst[8] = 0;   dst[9] = 0;  dst[10] = 1;
    return dst;

  },

  scaling([sx, sy], dst) {
    dst = dst || new Float32Array(12);
    dst[0] = sx;  dst[1] = 0;   dst[2] = 0;
    dst[4] = 0;   dst[5] = sy;  dst[6] = 0;
    dst[8] = 0;   dst[9] = 0;   dst[10] = 1;
    return dst;
  },
  ...
```

下面则是更新后的 3D 版本

```js
const mat4 = {
  ...
  translation([tx, ty, tz], dst) {
    dst = dst || new Float32Array(16);
    dst[ 0] = 1;   dst[ 1] = 0;   dst[ 2] = 0;   dst[ 3] = 0;
    dst[ 4] = 0;   dst[ 5] = 1;   dst[ 6] = 0;   dst[ 7] = 0;
    dst[ 8] = 0;   dst[ 9] = 0;   dst[10] = 1;   dst[11] = 0;
    dst[12] = tx;  dst[13] = ty;  dst[14] = tz;  dst[15] = 1;
    return dst;
  },

  rotationX(angleInRadians, dst) {
    const c = Math.cos(angleInRadians);
    const s = Math.sin(angleInRadians);
    dst = dst || new Float32Array(16);
    dst[ 0] = 1;  dst[ 1] = 0;   dst[ 2] = 0;  dst[ 3] = 0;
    dst[ 4] = 0;  dst[ 5] = c;   dst[ 6] = s;  dst[ 7] = 0;
    dst[ 8] = 0;  dst[ 9] = -s;  dst[10] = c;  dst[11] = 0;
    dst[12] = 0;  dst[13] = 0;   dst[14] = 0;  dst[15] = 1;
    return dst;
  },

  rotationY(angleInRadians, dst) {
    const c = Math.cos(angleInRadians);
    const s = Math.sin(angleInRadians);
    dst = dst || new Float32Array(16);
    dst[ 0] = c;  dst[ 1] = 0;  dst[ 2] = -s;  dst[ 3] = 0;
    dst[ 4] = 0;  dst[ 5] = 1;  dst[ 6] = 0;   dst[ 7] = 0;
    dst[ 8] = s;  dst[ 9] = 0;  dst[10] = c;   dst[11] = 0;
    dst[12] = 0;  dst[13] = 0;  dst[14] = 0;   dst[15] = 1;
    return dst;
  },

  rotationZ(angleInRadians, dst) {
    const c = Math.cos(angleInRadians);
    const s = Math.sin(angleInRadians);
    dst = dst || new Float32Array(16);
    dst[ 0] = c;   dst[ 1] = s;  dst[ 2] = 0;  dst[ 3] = 0;
    dst[ 4] = -s;  dst[ 5] = c;  dst[ 6] = 0;  dst[ 7] = 0;
    dst[ 8] = 0;   dst[ 9] = 0;  dst[10] = 1;  dst[11] = 0;
    dst[12] = 0;   dst[13] = 0;  dst[14] = 0;  dst[15] = 1;
    return dst;
  },

  scaling([sx, sy, sz], dst) {
    dst = dst || new Float32Array(16);
    dst[ 0] = sx;  dst[ 1] = 0;   dst[ 2] = 0;    dst[ 3] = 0;
    dst[ 4] = 0;   dst[ 5] = sy;  dst[ 6] = 0;    dst[ 7] = 0;
    dst[ 8] = 0;   dst[ 9] = 0;   dst[10] = sz;   dst[11] = 0;
    dst[12] = 0;   dst[13] = 0;   dst[14] = 0;    dst[15] = 1;
    return dst;
  },
  ...
```

同样地，
我们也会做一组简化后的辅助函数。
先看 2D 的版本。

```js
  translate(m, translation, dst) {
    return mat3.multiply(m, mat3.translation(translation), dst);
  },

  rotate(m, angleInRadians, dst) {
    return mat3.multiply(m, mat3.rotation(angleInRadians), dst);
  },

  scale(m, scale, dst) {
    return mat3.multiply(m, mat3.scaling(scale), dst);
  },
```

再来看 3D 版本。
变化并不大，
除了名字换成 `mat4`，
以及增加了另外两个旋转函数。

```js
  translate(m, translation, dst) {
    return mat4.multiply(m, mat4.translation(translation), dst);
  },

  rotateX(m, angleInRadians, dst) {
    return mat4.multiply(m, mat4.rotationX(angleInRadians), dst);
  },

  rotateY(m, angleInRadians, dst) {
    return mat4.multiply(m, mat4.rotationY(angleInRadians), dst);
  },

  rotateZ(m, angleInRadians, dst) {
    return mat4.multiply(m, mat4.rotationZ(angleInRadians), dst);
  },

  scale(m, scale, dst) {
    return mat4.scaling(m, mat4.scaling(scale), dst);
  },
  ...
```

我们还需要一个 4x4 的矩阵乘法函数

```js
  multiply(a, b, dst) {
    dst = dst || new Float32Array(16);
    const b00 = b[0 * 4 + 0];
    const b01 = b[0 * 4 + 1];
    const b02 = b[0 * 4 + 2];
    const b03 = b[0 * 4 + 3];
    const b10 = b[1 * 4 + 0];
    const b11 = b[1 * 4 + 1];
    const b12 = b[1 * 4 + 2];
    const b13 = b[1 * 4 + 3];
    const b20 = b[2 * 4 + 0];
    const b21 = b[2 * 4 + 1];
    const b22 = b[2 * 4 + 2];
    const b23 = b[2 * 4 + 3];
    const b30 = b[3 * 4 + 0];
    const b31 = b[3 * 4 + 1];
    const b32 = b[3 * 4 + 2];
    const b33 = b[3 * 4 + 3];
    const a00 = a[0 * 4 + 0];
    const a01 = a[0 * 4 + 1];
    const a02 = a[0 * 4 + 2];
    const a03 = a[0 * 4 + 3];
    const a10 = a[1 * 4 + 0];
    const a11 = a[1 * 4 + 1];
    const a12 = a[1 * 4 + 2];
    const a13 = a[1 * 4 + 3];
    const a20 = a[2 * 4 + 0];
    const a21 = a[2 * 4 + 1];
    const a22 = a[2 * 4 + 2];
    const a23 = a[2 * 4 + 3];
    const a30 = a[3 * 4 + 0];
    const a31 = a[3 * 4 + 1];
    const a32 = a[3 * 4 + 2];
    const a33 = a[3 * 4 + 3];

    dst[0] = b00 * a00 + b01 * a10 + b02 * a20 + b03 * a30;
    dst[1] = b00 * a01 + b01 * a11 + b02 * a21 + b03 * a31;
    dst[2] = b00 * a02 + b01 * a12 + b02 * a22 + b03 * a32;
    dst[3] = b00 * a03 + b01 * a13 + b02 * a23 + b03 * a33;

    dst[4] = b10 * a00 + b11 * a10 + b12 * a20 + b13 * a30;
    dst[5] = b10 * a01 + b11 * a11 + b12 * a21 + b13 * a31;
    dst[6] = b10 * a02 + b11 * a12 + b12 * a22 + b13 * a32;
    dst[7] = b10 * a03 + b11 * a13 + b12 * a23 + b13 * a33;

    dst[8] = b20 * a00 + b21 * a10 + b22 * a20 + b23 * a30;
    dst[9] = b20 * a01 + b21 * a11 + b22 * a21 + b23 * a31;
    dst[10] = b20 * a02 + b21 * a12 + b22 * a22 + b23 * a32;
    dst[11] = b20 * a03 + b21 * a13 + b22 * a23 + b23 * a33;

    dst[12] = b30 * a00 + b31 * a10 + b32 * a20 + b33 * a30;
    dst[13] = b30 * a01 + b31 * a11 + b32 * a21 + b33 * a31;
    dst[14] = b30 * a02 + b31 * a12 + b32 * a22 + b33 * a32;
    dst[15] = b30 * a03 + b31 * a13 + b32 * a23 + b33 * a33;

    return dst;
  },
```

我们还需要更新 projection 函数。
下面是之前的版本

```js
  projection(width, height, dst) {
    // Note: This matrix flips the Y axis so that 0 is at the top.
    dst = dst || new Float32Array(12);
    dst[0] = 2 / width;  dst[1] = 0;             dst[2] = 0;
    dst[4] = 0;          dst[5] = -2 / height;   dst[6] = 0;
    dst[8] = -1;         dst[9] = 1;             dst[10] = 1;
    return dst;
  },
```

它的作用是把像素空间转换到 clip space。
现在我们第一次尝试把它扩展到 3D，
可以先这样写

```js
  projection(width, height, depth, dst) {
    // Note: This matrix flips the Y axis so that 0 is at the top.
    dst = dst || new Float32Array(16);
    dst[ 0] = 2 / width;  dst[ 1] = 0;            dst[ 2] = 0;            dst[ 3] = 0;
    dst[ 4] = 0;          dst[ 5] = -2 / height;  dst[ 6] = 0;            dst[ 7] = 0;
    dst[ 8] = 0;          dst[ 9] = 0;            dst[10] = 0.5 / depth;  dst[11] = 0;
    dst[12] = -1;         dst[13] = 1;            dst[14] = 0.5;          dst[15] = 1;
    return dst;
  },
```

就像 X 和 Y
需要从像素空间映射到 clip space 一样，
Z 也一样要做对应转换。
在这里，
我们让 Z 轴
也使用类似“像素单位”的概念。
我们会像给 `width`
那样传入一个 `depth` 值，
这样我们的空间会是：
宽度从 0 到 `width` 像素，
高度从 0 到 `height` 像素，
而深度则从 `-depth / 2`
到 `+depth / 2`。

我们的 uniform 中也需要提供一个 4x4 矩阵

```js
  // color, matrix
-  const uniformBufferSize = (4 + 12) * 4;
+  const uniformBufferSize = (4 + 16) * 4;
  const uniformBuffer = device.createBuffer({
    label: 'uniforms',
    size: uniformBufferSize,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
  });

  const uniformValues = new Float32Array(uniformBufferSize / 4);

  // offsets to the various uniform values in float32 indices
  const kColorOffset = 0;
  const kMatrixOffset = 4;

  const colorValue = uniformValues.subarray(kColorOffset, kColorOffset + 4);
-  const matrixValue = uniformValues.subarray(kMatrixOffset, kMatrixOffset + 12);
+  const matrixValue = uniformValues.subarray(kMatrixOffset, kMatrixOffset + 16);
```

还需要更新计算矩阵的代码

```js
 const settings = {
-    translation: [150, 100],
-    rotation: degToRad(30),
-    scale: [1, 1],
+    translation: [45, 100, 0],
+    rotation: [degToRad(40), degToRad(25), degToRad(325)],
+    scale: [1, 1, 1],
  };

  ...

  function render() {
    ...

-    mat3.projection(canvas.clientWidth, canvas.clientHeight, matrixValue);
-    mat3.translate(matrixValue, settings.translation, matrixValue);
-    mat3.rotate(matrixValue, settings.rotation, matrixValue);
-    mat3.scale(matrixValue, settings.scale, matrixValue);
+    mat4.projection(canvas.clientWidth, canvas.clientHeight, 400, matrixValue);
+    mat4.translate(matrixValue, settings.translation, matrixValue);
+    mat4.rotateX(matrixValue, settings.rotation[0], matrixValue);
+    mat4.rotateY(matrixValue, settings.rotation[1], matrixValue);
+    mat4.rotateZ(matrixValue, settings.rotation[2], matrixValue);
+    mat4.scale(matrixValue, settings.scale, matrixValue);
```

{{{example url="../webgpu-orthographic-projection-step-1-flat-f.html"}}}

现在遇到的第一个问题是：
我们的数据仍然是一个扁平的 F，
所以很难看出任何 3D 效果。
为了修复这个问题，
我们需要把数据扩展成真正的 3D。
当前这个 F
由 3 个矩形组成，
每个矩形由 2 个三角形构成。
如果要把它变成 3D，
总共会需要 16 个矩形：
正面 3 个、背面 3 个、左侧 1 个、右侧 4 个、
顶部 2 个、底部 3 个。

<img class="webgpu_center noinvertdark" style="width: 400px;" src="resources/3df.svg" />

我们只需要把当前所有顶点位置复制一份，
然后沿 Z 方向平移，
再用索引把它们全部连接起来即可。

```js
function createFVertices() {
  const vertexData = new Float32Array([
    // left column
    0, 0, 0,
    30, 0, 0,
    0, 150, 0,
    30, 150, 0,

    // top rung
    30, 0, 0,
    100, 0, 0,
    30, 30, 0,
    100, 30, 0,

    // middle rung
    30, 60, 0,
    70, 60, 0,
    30, 90, 0,
    70, 90, 0,

+    // left column back
+    0, 0, 30,
+    30, 0, 30,
+    0, 150, 30,
+    30, 150, 30,
+
+    // top rung back
+    30, 0, 30,
+    100, 0, 30,
+    30, 30, 30,
+    100, 30, 30,
+
+    // middle rung back
+    30, 60, 30,
+    70, 60, 30,
+    30, 90, 30,
+    70, 90, 30,
  ]);

  const indexData = new Uint32Array([
+    // front
    0,  1,  2,    2,  1,  3,  // left column
    4,  5,  6,    6,  5,  7,  // top run
    8,  9, 10,   10,  9, 11,  // middle run

+    // back
+    12,  13,  14,   14, 13, 15,  // left column back
+    16,  17,  18,   18, 17, 19,  // top run back
+    20,  21,  22,   22, 21, 23,  // middle run back
+
+    0, 5, 12,   12, 5, 17,   // top
+    5, 7, 17,   17, 7, 19,   // top rung right
+    6, 7, 18,   18, 7, 19,   // top rung bottom
+    6, 8, 18,   18, 8, 20,   // between top and middle rung
+    8, 9, 20,   20, 9, 21,   // middle rung top
+    9, 11, 21,  21, 11, 23,  // middle rung right
+    10, 11, 22, 22, 11, 23,  // middle rung bottom
+    10, 3, 22,  22, 3, 15,   // stem right
+    2, 3, 14,   14, 3, 15,   // bottom
+    0, 2, 12,   12, 2, 14,   // left
  ]);

  return {
    vertexData,
    indexData,
    numVertices: indexData.length,
  };
}
```

这一版的效果如下

{{{example url="../webgpu-orthographic-projection-step-2-3d-f.html"}}}

拖动这些滑块时，其实很难明显看出它已经是 3D 了。我们试着给每个矩形涂上不同的颜色。为此，我们要在顶点着色器里再添加一个 attribute，并通过一个[阶段间变量](webgpu-inter-stage-variables.html)把它从顶点着色器传到片元着色器。

首先更新 shader：

```wgsl
struct Uniforms {
-  color: vec4f,
  matrix: mat4x4f,
};

struct Vertex {
  @location(0) position: vec4f,
+  @location(1) color: vec4f,
};

struct VSOutput {
  @builtin(position) position: vec4f,
+  @location(0) color: vec4f,
};

@group(0) @binding(0) var<uniform> uni: Uniforms;

@vertex fn vs(vert: Vertex) -> VSOutput {
  var vsOut: VSOutput;
  vsOut.position = uni.matrix * vert.position;
+  vsOut.color = vert.color;
  return vsOut;
}

@fragment fn fs(vsOut: VSOutput) -> @location(0) vec4f {
-  return uni.color;
+  return vsOut.color;
}
```

我们需要把颜色加入顶点数据里，不过这里有个问题。当前我们使用索引来共享顶点。但如果我们希望每个面都有不同的颜色，那么这些顶点就不能再共享了，因为每个顶点只能携带一种颜色。

<img src="resources/cube-faces-vertex-no-texture.svg" class="webgpu_center" style="width:400px;" />

上图里的角点顶点会被它共享的 3 个面分别使用一次，但它每次都需要不同的颜色，因此继续使用索引就会变得很麻烦。[^flat-interpolation]

[^flat-interpolation]: 当然，也可以通过更巧妙地安排索引，再配合[阶段间变量那篇文章](webgpu-inter-stage-varaibles.html#a-interpolate)中提到的 `@interpolate(flat)`，依然使用索引。

所以，我们把数据从索引形式展开成非索引形式。顺便把顶点颜色也一起加进去，这样 F 的每个部分都能拥有不同颜色。

```js
function createFVertices() {
  const positions = [
    // left column front
     0,   0,   0,
    30,   0,   0,
     0, 150,   0,
    30, 150,   0,

    // top rung front
    30,   0,   0,
   100,   0,   0,
    30,  30,   0,
   100,  30,   0,

    // middle rung front
    30,  60,   0,
    70,  60,   0,
    30,  90,   0,
    70,  90,   0,

    // left column back
     0,   0,  30,
    30,   0,  30,
     0, 150,  30,
    30, 150,  30,

    // top rung back
    30,   0,  30,
   100,   0,  30,
    30,  30,  30,
   100,  30,  30,

    // middle rung back
    30,  60,  30,
    70,  60,  30,
    30,  90,  30,
    70,  90,  30,
  ];

  const indices = [
    // front
    0,  1,  2,    2,  1,  3,  // left column
    4,  5,  6,    6,  5,  7,  // top run
    8,  9, 10,   10,  9, 11,  // middle run

    // back
    12,  13,  14,   14, 13, 15,  // left column back
    16,  17,  18,   18, 17, 19,  // top run back
    20,  21,  22,   22, 21, 23,  // middle run back

    0, 5, 12,   12, 5, 17,   // top
    5, 7, 17,   17, 7, 19,   // top rung right
    6, 7, 18,   18, 7, 19,   // top rung bottom
    6, 8, 18,   18, 8, 20,   // between top and middle rung
    8, 9, 20,   20, 9, 21,   // middle rung top
    9, 11, 21,  21, 11, 23,  // middle rung right
    10, 11, 22, 22, 11, 23,  // middle rung bottom
    10, 3, 22,  22, 3, 15,   // stem right
    2, 3, 14,   14, 3, 15,   // bottom
    0, 2, 12,   12, 2, 14,   // left
  ];

  const quadColors = [
      200,  70, 120,  // left column front
      200,  70, 120,  // top rung front
      200,  70, 120,  // middle rung front

       80,  70, 200,  // left column back
       80,  70, 200,  // top rung back
       80,  70, 200,  // middle rung back

       70, 200, 210,  // top
      160, 160, 220,  // top rung right
       90, 130, 110,  // top rung bottom
      200, 200,  70,  // between top and middle rung
      210, 100,  70,  // middle rung top
      210, 160,  70,  // middle rung right
       70, 180, 210,  // middle rung bottom
      100,  70, 210,  // stem right
       76, 210, 100,  // bottom
      140, 210,  80,  // left
  ];

  const numVertices = indices.length;
  const vertexData = new Float32Array(numVertices * 4); // xyz + color
  const colorData = new Uint8Array(vertexData.buffer);

  for (let i = 0; i < indices.length; ++i) {
    const positionNdx = indices[i] * 3;
    const position = positions.slice(positionNdx, positionNdx + 3);
    vertexData.set(position, i * 4);

    const quadNdx = (i / 6 | 0) * 3;
    const color = quadColors.slice(quadNdx, quadNdx + 3);
    colorData.set(color, i * 16 + 12);  // set RGB
    colorData[i * 16 + 15] = 255;       // set A
  }

  return {
    vertexData,
    numVertices,
  };
}
```

我们遍历每个索引，取出该索引对应的位置数据，并把位置值写进 `vertexData`。同时我们又创建了一个指向同一块底层数据的视图 `colorData`，这样就可以按矩形面的索引来取颜色值了。每 6 个顶点对应一个矩形面，所以我们给这一组 6 个顶点都写入相同颜色。最终数据会长这样：

<img class="webgpu_center" style="background-color: transparent; width: 1024px;" src="resources/vertex-buffer-f32x3-u8x4.svg" />

我们添加的颜色是 0 到 255 范围内的无符号字节，和 [CSS 的 `rgb()` 颜色](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/rgb) 很像。
由于我们在 pipeline 里把该属性的类型设成了 `unorm8x4`（unsigned normalized 8 bit value x 4），GPU 在从缓冲区取出这些值并传给 shader 时，会自动把它们归一化。也就是说，这些值会被转换到 0 到 1 的范围内，这里具体就是除以 255。

现在既然有了这些数据，我们就需要修改 pipeline 来使用它。

```js
  const pipeline = device.createRenderPipeline({
    label: '2 attributes',
    layout: 'auto',
    vertex: {
      module,
      buffers: [
        {
-          arrayStride: (3) * 4, // (3) floats, 4 bytes each
+          arrayStride: (4) * 4, // (3) floats 4 bytes each + one 4 byte color
          attributes: [
            {shaderLocation: 0, offset: 0, format: 'float32x3'},  // position
+            {shaderLocation: 1, offset: 12, format: 'unorm8x4'},  // color
          ],
        },
      ],
    },
    fragment: {
      module,
      targets: [{ format: presentationFormat }],
    },
  });
```

我们也不再需要创建索引缓冲区了。

```js
-  const { vertexData, indexData, numVertices } = createFVertices();
+  const { vertexData, numVertices } = createFVertices();
  const vertexBuffer = device.createBuffer({
    label: 'vertex buffer vertices',
    size: vertexData.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST,
  });
  device.queue.writeBuffer(vertexBuffer, 0, vertexData);
-  const indexBuffer = device.createBuffer({
-    label: 'index buffer',
-    size: indexData.byteLength,
-    usage: GPUBufferUsage.INDEX | GPUBufferUsage.COPY_DST,
-  });
-  device.queue.writeBuffer(indexBuffer, 0, indexData);
```

同时绘制时也要改成非索引绘制：

```js
 function render() {
    ...
    pass.setPipeline(pipeline);
    pass.setVertexBuffer(0, vertexBuffer);
-    pass.setIndexBuffer(indexBuffer, 'uint32');

    ...

    pass.setBindGroup(0, bindGroup);
-    pass.drawIndexed(numVertices);
+    pass.draw(numVertices);

    ...
  }
```

现在我们会得到下面这个效果：

{{{example url="../webgpu-orthographic-projection-step-3-colored-3d-f.html"}}}

呃，这是什么情况？问题在于，这个 3D “F” 的各个部分，比如前面、后面、侧面等等，会按照它们在几何数据中出现的顺序被依次绘制。这样有时就会出现后面的面反而在前面的面之后绘制，从而把前面的内容盖住。

<img class="webgpu_center" style="background-color: transparent; width: 163px;" src="resources/polygon-drawing-order.gif" />

这里带有 <span style="background: rgb(200, 70, 120); color: white; padding: 0.25em">偏红色</span> 的部分其实是 “F” 的**正面**，但因为它在数据里最先出现，所以最先被绘制；接着位于它后面的其它三角形又被绘制出来，把它覆盖掉了。比如 <span style="background: rgb(80, 70, 200); color: white; padding: 0.25em">紫色</span> 的部分实际上是 “F” 的背面，它之所以第二个被画出来，只是因为它在数据里排第二。

在 WebGPU 中，三角形有正面和背面的概念。默认情况下，如果一个三角形在裁剪空间中的顶点顺序是逆时针方向，它就被认为是正面；如果顶点顺序是顺时针方向，它就被认为是背面。

<img src="resources/triangle-winding.svg" class="webgpu_center" style="width: 400px;" />

GPU 可以只绘制正面三角形，或者只绘制背面三角形。我们可以通过修改 pipeline 来开启这个功能：

```js
  const pipeline = device.createRenderPipeline({
    label: '2 attributes',
    layout: 'auto',
    vertex: {
      module,
      buffers: [
        {
          arrayStride: (4) * 4, // (3) floats 4 bytes each + one 4 byte color
          attributes: [
            {shaderLocation: 0, offset: 0, format: 'float32x3'},  // position
            {shaderLocation: 1, offset: 12, format: 'unorm8x4'},  // color
          ],
        },
      ],
    },
    fragment: {
      module,
      targets: [{ format: presentationFormat }],
    },
+    primitive: {
+      cullMode: 'back',
+    },
  });
```

把 `cullMode` 设为 `back` 后，所有“背向”的三角形都会被剔除。
这里的 “culling” 可以简单理解成“不要绘制”。
因此，当 `cullMode` 设为 `'back'` 时，效果如下：

{{{example url="../webgpu-orthographic-projection-step-4-cullmode-back.html"}}}

咦，三角形怎么少了这么多？原因是其中很多三角形的朝向写反了。把模型转一转，你会发现从另一侧看时它们又出现了。幸运的是，这很好修复。我们只要找出那些方向反了的三角形，然后交换其中两个顶点即可。比如某个方向错误的三角形索引是：

<div class="webgpu_center"><pre class="webgpu_math">
6, 7, 8,
</pre></div>

那我们只要交换其中两个索引，就能让它朝向反过来：

<div class="webgpu_center"><pre class="webgpu_math">
6, 8, 7,
</pre></div>

这里有个很重要的点。对于 WebGPU 来说，一个三角形究竟算顺时针还是逆时针，是看这个三角形在裁剪空间中的顶点顺序来决定的。换句话说，WebGPU 是在顶点经过顶点着色器中的矩阵变换之后，才判断它到底是正面还是背面。

这意味着，例如一个顺时针三角形如果在 X 方向上缩放了 `-1`，它就会变成逆时针；又或者一个顺时针三角形如果旋转了 180 度，也会变成逆时针。之前因为我们没有设置 `cullMode`，所以无论是顺时针（正面）还是逆时针（背面）三角形都能看到。现在一旦把 `cullMode` 设为 `back`，任何一个原本正面的三角形只要因为缩放、旋转或其它原因翻转了朝向，WebGPU 就不会绘制它。

这其实正是我们想要的效果，因为在 3D 中旋转一个物体时，通常你会希望当前朝向观察者的那些三角形被当作正面。

但是！还记得吗，在裁剪空间里，`+Y` 朝上；而在我们的像素空间里，`+Y` 朝下。也就是说，我们的矩阵实际上把所有三角形在垂直方向上翻转了。所以，如果我们想继续用 `+Y` 朝下的坐标系来绘制，要么把 `cullMode` 设为 `'front'`，要么把所有三角形的顶点顺序整体翻转。

这里我们把 `cullMode` 改成 `'front'`，同时也修正顶点数据，让所有三角形都保持一致的朝向。

```js
  const indices = [
    // front
    0,  1,  2,    2,  1,  3,  // left column
    4,  5,  6,    6,  5,  7,  // top run
    8,  9, 10,   10,  9, 11,  // middle run

    // back
-    12,  13,  14,   14, 13, 15,  // left column back
+    12,  14,  13,   14, 15, 13,  // left column back
-    16,  17,  18,   18, 17, 19,  // top run back
+    16,  18,  17,   18, 19, 17,  // top run back
-    20,  21,  22,   22, 21, 23,  // middle run back
+    20,  22,  21,   22, 23, 21,  // middle run back

-    0, 5, 12,   12, 5, 17,   // top
+    0, 12, 5,   12, 17, 5,   // top
-    5, 7, 17,   17, 7, 19,   // top rung right
+    5, 17, 7,   17, 19, 7,   // top rung right
    6, 7, 18,   18, 7, 19,   // top rung bottom
-    6, 8, 18,   18, 8, 20,   // between top and middle rung
+    6, 18, 8,   18, 20, 8,   // between top and middle rung
-    8, 9, 20,   20, 9, 21,   // middle rung top
+    8, 20, 9,   20, 21, 9,   // middle rung top
-    9, 11, 21,  21, 11, 23,  // middle rung right
+    9, 21, 11,  21, 23, 11,  // middle rung right
    10, 11, 22, 22, 11, 23,  // middle rung bottom
-    10, 3, 22,  22, 3, 15,   // stem right
+    10, 22, 3,  22, 15, 3,   // stem right
    2, 3, 14,   14, 3, 15,   // bottom
    0, 2, 12,   12, 2, 14,   // left
  ];
```

```js
  const pipeline = device.createRenderPipeline({
    ...
    primitive: {
-      cullMode: 'back',
+      cullMode: 'front',
    },
  });
```

经过这些修改，让所有三角形都朝向一致后，我们得到如下结果：

{{{example url="../webgpu-orthographic-projection-step-5-order-fixed.html"}}}

这已经更接近正确结果了，但还有最后一个问题。即使现在所有三角形的朝向都正确了，而且背向我们的三角形也被剔除了，仍然会有一些本该在后面的三角形盖住本该在前面的三角形。

## <a id="a-depth-textures"></a>引入“深度纹理”

深度纹理，有时也叫深度缓冲（depth-buffer）或者 Z-Buffer，本质上是一张由*深度*纹素组成的矩形纹理。颜色纹理中的每一个颜色纹素，都对应深度纹理中的一个深度纹素。
如果我们创建并绑定一张深度纹理，那么 WebGPU 在绘制每个像素时，也可以同时写入一个深度像素。它使用的是顶点着色器输出结果中的 Z 值。就像 X 和 Y 需要转换到裁剪空间一样，Z 也同样位于裁剪空间。对于 Z 来说，裁剪空间范围是 0 到 +1。

在 WebGPU 绘制一个颜色像素之前，它会先检查对应的深度像素。如果当前要绘制的那个像素的深度（Z）值，相对于深度纹理里已经存在的值，不满足某个条件，那么 WebGPU 就不会绘制新的颜色像素。否则，它不仅会绘制新的颜色像素，还会把 fragment shader 产生的颜色写入颜色纹理，并把新的深度值写入深度纹理。

这意味着，位于其它像素后方的像素将不会被画出来。

要设置并使用深度纹理，我们需要先更新 pipeline：

```js
  const pipeline = device.createRenderPipeline({
    label: '2 attributes',
    layout: 'auto',
    vertex: {
      module,
      buffers: [
        {
          arrayStride: (4) * 4, // (3) floats 4 bytes each + one 4 byte color
          attributes: [
            {shaderLocation: 0, offset: 0, format: 'float32x3'},  // position
            {shaderLocation: 1, offset: 12, format: 'unorm8x4'},  // color
          ],
        },
      ],
    },
    fragment: {
      module,
      targets: [{ format: presentationFormat }],
    },
    primitive: {
      cullMode: 'front',
    },
+    depthStencil: {
+      depthWriteEnabled: true,
+      depthCompare: 'less',
+      format: 'depth24plus',
+    },
  });
```

上面我们设置了 `depthCompare: 'less'`。这表示：只有当新像素的 Z 值“小于”深度纹理中对应像素的当前值时，才绘制这个新像素。其它可选值还包括 `never`、`equal`、`less-equal`、`greater`、`not-equal`、`greater-equal` 和 `always`。

`depthWriteEnabled: true` 表示：如果当前像素通过了 `depthCompare` 测试，就把它的 Z 值写入深度纹理。在我们的例子里，这意味着每当正在绘制的像素 Z 值小于深度纹理中已有值时，这个像素就会被绘制，并且深度纹理也会被更新。这样一来，如果之后又尝试绘制一个更靠后的像素（也就是 Z 值更大），它就不会被绘制出来。

`format` 的作用类似于 `fragment.targets[?].format`，它表示我们要使用的深度纹理格式。可用的深度纹理格式已经在[纹理那篇文章](webgpu-textures.html#a-depth-stencil-formats)中列出来了。`depth24plus` 是一个很合适的默认选择。

我们还需要更新 render pass descriptor，让它带上一个深度模板附件（depth stencil attachment）。

```js
  const renderPassDescriptor = {
    label: 'our basic canvas renderPass',
    colorAttachments: [
      {
        // view: <- to be filled out when we render
        loadOp: 'clear',
        storeOp: 'store',
      },
    ],
+    depthStencilAttachment: {
+      // view: <- to be filled out when we render
+      depthClearValue: 1.0,
+      depthLoadOp: 'clear',
+      depthStoreOp: 'store',
+    },
  };
```

深度值通常位于 0.0 到 1.0 之间。这里我们把 `depthClearValue` 设为 1，这很合理，因为前面我们把 `depthCompare` 设成了 `less`。

最后，我们还需要创建一张深度纹理。这里有个需要注意的地方：它的尺寸必须和颜色附件一致；而在这个例子中，颜色附件就是我们从 canvas 拿到的那张纹理。

当我们在 `ResizeObserver` 回调里修改 canvas 大小时，canvas 对应的纹理尺寸也会变化。更准确地说，每次调用 `context.getCurrentTexture()` 得到的那张纹理，尺寸都会等于当前 canvas 的尺寸。考虑到这一点，我们就在渲染时动态创建正确大小的深度纹理。

```js
+  let depthTexture;

  function render() {
    // Get the current texture from the canvas context and
    // set it as the texture to render to.
-    renderPassDescriptor.colorAttachments[0].view =
-        context.getCurrentTexture().createView();
+    const canvasTexture = context.getCurrentTexture();
+    renderPassDescriptor.colorAttachments[0].view = canvasTexture.createView();

+    // If we don't have a depth texture OR if its size is different
+    // from the canvasTexture when make a new depth texture
+    if (!depthTexture ||
+        depthTexture.width !== canvasTexture.width ||
+        depthTexture.height !== canvasTexture.height) {
+      if (depthTexture) {
+        depthTexture.destroy();
+      }
+      depthTexture = device.createTexture({
+        size: [canvasTexture.width, canvasTexture.height],
+        format: 'depth24plus',
+        usage: GPUTextureUsage.RENDER_ATTACHMENT,
+      });
+    }
+    renderPassDescriptor.depthStencilAttachment.view = depthTexture.createView();

  ...
```

加入深度纹理之后，我们得到：

{{{example url="../webgpu-orthographic-projection-step-6-depth-texture.html"}}}

这就是真正的 3D 了！

## Ortho / Orthographic（正交）

还有一个小点要说明。大多数 3D 数学库里，并不会有一个叫 `projection` 的函数专门用来做从裁剪空间到像素空间的这种转换。更常见的做法是提供一个名为 `ortho` 或 `orthographic` 的函数，大概像这样：

```js
const mat4 = {
  ...
  ortho(left, right, bottom, top, near, far, dst) {
    dst = dst || new Float32Array(16);

    dst[0] = 2 / (right - left);
    dst[1] = 0;
    dst[2] = 0;
    dst[3] = 0;

    dst[4] = 0;
    dst[5] = 2 / (top - bottom);
    dst[6] = 0;
    dst[7] = 0;

    dst[8] = 0;
    dst[9] = 0;
    dst[10] = 1 / (near - far);
    dst[11] = 0;

    dst[12] = (right + left) / (left - right);
    dst[13] = (top + bottom) / (bottom - top);
    dst[14] = near / (near - far);
    dst[15] = 1;

    return dst;
  },
  ...
```

和前面那个简化版的 `projection` 函数不同，后者只有 width、height 和 depth 三个参数；而这个更常见的正交投影函数可以传入 left、right、bottom、top、near 和 far，因此更加灵活。

如果想让它实现和我们原来 `projection` 函数相同的效果，可以这样调用：

```js
-    mat4.projection(canvas.clientWidth, canvas.clientHeight, 400, matrixValue);
+    mat4.ortho(
+        0,                   // left
+        canvas.clientWidth,  // right
+        canvas.clientHeight, // bottom
+        0,                   // top
+        200,                 // near
+        -200,                // far
+        matrixValue,         // dst
+    );   
```

{{{example url="../webgpu-orthographic-projection-step-7-ortho.html"}}}

下一篇我们会讲[如何让它具备透视效果](webgpu-perspective-projection.html)。

<div class="webgpu_bottombar">
<h3>为什么叫 orthographic projection？</h3>
<p>
这里的 Orthographic 来自单词 <i>orthogonal</i>（正交的）
</p>
<blockquote>
<h2>orthogonal</h2>
<p><i>形容词</i>：</p>
<ol><li>正交的；涉及直角的</li></ol>
</blockquote>
</div>

<!-- keep this at the bottom of the article -->
<link href="webgpu-orthographic-projection.css" rel="stylesheet">
<script type="module" src="webgpu-orthographic-projection.js"></script>
