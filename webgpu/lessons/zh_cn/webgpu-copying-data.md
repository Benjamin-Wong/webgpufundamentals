Title: WebGPU 复制数据
Description: 在 buffer 与 texture 之间复制数据
TOC: 复制数据

到目前为止的大多数文章里，
我们主要使用的是 `writeBuffer`
把数据写进 buffer，
以及 `writeTexture`
把数据写进 texture。
但实际上，把数据放进 buffer 或 texture
还有好几种方式。

## `writeBuffer`

`writeBuffer` 会把 JavaScript 中
`TypedArray` 或 `ArrayBuffer`
里的数据复制到一个 buffer 中。
可以说，这是把数据放进 buffer
最直接的一种方式。

`writeBuffer` 的格式如下

```js
device.queue.writeBuffer(
  destBuffer,  // the buffer to write to
  destOffset,  // where in the destination buffer to start writing
  srcData,     // a typedArray or arrayBuffer
  srcOffset?,  // offset in **elements** in srcData to start copying
  size?,       // size in **elements** of srcData to copy
)
```

如果没有传 `srcOffset`，
它默认就是 `0`。
如果没有传 `size`，
那默认就是 `srcData` 的大小。

> 重要：`srcOffset` 和 `size` 的单位，
> 是 `srcData` 的“元素数”，而不是字节数。
>
> 换句话说，
>
> ```js
> device.queue.writeBuffer(
>   someBuffer,
>   someOffset,
>   someFloat32Array,
>   6,
>   7,
> )
> ``` 
>
> 上面的代码，
> 会从第 6 个 float32 开始，
> 复制 7 个 float32 的数据。
> 也可以换个说法：
> 它会从 `someFloat32Array`
> 所引用的那段 arrayBuffer 中，
> 从第 24 个字节开始，
> 复制 28 个字节的数据。

## `writeTexture`

`writeTexture` 会把 JavaScript 中
`TypedArray` 或 `ArrayBuffer`
里的数据复制到一张纹理中。
  
`writeTexture` 的签名如下

```js
device.queue.writeTexture(
  // details of the destination
  { texture, mipLevel: 0, origin: [0, 0, 0], aspect: "all" },

  // the source data
  srcData,

  // details of the source data
  { offset: 0, bytesPerRow, rowsPerImage },

  // size:
  [ width, height, depthOrArrayLayers ] or { width, height, depthOrArrayLayers }
)
```

需要注意几点：

* `texture` 必须带有 `GPUTextureUsage.COPY_DST` usage

* `mipLevel`、`origin` 和 `aspect` 都有默认值，
  所以很多时候不需要显式指定

* `bytesPerRow`：表示从一行 *block row* 数据，
  前进到下一行 *block row* 数据需要跨过多少字节。

   如果你复制的数据超过 1 个 *block row*，
   这个值就是必须的。
   而大多数情况下，你复制的数据确实都会超过 1 个 *block row*，
   所以它几乎总是必需的。

* `rowsPerImage`：表示从一张图像的起始位置，
  前进到下一张图像的起始位置，需要跨过多少个 *block row*。

   如果你复制的数据超过 1 层，
   这个值就是必需的。
   换句话说，如果 size 参数里的 `depthOrArrayLayers > 1`，
   你就需要提供这个值。

你可以把整个复制过程理解成这样

```js
   // pseudo code
   const [x, y, z] = origin ?? [0, 0, 0];
   const [blockWidth, blockHeight, bytesPerBlock] =
      getBlockInfoForTextureFormat(texture.format);

   const blocksAcross = width / blockWidth;
   const blocksDown = height / blockHeight;
   const bytesPerBlockRow = blocksAcross * bytesPerBlock;

   for (layer = 0; layer < depthOrArrayLayers; layer) {
      for (row = 0; row < blocksDown; ++row) {
        const start = offset + (layer * rowsPerImage + row) * bytesPerRow;
        copyRowToTexture(
            texture,               // texture to copy to
            x, y + row, z + layer, // where in texture to copy to
            srcDataAsBytes + start,
            bytesPerBlockRow);
      }
   }
```

### <a id="a-block-rows"></a>**block row**

纹理是按 block 组织的。
对于大多数 *普通* 纹理，
block 的宽和高都等于 1。
但对压缩纹理来说就不是这样了。
例如 `bc1-rgba-unorm`
这个格式，
它的 block 宽度是 4，高度也是 4。
这意味着，如果你设置 width 为 8，
height 为 12，
那最终只会复制 6 个 block：
第一行 2 个 block，
第二行 2 个，
第三行 2 个。

