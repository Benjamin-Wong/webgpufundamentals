Title: WebGPU 工作原理
Description: WebGPU 是如何工作的
TOC: 工作原理

让我们尝试用 JavaScript 来模拟 GPU 使用顶点着色器和片元着色器所做的事情，
从而解释 WebGPU 的工作原理。希望这能帮助你对底层发生的事情建立起直觉感受。

如果你熟悉
[Array.map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map)，
眯起眼睛仔细想想，就能大致理解这两种着色器函数是怎么工作的。
使用 `Array.map` 时，你提供一个函数来对某个值进行变换。

示例：

```js
const shader = v => v * 2;  // double the input
const input = [1, 2, 3, 4];
const output = input.map(shader);   // result [2, 4, 6, 8]
```

上面这个用于 `Array.map` 的"着色器"，只是一个接收一个数值后返回其双倍的函数。
这大概是 JavaScript 中最接近"着色器"含义的类比了。
它是一个返回或生成值的函数，你不直接调用它，
而是先指定它，然后由系统替你调用。

对于 GPU 顶点着色器来说，你不是对一个输入数组做映射，
而是只指定你想让函数被调用多少次。

```js
function draw(count, vertexShaderFn) {
  const internalBuffer = [];
  for (let i = 0; i < count; ++i) {
    internalBuffer[i] = vertexShaderFn(i);
  }
  console.log(JSON.stringify(internalBuffer));
}
```

这样做的一个结果是，与 `Array.map` 不同，我们不再需要一个源数组来完成操作。

```js
const shader = v => v * 2;
const count = 4;
draw(count, shader);
// outputs [0, 2, 4, 6]
```

让 GPU 工作变得复杂的是，这些函数运行在你电脑中一个独立的系统上——GPU。
这意味着你创建和引用的所有数据，都必须以某种方式传送给 GPU，
然后你还需要告诉着色器你把数据放在哪里以及如何访问它。

顶点着色器和片元着色器可以通过 6 种方式获取数据：Uniforms、Attributes、Buffers、Textures、跨阶段变量（Inter-Stage Variables）、Constants。

1. Uniforms

   Uniforms 是在着色器每次迭代中都保持相同的值。
   可以把它们理解为常量全局变量。
   你可以在着色器运行之前设置它们，
   但在着色器执行期间，它们保持不变，
   也就是说它们保持*一致（uniform）*。

   让我们修改 `draw` 函数，把 uniforms 传递给着色器。
   为此，我们将创建一个名为 `bindings` 的数组，并用它传入 uniforms。

   ```js
   *function draw(count, vertexShaderFn, bindings) {
     const internalBuffer = [];
     for (let i = 0; i < count; ++i) {
   *    internalBuffer[i] = vertexShaderFn(i, bindings);
     }
     console.log(JSON.stringify(internalBuffer));
   }
   ```

   然后修改着色器来使用 uniforms

   ```js
   const vertexShader = (v, bindings) => {
     const uniforms = bindings[0];
     return v * uniforms.multiplier;
   };
   const count = 4;
   const uniforms1 = {multiplier: 3};
   const uniforms2 = {multiplier: 5};
   const bindings1 = [uniforms1];
   const bindings2 = [uniforms2];
   draw(count, vertexShader, bindings1);
   // outputs [0, 3, 6, 9]
   draw(count, vertexShader, bindings2);
   // outputs [0, 5, 10, 15]
   ```

   希望 uniforms 的概念现在看起来已经很直观了。
   通过 `bindings` 间接访问的方式，是因为这与 WebGPU 中的实际做法"类似"。
   如前所述，我们通过位置/索引来访问数据，在这里是 `bindings[0]`。

