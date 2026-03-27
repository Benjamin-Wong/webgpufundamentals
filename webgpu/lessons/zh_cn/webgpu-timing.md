Title: WebGPU 性能计时
Description: 在 WebGPU 中对操作进行计时
TOC: 性能计时

我们来看看，在做性能分析时，
有哪些内容可能值得计时。
这里我们会测 3 件事：

* 每秒帧数，也就是 fps
* 每帧花在 JavaScript 上的时间
* 每帧花在 GPU 上的时间

首先，我们拿
[顶点缓冲区那篇文章](webgpu-vertex-buffers.html)
里的圆形示例做基础，
并让它们动起来，
这样就有了一个很容易看出性能变化的例子。

在那个示例里，我们有 3 个 vertex buffer。
一个用来存圆形顶点的位置和亮度。
一个用来存“每个实例都有，但不会变化”的数据，
其中包括圆的偏移量和颜色。
最后一个用来存每次渲染都会变化的数据，
在原来的例子里，
它存的是缩放值，
这样我们就能保持圆的纵横比正确，
保证它们在用户缩放窗口时仍然是圆，
而不会变成椭圆。

我们想让它们动起来，
那就把 offset
也挪到和 scale 同一个 buffer 里。
首先先修改 render pipeline，
让 offset 和 scale
使用同一个 buffer。

```js
  const pipeline = device.createRenderPipeline({
    label: 'per vertex color',
    layout: 'auto',
    vertex: {
      module,
      buffers: [
        {
          arrayStride: 2 * 4 + 4, // 2 floats, 4 bytes each + 4 bytes
          attributes: [
            {shaderLocation: 0, offset: 0, format: 'float32x2'},  // position
            {shaderLocation: 4, offset: 8, format: 'unorm8x4'},   // perVertexColor
          ],
        },
        {
-          arrayStride: 4 + 2 * 4, // 4 bytes + 2 floats, 4 bytes each
+          arrayStride: 4, // 4 bytes
          stepMode: 'instance',
          attributes: [
            {shaderLocation: 1, offset: 0, format: 'unorm8x4'},   // color
-            {shaderLocation: 2, offset: 4, format: 'float32x2'},  // offset
          ],
        },
        {
-          arrayStride: 2 * 4, // 2 floats, 4 bytes each
+          arrayStride: 4 * 4, // 4 floats, 4 bytes each
          stepMode: 'instance',
          attributes: [
-            {shaderLocation: 3, offset: 0, format: 'float32x2'},   // scale
+            {shaderLocation: 2, offset: 0, format: 'float32x2'},  // offset
-            {shaderLocation: 3, offset: 0, format: 'float32x2'},   // scale
+            {shaderLocation: 3, offset: 8, format: 'float32x2'},   // scale
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

接下来，再修改设置 vertex buffer 的那部分代码，
把 offset 和 scale
一起放到同一个 changing buffer 里。

```js
  // create 2 vertex buffers
  const staticUnitSize =
-    4 +     // color is 4 bytes
-    2 * 4;  // offset is 2 32bit floats (4bytes each)
+    4;     // color is 4 bytes
  const changingUnitSize =
-    2 * 4;  // scale is 2 32bit floats (4bytes each)
+    2 * 4 + // offset is 2 32bit floats (4bytes each)
+    2 * 4;  // scale is 2 32bit floats (4bytes each)
  const staticVertexBufferSize = staticUnitSize * kNumObjects;
  const changingVertexBufferSize = changingUnitSize * kNumObjects;

  const staticVertexBuffer = device.createBuffer({
    label: 'static vertex for objects',
    size: staticVertexBufferSize,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST,
  });

  const changingVertexBuffer = device.createBuffer({
    label: 'changing storage for objects',
    size: changingVertexBufferSize,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST,
  });

  // offsets to the various uniform values in float32 indices
  const kColorOffset = 0;
-  const kOffsetOffset = 1;
+
-  const kScaleOffset = 0;
+  const kOffsetOffset = 0;
+  const kScaleOffset = 2;

  {
    const staticVertexValuesU8 = new Uint8Array(staticVertexBufferSize);
-    const staticVertexValuesF32 = new Float32Array(staticVertexValuesU8.buffer);
    for (let i = 0; i < kNumObjects; ++i) {
      const staticOffsetU8 = i * staticUnitSize;
-      const staticOffsetF32 = staticOffsetU8 / 4;

      // These are only set once so set them now
      staticVertexValuesU8.set(        // set the color
          [rand() * 255, rand() * 255, rand() * 255, 255],
          staticOffsetU8 + kColorOffset);

-      staticVertexValuesF32.set(      // set the offset
-          [rand(-0.9, 0.9), rand(-0.9, 0.9)],
-          staticOffsetF32 + kOffsetOffset);

      objectInfos.push({
        scale: rand(0.2, 0.5),
+        offset: [rand(-0.9, 0.9), rand(-0.9, 0.9)],
+        velocity: [rand(-0.1, 0.1), rand(-0.1, 0.1)],
      });
    }
-    device.queue.writeBuffer(staticVertexBuffer, 0, staticVertexValuesF32);
+    device.queue.writeBuffer(staticVertexBuffer, 0, staticVertexValuesU8);
  }