对于压缩纹理，
size 和 origin
都必须对齐到 block 的尺寸。

> 重要：在 WebGPU 中，
> 任何需要 size（也就是 `GPUExtent3D`）的地方，
> 都可以是一个包含 1 到 3 个数字的数组，
> 也可以是一个包含 1 到 3 个属性的对象。
> 其中 `height` 和 `depthOrArrayLayers`
> 默认值都是 1，所以
>
> * `[2]` 表示 width = 2, height = 1, depthOrArrayLayers = 1
> * `[2, 3]` 表示 width = 2, height = 3, depthOrArrayLayers = 1
> * `[2, 3, 4]` 表示 width = 2, height = 3, depthOrArrayLayers = 4
> * `{ width: 2 }` 表示 width = 2, height = 1, depthOrArrayLayers = 1
> * `{ width: 2, height: 3 }` 表示 width = 2, height = 3, depthOrArrayLayers = 1
> * `{ width: 2, height: 3, depthOrArrayLayers: 4 }` 表示 width = 2, height = 3, depthOrArrayLayers = 4

> 同样地，任何出现 origin（也就是 `GPUOrigin3D`）的地方，
> 你也可以传一个包含 3 个数字的数组，
> 或者一个带有 `x`、`y`、`z` 属性的对象。
> 它们默认都为 0，所以
>
> * `[5]` 表示 x = 5, y = 0, z = 0
> * `[5, 6]` 表示 x = 5, y = 6, z = 0
> * `[5, 6, 7]` 表示 x = 5, y = 6, z = 7
> * `{ x: 5 }` 表示 x = 5, y = 0, z = 0
> * `{ x: 5, y: 6 }` 表示 x = 5, y = 6, z = 0
> * `{ x: 5, y: 6, z: 7 }` 表示 x = 5, y = 6, z = 7

* `aspect` 主要只会在向 depth-stencil 格式复制数据时起作用。
  一次只能复制一个 aspect，
  也就是只能选 `depth-only`
  或 `stencil-only` 其中之一。

> 冷知识：texture 上本身有 `width`、`height`
> 和 `depthOrArrayLayer` 属性，
> 所以它本身就是一个合法的 `GPUExtent3D`。
> 换句话说，假设我们有这样一张纹理
>
> ```js
> const texture = device.createTexture({
>   format: 'r8unorm',
>   size: [2, 4],
>   usage: GPUTextureUsage.COPY_DST | GPUTextureUsage.TEXTURE_ATTACHMENT,
> });
> ```
>
> 那么下面这些写法都能工作
>
> ```js
> // copy 2x4 pixels of data to texture
> const bytesPerRow = 2;
> device.queue.writeTexture({ texture }, data, { bytesPerRow }, [2, 4]);
> device.queue.writeTexture({ texture }, data, { bytesPerRow }, [texture.width, texture.height]);
> device.queue.writeTexture({ texture }, data, { bytesPerRow }, {width: 2, height: 4});
> device.queue.writeTexture({ texture }, data, { bytesPerRow }, {width: texture.width, height: texture.height});
> device.queue.writeTexture({ texture }, data, { bytesPerRow }, texture); // !!!
> ```
>
> 最后一种之所以也成立，
> 是因为 texture 自身就有 `width`、`height`
> 和 `depthOrArrayLayers`。
> 我们之前一直没有用这种写法，
> 是因为它不够直观，
> 但从语法上讲它是合法的。

## `copyBufferToBuffer`

顾名思义，`copyBufferToBuffer`
是把数据从一个 buffer
复制到另一个 buffer。

签名如下：

```js
encoder.copyBufferToBuffer(
  source,       // buffer to copy from
  sourceOffset, // where to start copying from
  dest,         // buffer to copy to
  destOffset,   // where to start copying to
  size,         // how many bytes to copy
)
```

* `source` 必须带有 `GPUBufferUsage.COPY_SRC`
* `dest` 必须带有 `GPUBufferUsage.COPY_DST`
* `size` 必须是 4 的倍数

## `copyBufferToTexture`

顾名思义，`copyBufferToTexture`
是把数据从 buffer 复制到 texture。

签名如下：

```js
encoder.copyBufferToTexture(
  // details of the source buffer
  { buffer, offset: 0, bytesPerRow, rowsPerImage },

  // details of the destination texture
  { texture, mipLevel: 0, origin: [0, 0, 0], aspect: "all" },

  // size:
  [ width, height, depthOrArrayLayers ] or { width, height, depthOrArrayLayers }
)
```

它和 `writeTexture`
几乎拥有完全相同的参数。
最大的不同在于：
`bytesPerRow` **必须是 256 的倍数！！**