2. Attributes（仅顶点着色器）

   Attributes 为着色器的每次迭代提供不同的数据。
   在上面的 `Array.map` 中，值 `v` 是从 `input` 中取出并自动传给函数的，
   这与着色器中的 attribute 非常相似。

   区别在于，我们并不是在对 input 做映射，
   我们只是在计数，因此我们需要告诉 WebGPU
   这些输入是什么，以及如何从中取出数据。

   想象一下我们像这样更新 `draw`：

   ```js
   *function draw(count, vertexShaderFn, bindings, attribsSpec) {
     const internalBuffer = [];
     for (let i = 0; i < count; ++i) {
   *    const attribs = getAttribs(attribsSpec, i);
   *    internalBuffer[i] = vertexShaderFn(i, bindings, attribs);
     }
     console.log(JSON.stringify(internalBuffer));
   }

   +function getAttribs(attribs, ndx) {
   +  return attribs.map(({source, offset, stride}) => source[ndx * stride + offset]);
   +}
   ```

   然后我们可以这样调用它：

   ```js
   const buffer1 = [0, 1, 2, 3, 4, 5, 6, 7];
   const buffer2 = [11, 22, 33, 44];
   const attribsSpec = [
     { source: buffer1, offset: 0, stride: 2, },
     { source: buffer1, offset: 1, stride: 2, },
     { source: buffer2, offset: 0, stride: 1, },
   ];
   const vertexShader = (v, bindings, attribs) => (attribs[0] + attribs[1]) * attribs[2];
   const bindings = [];
   const count = 4;
   draw(count, vertexShader, bindings, attribsSpec);
   // outputs [11, 110, 297, 572]
   ```

   如你所见，`getAttribs` 使用 `offset` 和 `stride`
   来计算对应 `source` 缓冲区中的索引并取出值，
   然后将取出的值传给着色器。每次迭代中 `attribs` 都会不同。

   ```
    iteration |  attribs
    ----------+-------------
        0     | [0, 1, 11]
        1     | [2, 3, 22]
        2     | [4, 5, 33]
        3     | [6, 7, 44]
   ```

3. Raw Buffers

   Buffers 本质上就是数组，在我们的类比中，
   让我们创建一个使用 buffers 的 `draw` 版本。
   我们将通过 `bindings` 传递这些缓冲区，就像处理 uniforms 一样。

   ```js
   const buffer1 = [0, 1, 2, 3, 4, 5, 6, 7];
   const buffer2 = [11, 22, 33, 44];
   const attribsSpec = [];
   const bindings = [
     buffer1,
     buffer2,
   ];
   const vertexShader = (ndx, bindings, attribs) => 
       (bindings[0][ndx * 2] + bindings[0][ndx * 2 + 1]) * bindings[1][ndx];
   const count = 4;
   draw(count, vertexShader, bindings, attribsSpec);
   // outputs [11, 110, 297, 572]
   ```

   这里我们得到了与 attributes 相同的结果，
   区别是这次系统不再替我们从缓冲区中取出值，
   而是由我们自己计算索引来访问绑定的缓冲区。
   这比 attributes 更灵活，因为我们可以对数组进行随机访问。
   但出于同样的原因，它可能也更慢。
   由于 attributes 的访问是有规律的，GPU 知道数据会被顺序访问，
   可以据此进行优化。比如顺序访问通常对缓存友好。
   而当我们自己计算索引时，GPU 在我们实际访问之前并不知道
   我们要访问缓冲区的哪个部分。