```

到了 render 阶段，
我们就可以根据圆的速度更新它们的 offset，
再把这些数据上传给 GPU。

```js
+  const euclideanModulo = (x, a) => x - a * Math.floor(x / a);

+  let then = 0;
-  function render() {
  function render(now) {
+    now *= 0.001;  // convert to seconds
+    const deltaTime = now - then;
+    then = now;

...
      // set the scales for each object
-    objectInfos.forEach(({scale}, ndx) => {
-      const offset = ndx * (changingUnitSize / 4);
-      vertexValues.set([scale / aspect, scale], offset + kScaleOffset); // set the scale
+    objectInfos.forEach(({scale, offset, veloctiy}, ndx) => {
+      // -1.5 to 1.5
+      offset[0] = euclideanModulo(offset[0] + velocity[0] * deltaTime + 1.5, 3) - 1.5;
+      offset[1] = euclideanModulo(offset[1] + velocity[1] * deltaTime + 1.5, 3) - 1.5;

+      const off = ndx * (changingUnitSize / 4);
+      vertexValues.set(offset, off + kOffsetOffset);
      vertexValues.set([scale / aspect, scale], off + kScaleOffset);
    });

...

+    requestAnimationFrame(render);
  }
+  requestAnimationFrame(render);

  const observer = new ResizeObserver(entries => {
    for (const entry of entries) {
      const canvas = entry.target;
      const width = entry.contentBoxSize[0].inlineSize;
      const height = entry.contentBoxSize[0].blockSize;
      canvas.width = Math.max(1, Math.min(width, device.limits.maxTextureDimension2D));
      canvas.height = Math.max(1, Math.min(height, device.limits.maxTextureDimension2D));
-      // re-render
-      render();
    }
  });
  observer.observe(canvas);
```

我们这里还顺便切换到了一个 rAF 循环[^rAF]。

[^rAF]: `rAF` 就是 `requestAnimationFrame` 的缩写

<a id="a-euclidianModulo"></a>上面的代码使用了 `euclideanModulo`
来更新 offset。
`euclideanModulo` 返回的是欧几里得意义下的余数，
也就是始终为正的余数；
而 `%`
运算符返回的余数，
方向会和原值保持一致。
例如

<div class="webgpu_center">
  <div class="center">
    <div class="data-table center" data-table='{
  "cols": ["value", "% operator", "euclideanModulo"],
  "classNames": ["a", "b", "c"],
  "rows": [
    [ "0.3", "0.3", "0.3" ],
    [ "2.3", "0.3", "0.3" ],
    [ "4.3", "0.3", "0.3" ],
    [ "-1.7", "-1.7", "0.3" ],
    [ "-3.7", "-1.7", "0.3" ]
  ]
}'>
     </div>
  </div>
  <div>模 2 时，`%` 与 `euclideanModulo` 的区别</div>
</div>

换个角度看，
下面是 `%`
运算符和 `euclideanModulo`
的曲线图对比

<div class="webgpu_center">
  <img style="width: 700px" src="resources/euclidean-modulo.svg">
  <div>euclideanModule(v, 2)</div>
</div>
<div class="webgpu_center">
  <img  style="width: 700px" src="resources/modulo.svg">
  <div>v % 2</div>
</div>

所以，上面的代码会先把 offset
（它当前是在 clip space 中）
加上 1.5。
然后对 3 取 `euclideanModulo`，
得到一个始终包裹在 0.0 到 3.0 之间的值，
再减去 1.5。
这样最终数值就会始终保持在 -1.5 到 +1.5 之间，
并且会从一边绕回到另一边。
之所以用 -1.5 到 +1.5，
是为了让圆在完全离开屏幕后才从另一侧出现。[^offscreen]

[^offscreen]: 这只在圆半径小于 0.5 时成立，
不过为了不让示例代码膨胀得太复杂，
这里就没有加入更复杂的尺寸检查。

为了让我们有东西可以调节，
再加一个选项，
让我们能设置要绘制多少个圆。

```js
-  const kNumObjects = 100;
+  const kNumObjects = 10000;


...

  const settings = {
    numObjects: 100,
  };

  const gui = new GUI();
  gui.add(settings, 'numObjects', 0, kNumObjects, 1);

  ...

    // set the scale and offset for each object
-    objectInfos.forEach(({scale, offset, veloctiy}, ndx) => {
+    for (let ndx = 0; ndx < settings.numObjects; ++ndx) {
+      const {scale, offset, velocity} = objectInfos[ndx];

      // -1.5 to 1.5
      offset[0] = euclideanModulo(offset[0] + velocity[0] * deltaTime + 1.5, 3) - 1.5;
      offset[1] = euclideanModulo(offset[1] + velocity[1] * deltaTime + 1.5, 3) - 1.5;

      const off = ndx * (changingUnitSize / 4);
      vertexValues.set(offset, off + kOffsetOffset);
      vertexValues.set([scale / aspect, scale], off + kScaleOffset);
-    });
+    }

    // upload all offsets and scales at once
-    device.queue.writeBuffer(changingVertexBuffer, 0, vertexValues);
+    device.queue.writeBuffer(
        changingVertexBuffer, 0,
        vertexValues, 0, settings.numObjects * changingUnitSize / 4);

-    pass.draw(numVertices, kNumObjects);
+    pass.draw(numVertices, settings.numObjects);
```

这样一来，
我们就有了一个会动的场景，
并且可以通过调整圆的数量来控制工作量大小。

{{{example url="../webgpu-timing-animated.html"}}}

接下来，在这个基础上，
加上 fps
以及 JavaScript 耗时统计。

首先需要一种方式来展示这些信息，
所以我们加一个 `<pre>`
元素，
并把它定位到 canvas 上方。

```html
  <body>
    <canvas></canvas>
+    <pre id="info"></pre>
  </body>
```

```css
html, body {
  margin: 0;       /* remove the default margin          */
  height: 100%;    /* make the html,body fill the page   */
}
canvas {
  display: block;  /* make the canvas act like a block   */
  width: 100%;     /* make the canvas fill its container */
  height: 100%;
}
+#info {
+  position: absolute;
+  top: 0;
+  left: 0;
+  margin: 0;
+  padding: 0.5em;
+  background-color: rgba(0, 0, 0, 0.8);
+  color: white;
+}
```

其实我们已经有显示 fps 所需的数据了，
就是上面算出来的 `deltaTime`。

至于 JavaScript 耗时，
只需要记录
`requestAnimationFrame`
开始的时间，
以及它结束的时间即可。

```js
  let then = 0;
  function render(now) {
    now *= 0.001;  // convert to seconds
    const deltaTime = now - then;
    then = now;

+    const startTime = performance.now();

    ...

+    const jsTime = performance.now() - startTime;

+    infoElem.textContent = `\
+fps: ${(1 / deltaTime).toFixed(1)}
+js: ${jsTime.toFixed(1)}ms
+`;

    requestAnimationFrame(render);
  }
  requestAnimationFrame(render);
```

这样我们就得到了前两个时间指标。

{{{example url="../webgpu-timing-with-fps-js-time.html"}}}

## <a id="a-timestamp-query"></a>给 GPU 计时

WebGPU 提供了一个**可选**
feature，
叫做 `'timestamp-query'`，
可以用来测量某个 GPU 操作花了多长时间。
因为它是可选 feature，
所以我们得像[limits 和 features 那篇文章](webgpu-limits-and-features.html)中介绍的那样，
先检查它是否存在并显式请求它。

```js
async function main() {
  const adapter = await navigator.gpu?.requestAdapter();
-  const device = await adapter?.requestDevice();
+  const canTimestamp = adapter.features.has('timestamp-query');
+  const device = await adapter?.requestDevice({
+    requiredFeatures: [
+      ...(canTimestamp ? ['timestamp-query'] : []),
+     ],
+  });
  if (!device) {
    fail('need a browser that supports WebGPU');
    return;
  }
```

上面我们用 `canTimestamp`
记录 adapter 是否支持
`'timestamp-query'`
feature。
如果支持，
那就在创建 device 时把它请求下来。

启用这个 feature 后，
我们就可以让 WebGPU 为 render pass
或 compute pass 记录 *timestamp*。
做法是创建一个 `GPUQuerySet`，
并把它挂到 render pass
或 compute pass 上。
`GPUQuerySet` 本质上可以理解成一个查询结果数组。
你告诉 WebGPU：
这个数组中的哪个元素要记录 pass 开始时间，
哪个元素要记录 pass 结束时间。
之后再把这些 timestamp
复制到一个 buffer 里，
再把这个 buffer map 出来，
就能在 JavaScript 中读取这些结果。[^mapping-not-necessary]

[^mapping-not-necessary]: 把 query 结果复制到一个可映射 buffer，
只是为了能在 JavaScript 中读取这些值。
如果你的场景只需要结果继续保留在 GPU 上，
例如要作为后续某个过程的输入，
那其实不需要把结果复制到可映射 buffer。

所以第一步，先创建 query set。

```js
  const querySet = device.createQuerySet({
     type: 'timestamp',
     count: 2,
  });
```

`count` 至少需要是 2，
这样我们才能同时写入
开始和结束两个 timestamp。

接下来需要一个 buffer，
把 querySet 里的结果转成我们可以访问的数据。

```js
  const resolveBuffer = device.createBuffer({
    size: querySet.count * 8,
    usage: GPUBufferUsage.QUERY_RESOLVE | GPUBufferUsage.COPY_SRC,
  });
```

querySet 中的每个元素都占 8 字节。
这个 buffer
必须带有 `QUERY_RESOLVE` usage。
另外，如果我们还想在 JavaScript 中读回结果，
那它还需要 `COPY_SRC` usage，
这样才能把结果再复制到一个可映射 buffer 中。

最后，再创建一个可映射 buffer，
用来真正读取这些结果。

```js
  const resultBuffer = device.createBuffer({
    size: resolveBuffer.size,
    usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.MAP_READ,
  });
```

不过这段代码必须包起来，
只在 feature 存在时才创建这些对象，
否则一旦尝试创建 `'timestamp'`
类型的 querySet，
就会直接报错。

```js
+  const { querySet, resolveBuffer, resultBuffer } = (() => {
+    if (!canTimestamp) {
+      return {};
+    }

    const querySet = device.createQuerySet({
       type: 'timestamp',
       count: 2,
    });
    const resolveBuffer = device.createBuffer({
      size: querySet.count * 8,
      usage: GPUBufferUsage.QUERY_RESOLVE | GPUBufferUsage.COPY_SRC,
    });
    const resultBuffer = device.createBuffer({
      size: resolveBuffer.size,
      usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.MAP_READ,
    });
+    return {querySet, resolveBuffer, resultBuffer };
+  })();
```

然后在 render pass descriptor 中，
告诉它要使用哪个 querySet，
以及 querySet 中的哪个索引
分别用于记录开始和结束时间戳。

```js
  const renderPassDescriptor = {
    label: 'our basic canvas renderPass with timing',
    colorAttachments: [
      {
        // view: <- to be filled out when we render
        clearValue: [0.3, 0.3, 0.3, 1],
        loadOp: 'clear',
        storeOp: 'store',
      },
    ],
    ...(canTimestamp && {
      timestampWrites: {
        querySet,
        beginningOfPassWriteIndex: 0,
        endOfPassWriteIndex: 1,
      },
    }),
  };
```

上面如果 feature 存在，
我们就往 `renderPassDescriptor`
里加一个 `timestampWrites`
字段，
并指定 querySet，
让它把开始时间写到第 0 个元素，
结束时间写到第 1 个元素。

在 pass 结束之后，
我们还需要调用 `resolveQuerySet`。
它会把 query 的结果写进一个 buffer。
我们需要传入 querySet、
从 query set 的哪个索引开始解析、
要解析多少个条目、
写到哪个 buffer、
以及写入这个 buffer 的哪个 offset。

```js
    pass.end();

+    if (canTimestamp) {
+      encoder.resolveQuerySet(querySet, 0, querySet.count, resolveBuffer, 0);
+    }
```

我们还希望把 `resolveBuffer`
复制到 `resultsBuffer`，
这样就能把结果 map 出来并在 JavaScript 里查看。
不过这里有一个问题：
如果 `resultsBuffer`
当前已经处于 mapped 状态，
那我们就不能往它里复制。
好在 buffer 有一个 `mapState`
属性可以查看。
如果它是 `'unmapped'`
（初始状态），
那就说明现在可以安全复制。
另外两个可能值分别是：
`'pending'`，也就是我们调用 `mapAsync`
后立即进入的状态；
以及 `'mapped'`，
也就是 `mapAsync`
真正 resolve 后的状态。
调用 `unmap` 后，
它又会回到 `'unmapped'`。

```js
    if (canTimestamp) {
      encoder.resolveQuerySet(querySet, 0, 2, resolveBuffer, 0);
+      if (resultBuffer.mapState === 'unmapped') {
+        encoder.copyBufferToBuffer(resolveBuffer, 0, resultBuffer, 0, resultBuffer.size);
+      }
    }
```

在提交 command buffer 之后，
我们就可以去 map `resultBuffer` 了。
和上面一样，
也只有在它是 `'unmapped'`
时才去 map。

```js
+  let gpuTime = 0;

   ...

   function render(now) {

    ...

    const commandBuffer = encoder.finish();
    device.queue.submit([commandBuffer]);

+    if (canTimestamp && resultBuffer.mapState === 'unmapped') {
+      resultBuffer.mapAsync(GPUMapMode.READ).then(() => {
+        const times = new BigUint64Array(resultBuffer.getMappedRange());
+        gpuTime = Number(times[1] - times[0]);
+        resultBuffer.unmap();
+      });
+    }
```

query set 的结果单位是纳秒，
并且以 64 位整数形式存储。
为了在 JavaScript 中读取它们，
我们可以用一个
`BigUint64Array`
view。
不过 `BigUint64Array`
要格外小心使用：
当你从里面读出一个元素时，
得到的类型是 `bigint`，
不是普通的 `number`，
所以它不能直接参与很多常见数学运算。
另外，当你把它转换成 number 时，
还可能丢失精度，
因为 JavaScript 的 `number`
最多只能精确表示 53 位整数。
所以这里我们先让两个 `bigint`
相减，
结果仍然还是 `bigint`，
然后再把差值转换成普通 number，
以便像平常那样使用。

在上面的代码里，
我们并不是每一帧都把结果复制到 `resultBuffer`，
而是只在它当前未被映射时才复制。
这意味着我们也只会在某些帧上读取 GPU 时间。
很多时候看起来会像是“隔一帧读一次”，
但并没有严格保证，
因为 `mapAsync`
到底多久 resolve 并不确定。
因此我们把结果写进 `gpuTime`，
这样无论何时都能拿到最近一次记录到的 GPU 时间。

```js
    infoElem.textContent = `\
fps: ${(1 / deltaTime).toFixed(1)}
js: ${jsTime.toFixed(1)}ms
+gpu: ${canTimestamp ? `${(gpuTime / 1000).toFixed(1)}碌s` : 'N/A'}
`;
```

这样我们就能从 WebGPU 拿到 GPU 耗时了。

{{{example url="../webgpu-timing-with-timestamp.html"}}}

对我来说，这些数字变化得太快，
很难直接看出什么有用信息。
一种改进方式是，
计算一个滚动平均值。
下面是一个用来计算滚动平均的类。

```js
// Note: We disallow negative values as this is used for timestamp queries
// where it's possible for a query to return a beginning time greater than the
// end time. See: https://gpuweb.github.io/gpuweb/#timestamp
class NonNegativeRollingAverage {
  #total = 0;
  #samples = [];
  #cursor = 0;
  #numSamples;
  constructor(numSamples = 30) {
    this.#numSamples = numSamples;
  }
  addSample(v) {
    if (!Number.isNaN(v) && Number.isFinite(v) && v >= 0) {
      this.#total += v - (this.#samples[this.#cursor] || 0);
      this.#samples[this.#cursor] = v;
      this.#cursor = (this.#cursor + 1) % this.#numSamples;
    }
  }
  get() {
    return this.#total / this.#samples.length;
  }
}
```

它内部维护了一个值数组和一个总和。
每加入一个新值时，
就会先把最旧的那个值从总和中减掉，
再把新值加进去。

它可以这样使用。

```js
+const fpsAverage = new NonNegativeRollingAverage();
+const jsAverage = new NonNegativeRollingAverage();
+const gpuAverage = new NonNegativeRollingAverage();

function render(now) {
  ...

    const commandBuffer = encoder.finish();
    device.queue.submit([commandBuffer]);

    if (canTimestamp && resultBuffer.mapState === 'unmapped') {
      resultBuffer.mapAsync(GPUMapMode.READ).then(() => {
        const times = new BigUint64Array(resultBuffer.getMappedRange());
        gpuTime = Number(times[1] - times[0]);
+        gpuAverage.addSample(gpuTime / 1000);
        resultBuffer.unmap();
      });
    }

    const jsTime = performance.now() - startTime;

+    fpsAverage.addSample(1 / deltaTime);
+    jsAverage.addSample(jsTime);

    infoElem.textContent = `\
-fps: ${(1 / deltaTime).toFixed(1)}
-js: ${jsTime.toFixed(1)}ms
-gpu: ${canTimestamp ? `${(gpuTime / 1000).toFixed(1)}碌s` : 'N/A'}
+fps: ${fpsAverage.get().toFixed(1)}
+js: ${jsAverage.get().toFixed(1)}ms
+gpu: ${canTimestamp ? `${gpuAverage.get().toFixed(1)}碌s` : 'N/A'}
`;

    requestAnimationFrame(render);
  }
  requestAnimationFrame(render);
}
```

这样数字就会稳定一些了。

{{{example url="../webgpu-timing-with-timestamp-w-average.html"}}}

## <a id="a-timing-helper"></a>使用一个辅助类

对我来说，
上面这一整套流程其实挺繁琐的，
而且很容易一不小心写错。
我们得准备 3 个东西：
一个 querySet，
外加两个 buffer。
还要修改 renderPassDescriptor，
还要 resolve 结果，
再复制到可映射 buffer。

一种让这一切没那么繁琐的办法，
就是写一个专门帮我们做计时的类。
下面就是一个示例辅助类，
它可以在一定程度上简化这些问题。

```js
function assert(cond, msg = '') {
  if (!cond) {
    throw new Error(msg);
  }
}

// We track command buffers so we can generate an error if
// we try to read the result before the command buffer has been executed.
const s_unsubmittedCommandBuffer = new Set();

/* global GPUQueue */
GPUQueue.prototype.submit = (function(origFn) {
  return function(commandBuffers) {
    origFn.call(this, commandBuffers);
    commandBuffers.forEach(cb => s_unsubmittedCommandBuffer.delete(cb));
  };
})(GPUQueue.prototype.submit);

// See https://webgpufundamentals.org/webgpu/lessons/webgpu-timing.html
export default class TimingHelper {
  #canTimestamp;
  #device;
  #querySet;
  #resolveBuffer;
  #resultBuffer;
  #commandBuffer;
  #resultBuffers = [];
  // state can be 'free', 'need resolve', 'wait for result'
  #state = 'free';

  constructor(device) {
    this.#device = device;
    this.#canTimestamp = device.features.has('timestamp-query');
    if (this.#canTimestamp) {
      this.#querySet = device.createQuerySet({
         type: 'timestamp',
         count: 2,
      });
      this.#resolveBuffer = device.createBuffer({
        size: this.#querySet.count * 8,
        usage: GPUBufferUsage.QUERY_RESOLVE | GPUBufferUsage.COPY_SRC,
      });
    }
  }

  #beginTimestampPass(encoder, fnName, descriptor) {
    if (this.#canTimestamp) {
      assert(this.#state === 'free', 'state not free');
      this.#state = 'need resolve';

      const pass = encoder[fnName]({
        ...descriptor,
        ...{
          timestampWrites: {
            querySet: this.#querySet,
            beginningOfPassWriteIndex: 0,
            endOfPassWriteIndex: 1,
          },
        },
      });

      const resolve = () => this.#resolveTiming(encoder);
      const trackCommandBuffer = (cb) => this.#trackCommandBuffer(cb);
      pass.end = (function(origFn) {
        return function() {
          origFn.call(this);
          resolve();
        };
      })(pass.end);

      encoder.finish = (function(origFn) {
        return function() {
          const cb = origFn.call(this);
          trackCommandBuffer(cb);
          return cb;
        };
      })(encoder.finish);

      return pass;
    } else {
      return encoder[fnName](descriptor);
    }
  }

  beginRenderPass(encoder, descriptor = {}) {
    return this.#beginTimestampPass(encoder, 'beginRenderPass', descriptor);
  }

  beginComputePass(encoder, descriptor = {}) {
    return this.#beginTimestampPass(encoder, 'beginComputePass', descriptor);
  }

  #trackCommandBuffer(cb) {
    if (!this.#canTimestamp) {
      return;
    }
    assert(this.#state === 'need finish', 'you must call encoder.finish');
    this.#commandBuffer = cb;
    s_unsubmittedCommandBuffer.add(cb);
    this.#state = 'wait for result';
  }

  #resolveTiming(encoder) {
    if (!this.#canTimestamp) {
      return;
    }
    assert(
      this.#state === 'need resolve',
      'you must use timerHelper.beginComputePass or timerHelper.beginRenderPass',
    );
    this.#state = 'need finish';

    this.#resultBuffer = this.#resultBuffers.pop() || this.#device.createBuffer({
      size: this.#resolveBuffer.size,
      usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.MAP_READ,
    });

    encoder.resolveQuerySet(this.#querySet, 0, this.#querySet.count, this.#resolveBuffer, 0);
    encoder.copyBufferToBuffer(this.#resolveBuffer, 0, this.#resultBuffer, 0, this.#resultBuffer.size);
  }

  async getResult() {
    if (!this.#canTimestamp) {
      return 0;
    }
    assert(
      this.#state === 'wait for result',
      'you must call encoder.finish and submit the command buffer before you can read the result',
    );
    assert(!!this.#commandBuffer); // internal check
    assert(
      !s_unsubmittedCommandBuffer.has(this.#commandBuffer),
      'you must submit the command buffer before you can read the result',
    );
    this.#commandBuffer = undefined;
    this.#state = 'free';

    const resultBuffer = this.#resultBuffer;
    await resultBuffer.mapAsync(GPUMapMode.READ);
    const times = new BigUint64Array(resultBuffer.getMappedRange());
    const duration = Number(times[1] - times[0]);
    resultBuffer.unmap();
    this.#resultBuffers.push(resultBuffer);
    return duration;
  }
}
```

这些 `assert`
的作用，
就是帮助我们在错误使用这个类时及时报错。
例如，
如果我们结束了一个 pass
却没有 resolve 它，
或者虽然 resolve 了，
却在 command buffer 还没 submit 的时候就尝试去读结果。

有了这个类之后，
我们就可以删掉很多之前的样板代码。

```js
async function main() {
  const adapter = await navigator.gpu?.requestAdapter();
  const canTimestamp = adapter.features.has('timestamp-query');
  const device = await adapter?.requestDevice({
    requiredFeatures: [
      ...(canTimestamp ? ['timestamp-query'] : []),
     ],
  });
  if (!device) {
    fail('need a browser that supports WebGPU');
    return;
  }

+  const timingHelper = new TimingHelper(device);

  ...

-  const { querySet, resolveBuffer, resultBuffer } = (() => {
-    if (!canTimestamp) {
-      return {};
-    }
-
-    const querySet = device.createQuerySet({
-       type: 'timestamp',
-       count: 2,
-    });
-    const resolveBuffer = device.createBuffer({
-      size: querySet.count * 8,
-      usage: GPUBufferUsage.QUERY_RESOLVE | GPUBufferUsage.COPY_SRC,
-    });
-    const resultBuffer = device.createBuffer({
-      size: resolveBuffer.size,
-      usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.MAP_READ,
-    });
-    return {querySet, resolveBuffer, resultBuffer };
-  })();

  ...

  function render(now) {

    ...

-    const pass = encoder.beginRenderPass(renderPassDescriptor);
+    const pass = timingHelper.beginRenderPass(encoder, renderPassDescriptor);

    ...

    pass.end();

-    if (canTimestamp) {
-      encoder.resolveQuerySet(querySet, 0, querySet.count, resolveBuffer, 0);
-      if (resultBuffer.mapState === 'unmapped') {
-        encoder.copyBufferToBuffer(resolveBuffer, 0, resultBuffer, 0, resultBuffer.size);
-      }
-    }

    const commandBuffer = encoder.finish();
    device.queue.submit([commandBuffer]);

+    timingHelper.getResult().then(gpuTime => {
+        gpuAverage.addSample(gpuTime / 1000);
+    });

    ...
```

{{{example url="../webgpu-timing-with-timing-helper.html"}}}

关于这个 `TimingHelper`
类，还有几点说明：

* 你仍然需要在创建设备时，
  手动请求 `'timestamp-query'`
  这个 feature，
  但这个类会自己处理：
  device 上到底有没有这个 feature。

* 当你调用 `timerHelper.beginRenderPass`
  或 `timerHelper.beginComputePass`
  时，
  它会自动往 pass descriptor
  中加入相应字段。
  同时它返回的 pass encoder，
  其 `end`
  方法也会自动帮你 resolve query。

* 它的设计目标之一，
  就是如果你用错了，
  它会尽量直接报错提醒你。

* 它目前只处理 1 个 pass

  这里面其实有很多取舍，
  如果不进一步深入设计，
  很难说什么方案一定最好。

  理论上，
  一个能处理多个 pass 的类
  确实会很有用。
  但理想情况下，
  你应该使用一个足够大的
  `GPUQuerySet`
  来容纳全部 pass，
  而不是每个 pass
  单独创建一个 `GPUQuerySet`。

  可如果想这样做，
  你就得：
  要么让用户一开始就告诉你
  他们最多会有多少个 pass；
  要么把代码写得更复杂，
  先从一个较小的 `GPUQuerySet`
  开始，
  一旦 pass 数量超出当前容量，
  就删掉它再创建一个更大的。
  可那样一来，
  至少有 1 帧时间里，
  你还得额外处理多个 `GPUQuerySet`
  并存的情况。

  这一整套看起来有点过度设计了，
  所以目前这个类就只处理 1 个 pass，
  你如果真有需要，
  可以在它之上继续扩展，
  再决定是否要修改它的设计。

你也可以写一个 `NoTimingHelper`。

```js
class NoTimingHelper {
  constructor() { }
  beginRenderPass(encoder, descriptor = {}) {
    return encoder.beginTimestampPass(descriptor);
  }

  beginComputePass(encoder, descriptor = {}) {
    return encoder.beginComputePass(descriptor);
  }
  async getResult() { return 0; }
}
```

这是一种可能的做法：
让你在增加计时功能、
以及把计时关掉时，
都不需要改动太多代码。

总之，我已经把 `TimingHelper`
类用在了
[使用 compute shader 计算图像直方图](webgpu-compute-shaders-histogram.html)
那几篇文章的示例上。
下面是它们的列表。
其中只有视频示例是持续运行的，
所以大概是最值得看的一个。

* <a target="_blank" href="../webgpu-compute-shaders-histogram-video-w-timing.html">4 通道视频直方图</a>

其他示例都只运行一次，
并把结果打印到 JavaScript 控制台中。

* <a target="_blank" href="../webgpu-compute-shaders-histogram-4ch-optimized-more-w-timing.html">4 通道：每个 chunk 一个 workgroup 的直方图 + reduce</a>
* <a target="_blank" href="../webgpu-compute-shaders-histogram-4ch-race-fixed-w-timing.html">4 通道：每像素一个 workgroup 的直方图</a>
* <a target="_blank" href="../webgpu-compute-shaders-histogram-4ch-javascript-w-timing.html">4 通道 JavaScript 直方图</a>
* <a target="_blank" href="../webgpu-compute-shaders-histogram-optimized-more-w-timing.html">1 通道：每个 chunk 一个 workgroup 的直方图 + reduce</a>
* <a target="_blank" href="../webgpu-compute-shaders-histogram-optimized-w-timing.html">1 通道：每个 chunk 一个 workgroup 的直方图 + sum</a>
* <a target="_blank" href="../webgpu-compute-shaders-histogram-race-fixed-w-timing.html">1 通道：每像素一个 workgroup 的直方图</a>
* <a target="_blank" href="../webgpu-compute-shaders-histogram-slow-w-timing.html">1 通道：单核直方图</a>
* <a target="_blank" href="../webgpu-compute-shaders-histogram-javascript-w-timing.html">1 通道 JavaScript 直方图</a>

# <a id="a-implementation-defined"></a>重要：`timestamp-query` 的结果是实现定义的

这实际上意味着，
你可以用它做调试，
也可以用它来比较不同技术方案，
但你不能指望它对所有用户都返回相近的结果。
甚至连“相对关系”都不能完全假设。
不同 GPU 的工作方式不同，
它们也可能会跨 pass
做不同的渲染和计算优化。
这意味着，
在一台机器上，
第一个 pass 画 100 个东西可能花 200碌s，
第二个 pass 画另外 200 个东西也可能是 200碌s；
但换到另一张 GPU，
画前 100 个东西可能只花 100碌s，
画后 100 个东西却花 200碌s。
也就是说，
第一张 GPU 里两次之间的相对差值是 0碌s，
而第二张 GPU 里的相对差值却是 100碌s，
尽管它们被要求绘制的是同样的内容。

# <a id="a-implementation-defined"></a>重要：`timestamp-query` 并不是衡量性能的好指标

Timestamp query 并不是衡量性能的理想指标，
因为整体性能还受到许多其他因素的影响。
举个具体例子。
我们在[把图像加载进纹理那篇文章](webgpu-importing-textures.html#a-generating-mips-on-the-gpu)
里写过一个基于 render pass 的 mipmap 生成器。
我后来也写了一个基于 compute pass 的 mipmap 生成器。
当我用 timestamp-query
分别测它们时，
结果告诉我：
compute pass 的方法快 5 倍。
太棒了！
但后来我换成了一个吞吐量测试。
我不再使用 timestamp-query，
而是写了一个测试，
在 60fps 下不断增加需要生成 mipmap 的 2048x2048 纹理数量。
我会一直增加数量，
直到帧率跌破 60fps。
结果这个测试显示，
在一台机器上，
render pass 方案比 compute pass 方案快 20%，
而在另一台机器上，
也快了 8%。

重点在于：
你不能孤立地只看 timestamp-query，
就断言某个方案运行得更快。

<div class="webgpu_bottombar">默认情况下，<code>'timestamp-query'</code> 的时间值
会被量化到 100碌 秒。
在 Chrome 中，
如果你在 <a href="chrome://flags/#enable-webgpu-developer-features" target="_blank">about:flags</a>
里启用 <a href="chrome://flags/#enable-webgpu-developer-features" target="_blank">"enable-webgpu-developer-features"</a>，
时间值有可能不会被量化。
理论上这会给你更精确的计时结果。
不过通常来说，
100碌 秒量化后的值，
已经足够用来比较不同 shader 技术之间的性能差异了。
</div>