* `texture` 必须带有 `GPUTextureUsage.COPY_DST`
* `buffer` 必须带有 `GPUBufferUsage.COPY_SRC`

## `copyTextureToBuffer`

顾名思义，`copyTextureToBuffer`
是把数据从 texture 复制到 buffer。

签名如下：

```js
encoder.copyTextureToBuffer(
  // details of the source texture
  { texture, mipLevel: 0, origin: [0, 0, 0], aspect: "all" },

  // details of the destination buffer
  { buffer, offset: 0, bytesPerRow, rowsPerImage },

  // size:
  [ width, height, depthOrArrayLayers ] or { width, height, depthOrArrayLayers }
)
```

它和 `copyBufferToTexture`
参数很相似，
只是 texture（现在是源）
和 buffer（现在是目标）
对调了。
和 `copyBufferToTexture` 一样，
`bytesPerRow` **也必须是 256 的倍数！！**

* `texture` 必须带有 `GPUTextureUsage.COPY_SRC`
* `buffer` 必须带有 `GPUBufferUsage.COPY_DST`

## `copyTextureToTexture`

`copyTextureToTexture`
会把一张纹理中的某一部分，
复制到另一张纹理中。

这两张纹理必须要么格式完全相同，
要么唯一的区别仅仅是是否带有 `'-srgb'`
这个后缀。

签名如下：

```js
encoder.copyTextureToTexture(
  // details of the source texture
  src: { texture, mipLevel: 0, origin: [0, 0, 0], aspect: "all" },

  // details of the destination texture
  dst: { texture, mipLevel: 0, origin: [0, 0, 0], aspect: "all" },

  // size:
  [ width, height, depthOrArrayLayers ] or { width, height, depthOrArrayLayers }
)
```

* src.`texture` 必须带有 `GPUTextureUsage.COPY_SRC`
* dst.`texture` 必须带有 `GPUTextureUsage.COPY_DST`
* `width` 必须是 block width 的倍数
* `height` 必须是 block height 的倍数
* src.`origin[0]` 或 `.x` 必须是 block width 的倍数
* src.`origin[1]` 或 `.y` 必须是 block height 的倍数
* dst.`origin[0]` 或 `.x` 必须是 block width 的倍数
* dst.`origin[1]` 或 `.y` 必须是 block height 的倍数

## Shader

Shader 可以读写 storage buffer、
storage texture，
也可以间接向 texture 渲染。
这些其实都是把数据放进 buffer
和 texture 的方式。
换句话说，你也可以写 shader
来生成数据、复制数据、传递数据。

## 映射 Buffer

你可以对 buffer 进行 map。
把一个 buffer map 出来，
意思就是让它在 JavaScript 中变得可读或可写。
不过至少在 WebGPU v1 中，
可映射 buffer 有很严格的限制：
一个可映射 buffer
只能被用作“临时中转区”，
也就是只允许用来复制数据的来源或目标。
可映射 buffer
不能再被当成其他类型的 buffer 来使用
（例如 uniform buffer、vertex buffer、
index buffer、storage buffer 等等）[^mappedAtCreation]