4. Textures

   Textures 是一维、二维或三维的数据数组。
   当然，我们可以用缓冲区来实现自己的二维或三维数组。
   纹理的特别之处在于它们可以被*采样（sampled）*。
   采样意味着我们可以让 GPU 计算我们提供的值之间的插值。
   具体含义我们会在[纹理那篇文章](webgpu-textures.html)中介绍。
   现在，让我们再做一个 JavaScript 类比。

   首先创建一个函数 `textureSample`，用于在数组的值之间进行采样插值。

   ```js
   function textureSample(texture, ndx) {
     const startNdx = ndx | 0;  // round down to an int
     const fraction = ndx % 1;  // get the fractional part between indices
     const start = texture[startNdx];
     const end = texture[startNdx + 1];
     return start + (end - start) * fraction;  // compute value between start and end
   }
   ```

   GPU 上已经内置了类似这样的函数。

   现在在着色器中使用它：

   ```js
   const texture = [10, 20, 30, 40, 50, 60, 70, 80];
   const attribsSpec = [];
   const bindings = [
     texture,
   ];
   const vertexShader = (ndx, bindings, attribs) =>
       textureSample(bindings[0], ndx * 1.75);
   const count = 4;
   draw(count, vertexShader, bindings, attribsSpec);
   // outputs [10, 27.5, 45, 62.5]
   ```

   当 `ndx` 为 `3` 时，我们会向 `textureSample` 传入 `3 * 1.75` 也就是 `5.25`。
   它会计算出 `startNdx` 为 `5`，取出索引 `5` 和 `6` 处的值，
   分别是 `60` 和 `70`。`fraction` 为 `0.25`，
   所以结果是 `60 + (70 - 60) * 0.25`，即 `62.5`。

   看上面的代码，我们完全可以在自己的着色器函数中手动实现 `textureSample`，
   手动取出两个值然后做插值。
   GPU 提供这个特殊功能的原因是它可以做得更快，
   而且根据设置，它可能会读取多达 16 个 4 分量浮点值
   来产生 1 个 4 分量浮点值。如果手动实现，工作量会非常大。