[^mappedAtCreation]: 例外情况是你设置了 `mappedAtCreation: true`。
见 [mappedAtCreation](#a-mapped-at-creation)。

要创建一个可映射 buffer，
只能使用下面两种 usage 组合。

* `GPUBufferUsage.MAP_READ | GPUBufferUsage.COPY_DST`

  这种 buffer 可以先通过上面的复制命令，
  从另一个 buffer 或 texture
  把数据拷进来，
  然后再 map 出来，
  在 JavaScript 中读取其内容。

* `GPUBufferUsage.MAP_WRITE | GPUBufferUsage.COPY_SRC`

  这种 buffer 可以先在 JavaScript 中 map 出来，
  然后从 JavaScript 往里面写数据，
  最后 unmap，
  再通过上面的复制命令，
  把它的内容复制到其他 buffer 或 texture 中。

映射 buffer 的过程是异步的。
你需要调用
`buffer.mapAsync(mode, offset = 0, size?)`，
其中 `offset`
和 `size`
的单位都是字节。
如果没指定 `size`，
那默认就是整个 buffer 的大小。
`mode` 必须是
`GPUMapMode.READ`
或 `GPUMapMode.WRITE` 之一，
并且当然必须与你创建 buffer 时所传的
`MAP_`
usage 标志相匹配。

`mapAsync` 会返回一个 `Promise`。
当这个 promise resolve 之后，
buffer 才真正变成可映射状态。
接下来你就可以通过
`buffer.getMappedRange(offset = 0, size?)`
来获取它的全部或部分内容，
其中 `offset`
是相对于你 map 出来的那段 buffer 区间的字节偏移。
`getMappedRange` 返回的是一个 `ArrayBuffer`，
所以通常为了方便使用，
你会再基于它创建一个 TypedArray。

下面是一个映射 buffer 的例子

```js
const buffer = device.createBuffer({
  size: 1024,
  usage: GPUBufferUsage.MAP_READ | GPUBufferUsage.COPY_DST,
});

// map the entire buffer
await buffer.mapAsync(GPUMapMode.READ);

// get the entire buffer as an array of 32bit floats.
const f32 = new Float32Array(buffer.getMappedRange())

...

buffer.unmap();
```

注意：一旦 buffer 处于 mapped 状态，
它在调用 `unmap` 之前就不能再被 WebGPU 使用。
而且 `unmap` 一调用，
这个 buffer 在 JavaScript 侧就会立刻“消失”。
换句话说，还是以上面的例子为例

```js
const f32 = new Float32Array(buffer.getMappedRange())

f32[0] = 123;
console.log(f32[0]); // prints 123

buffer.unmap();

console.log(f32[0]); // prints undefined
```

我们其实已经在[第一篇文章](webgpu-fundamentals.html#a-run-computations-on-the-gpu)里见过
“把 buffer map 出来用于读取”的例子了：
当时我们在 storage buffer 中把一些数字翻倍，
然后把结果复制到一个可映射 buffer，
再 map 出来读取结果。

另一个例子是在[compute shader 基础](webgpu-compute-shaders.md)那篇文章里，
我们把各种 compute shader 的 `@builtin`
值输出到 storage buffer 中。
之后再把这些结果复制到一个可映射 buffer，
然后 map 出来查看结果。

## <a id="a-mapped-at-creation"></a>mappedAtCreation

`mappedAtCreation: true`
是一个可以在创建 buffer 时附加的标志。
在这种情况下，
这个 buffer 并不需要带上
`GPUBufferUsage.COPY_DST`
或 `GPUBufferUsage.MAP_WRITE`
这两个 usage 标志。

这是一个专门用于“在创建时就把数据放进 buffer”
的特殊标志。
你在创建 buffer 时，
加上 `mappedAtCreation: true`
即可。
buffer 会在创建完成时，
自动处于可写的 mapped 状态。
例如：

```js
 const buffer = device.createBuffer({
   size: 16,
   usage: GPUBufferUsage.UNIFORM,
   mappedAtCreation: true,
 });
 const arrayBuffer = buffer.getMappedRange(0, buffer.size);
 const f32 = new Float32Array(arrayBuffer);
 f32.set([1, 2, 3, 4]);
 buffer.unmap();
```

或者写得更简洁一点

```js
 const buffer = device.createBuffer({
   size: 16,
   usage: GPUBufferUsage.UNIFORM,
   mappedAtCreation: true,
 });
 new Float32Array(buffer.getMappedRange(0, buffer.size)).set([1, 2, 3, 4]);
 buffer.unmap();
```

注意，带有 `mappedAtCreation: true`
创建出来的 buffer，
并不会自动附带任何额外 flag。
它只是一个“在首次创建时方便你往里写数据”的快捷方式。
它在创建时就是 mapped 的，
但只要你 unmap 一次之后，
它就会像普通 buffer 一样工作，
只遵守你当初指定的那些 usage。
换句话说，
如果你之后还想继续往里 copy，
那就仍然需要 `GPUBufferUsage.COPY_DST`；
如果你之后还想再 map，
那就仍然需要 `GPUBufferUsage.MAP_READ`
或 `GPUBufferUsage.MAP_WRITE`。

## <a id="a-efficient"></a>高效地使用可映射 buffer

前面已经看到，map buffer 是异步的。
这意味着，从我们调用 `mapAsync`
请求把 buffer map 出来开始，
到它真正可以调用 `getMappedRange`
之间，中间会有一段不可确定的等待时间。

一个常见的解决思路是：
维护一组始终保持 mapped 状态的 buffer。
由于它们本来就已经处于 mapped 状态，
所以可以立刻使用。
你一旦用掉其中一个并调用了 `unmap`，
同时在提交完使用这个 buffer 的命令之后，
就立刻再次请求把它 map 回来。
等它的 promise resolve 后，
再把它放回“已映射 buffer 池”中。
如果某一时刻你需要一个 mapped buffer
但池里一个都没有，
那就新建一个，
并把它也加入池中。