5. 跨阶段变量 Inter-Stage Variables（仅片元着色器）

   跨阶段变量是从顶点着色器传递给片元着色器的输出。
   如前所述，顶点着色器输出的位置被用来绘制/光栅化点、线和三角形。

   假设我们正在画一条线。假设顶点着色器被调用了两次，
   第一次输出相当于 `5,0` 的位置，第二次输出相当于 `25,4` 的位置。
   根据这两个点，GPU 会从 `5,0` 到 `25,4`（不含端点）画一条线，
   为此会调用我们的片元着色器 20 次，每个像素调用一次。
   每次调用时，由我们决定返回什么颜色。

   假设我们有一对辅助函数来在两点之间画线。
   第一个函数计算需要绘制多少个像素以及一些辅助值，
   第二个函数根据这些信息和像素编号给出像素位置。示例：

   ```js
   const line = calcLine([10, 10], [13, 13]);
   for (let i = 0; i < line.numPixels; ++i) {
     const p = calcLinePoint(line, i);
     console.log(p);
   }
   // prints
   // 10,10
   // 11,11
   // 12,12
   ```

   注意：`calcLine` 和 `calcLinePoint` 的具体实现并不重要，
   重要的是它们能工作，让上面的循环给出一条线上的像素位置。
   **如果你对它们的实现感到好奇，可以查看文章底部的在线代码示例。**

   现在，让我们修改顶点着色器，让它每次迭代输出 2 个值。
   有很多方式可以做到，这里展示其中一种：

   ```js
   const buffer1 = [5, 0, 25, 4];
   const attribsSpec = [
     {source: buffer1, offset: 0, stride: 2},
     {source: buffer1, offset: 1, stride: 2},
   ];
   const bindings = [];
   const dest = new Array(2);
   const vertexShader = (ndx, bindings, attribs) => [attribs[0], attribs[1]];
   const count = 2;
   draw(count, vertexShader, bindings, attribsSpec);
   // outputs [[5, 0], [25, 4]]
   ```

   现在写一些代码，每次取两个点并调用 `rasterizeLines` 来光栅化一条线。

   ```js
   function rasterizeLines(dest, destWidth, inputs, fragShaderFn, bindings) {
     for (let ndx = 0; ndx < inputs.length - 1; ndx += 2) {
       const p0 = inputs[ndx    ];
       const p1 = inputs[ndx + 1];
       const line = calcLine(p0, p1);
       for (let i = 0; i < line.numPixels; ++i) {
         const p = calcLinePoint(line, i);
         const offset = p[1] * destWidth + p[0];  // y * width + x
         dest[offset] = fragShaderFn(bindings);
       }
     }
   }
   ```

   我们可以这样更新 `draw` 来使用这段代码

   ```js
   -function draw(count, vertexShaderFn, bindings, attribsSpec) {
   +function draw(dest, destWidth,
   +              count, vertexShaderFn, fragmentShaderFn,
   +              bindings, attribsSpec,
   +) {
     const internalBuffer = [];
     for (let i = 0; i < count; ++i) {
       const attribs = getAttribs(attribsSpec, i);
       internalBuffer[i] = vertexShaderFn(i, bindings, attribs);
     }
   -  console.log(JSON.stringify(internalBuffer));
   +  rasterizeLines(dest, destWidth, internalBuffer,
   +                 fragmentShaderFn, bindings);
   }
   ```

   现在我们真正用上 `internalBuffer` 了 😃！

   来更新调用 `draw` 的代码：

   ```js
   const buffer1 = [5, 0, 25, 4];
   const attribsSpec = [
     {source: buffer1, offset: 0, stride: 2},
     {source: buffer1, offset: 1, stride: 2},
   ];
   const bindings = [];
   const vertexShader = (ndx, bindings, attribs) => [attribs[0], attribs[1]];
   const count = 2;
   -draw(count, vertexShader, bindings, attribsSpec);

   +const width = 30;
   +const height = 5;
   +const pixels = new Array(width * height).fill(0);
   +const fragShader = (bindings) => 6;

   *draw(
   *   pixels, width,
   *   count, vertexShader, fragShader,
   *   bindings, attribsSpec);
   ```

   如果把 `pixels` 以矩形方式打印，其中 `0` 显示为 `.`，结果如下

   ```
   .....666......................
   ........66666.................
   .............66666............
   ..................66666.......
   .......................66.....
   ```

   不幸的是，我们的片元着色器在每次迭代中没有任何变化的输入，
   所以无法为每个像素输出不同的值。这就是跨阶段变量的用武之地。
   让我们修改第一个着色器，让它输出一个额外的值。

   ```js
   const buffer1 = [5, 0, 25, 4];
   +const buffer2 = [9, 3];
   const attribsSpec = [
     {source: buffer1, offset: 0, stride: 2},
     {source: buffer1, offset: 1, stride: 2},
   +  {source: buffer2, offset: 0, stride: 1},
   ];
   const bindings = [];
   const dest = new Array(2);
   const vertexShader = (ndx, bindings, attribs) => 
   -    [attribs[0], attribs[1]];
   +    [[attribs[0], attribs[1]], [attribs[2]]];

   ...
   ```

   如果什么都不改，`draw` 内部循环结束后，`internalBuffer` 的值将是

   ```js
    [ 
      [[ 5, 0], [9]],
      [[25, 4], [3]],
    ]
   ```

   我们可以很容易地计算一个从 0.0 到 1.0 的值，表示当前在线段上的位置，
   用它来对我们刚添加的那个额外值进行插值。

   ```js
   function rasterizeLines(dest, destWidth, inputs, fragShaderFn, bindings) {
     for(let ndx = 0; ndx < inputs.length - 1; ndx += 2) {
   -    const p0 = inputs[ndx    ];
   -    const p1 = inputs[ndx + 1];
   +    const p0 = inputs[ndx    ][0];
   +    const p1 = inputs[ndx + 1][0];
   +    const v0 = inputs[ndx    ].slice(1);  // everything but the first value
   +    const v1 = inputs[ndx + 1].slice(1);
       const line = calcLine(p0, p1);
       for (let i = 0; i < line.numPixels; ++i) {
         const p = calcLinePoint(line, i);
   +      const t = i / line.numPixels;
   +      const interStageVariables = interpolateArrays(v0, v1, t);
         const offset = p[1] * destWidth + p[0];  // y * width + x
   -      dest[offset] = fragShaderFn(bindings);
   +      dest[offset] = fragShaderFn(bindings, interStageVariables);
       }
     }
   }

   +// interpolateArrays([[1,2]], [[3,4]], 0.25) => [[1.5, 2.5]]
   +function interpolateArrays(v0, v1, t) {
   +  return v0.map((array0, ndx) => {
   +    const array1 = v1[ndx];
   +    return interpolateValues(array0, array1, t);
   +  });
   +}

   +// interpolateValues([1,2], [3,4], 0.25) => [1.5, 2.5]
   +function interpolateValues(array0, array1, t) {
   +  return array0.map((a, ndx) => {
   +    const b = array1[ndx];
   +    return a + (b - a) * t;
   +  });
   +}
   ```

   现在我们可以在片元着色器中使用这些跨阶段变量了

   ```js
   -const fragShader = (bindings) => 6;
   +const fragShader = (bindings, interStageVariables) => 
   +    interStageVariables[0] | 0; // convert to int
   ```

   运行后结果如下

   ```
   .....988......................
   ........87776.................
   .............66655............
   ..................54443.......
   .......................33.....
   ```

   顶点着色器第一次迭代输出 `[[5,0], [9]]`，
   第二次迭代输出 `[[25,4], [3]]`，
   你可以看到，在片元着色器被调用的过程中，
   每个输出的第二个值在这两个值之间进行了插值。

   我们还可以再写一个函数 `mapTriangle`，
   给定 3 个点来光栅化一个三角形，并对三角形内部每个点调用片元着色器。
   它会从 3 个点而不是 2 个点来插值跨阶段变量。

以下是上面所有示例的在线运行版本，你可以动手试试，加深理解。

{{{example url="../webgpu-javascript-analogies.html"}}}

上面 JavaScript 中发生的事情是一种类比。
跨阶段变量实际上如何插值、线如何绘制、
缓冲区如何访问、纹理如何采样、uniforms 和 attributes 如何指定，
在 WebGPU 中的细节都有所不同。
但概念非常相似，希望这个 JavaScript 类比能帮助你建立对内部运作的心智模型。

为什么要设计成这样？如果你看 `draw` 和 `rasterizeLines`，
你会发现每次迭代都完全独立于其他迭代。
换句话说，你可以按任意顺序处理每次迭代。
不一定是 0、1、2、3、4 这样的顺序，
你可以按 3、1、4、0、2 的顺序处理，结果完全一样。
迭代之间相互独立这一事实，
意味着每次迭代都可以由不同的处理器并行执行。
2021 年的现代顶级 GPU 拥有 10000 个甚至更多的处理器，
这意味着最多可以有 10000 件事同时并行进行。
这就是使用 GPU 的强大之处——通过遵循这些模式，
系统可以将工作大规模并行化。

最大的限制是：

1. 着色器函数只能引用其输入
   （attributes、buffers、textures、uniforms、跨阶段变量）。

2. 着色器不能分配内存。

3. 着色器在引用它写入的对象时必须格外小心，
   也就是它正在生成值的那个对象。

   仔细想想这很合理。想象一下，如果上面的 `fragShader`
   试图直接引用 `dest`。这意味着在尝试并行化时将无法协调。
   哪次迭代先执行？如果第 3 次迭代引用了 `dest[0]`，
   那么第 0 次迭代就必须先运行；
   但如果第 0 次迭代引用了 `dest[3]`，
   那么第 3 次迭代又必须先运行。

   这种限制在 CPU 的多线程或多进程中同样存在，
   但在 GPU 领域，同时运行多达 10000 个处理器时，
   需要特殊的协调机制。我们会在其他文章中介绍一些相关技术。
