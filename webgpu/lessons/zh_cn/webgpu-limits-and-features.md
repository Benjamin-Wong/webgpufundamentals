Title: WebGPU 可选特性与限制
Description: 可选特性
TOC: 可选特性与限制

WebGPU 提供了不少可选特性（optional features）和限制（limits）。
下面我们来看看该如何检查它们，以及如何请求它们。

当你像下面这样请求一个 adapter 时

```js
const adapter = await navigator.gpu?.requestAdapter();
```

这个 adapter 会在 `adapter.limits`
上提供一组限制值，
并在 `adapter.features`
上提供一组特性名称。比如

```js
const adapter = await navigator.gpu?.requestAdapter();
console.log(adapter.limits.maxColorAttachments);
```

控制台里可能会打印出 `8`，
表示这个 adapter 最多支持
8 个 color attachment。

下面是所有 limit 的列表，
包括你当前默认 adapter 的 limit，
以及规范要求的最低 limit。

<div class="webgpu_center data-table limits" data-diagram="limits"></div>

所谓最低 limit，
就是所有支持 WebGPU 的设备
都应该至少具备的那些 limit。

此外，还有一组可选 feature。
例如，你可以这样查看它们

```js
const adapter = await navigator.gpu?.requestAdapter();
console.log(adapter.features);
```

它可能会打印出类似
`["texture-compression-astc", "texture-compression-bc"]`
这样的内容，
表示这些 feature 在你请求时是可用的。

下面是你当前默认 adapter 上可用的 feature 列表。

<div class="webgpu_center data-table features" data-diagram="features"></div>

> 注意：你可以在 [webgpureport.org](https://webgpureport.org)
> 查看你系统中所有 adapter 的 feature 和 limit。

## 请求 limits 和 features

默认情况下，当你请求一个 device 时，
你拿到的是最低限度的 limit
（也就是上面右侧那一列），
并且不会自动启用任何可选 feature。
这样设计的目的是：
如果你的程序始终控制在最低 limit 范围内，
那么它就可以运行在所有支持 WebGPU 的设备上。

不过，基于 adapter 上列出的可用 limits 和 features，
你也可以在调用 `requestDevice` 时，
通过 `requiredLimits`
和 `requiredFeatures`
把自己真正需要的能力请求下来。
例如

```js
const k1Gig = 1024 * 1024 * 1024;
const adapter = await navigator.gpu?.requestAdapter();
const device = adapter?.requestDevice({
  requiredLimits: { maxBufferSize: k1Gig },
  requiredFeatures: [ 'float32-filterable' ],
});
```

上面这段代码中，我们请求：
允许使用最大 1GB 的 buffer，
以及允许使用可过滤的 float32 纹理
（例如 `'rgba32float'`
配合 `minFilter: 'linear'`。
默认情况下，这种格式通常只能配合 `'nearest'` 使用）

如果这些请求中有任何一个无法满足，
那么 `requestDevice`
就会失败（也就是 promise 被 reject）。

## 不要把所有东西都请求下来

你可能会觉得，
干脆把所有 limit 和 feature
全都请求下来，
然后再在代码里检查自己实际用了哪些，
似乎很省事。

例如：

```js
function objLikeToObj(src) {
  const dst = {};
  for (const key in src) {
    dst[key] = src[key];
  }
  return dst;
}

//
// BAD!!! ?
//
async function main() {
  const adapter = await navigator?.gpu.requestAdapter();
  const device = await adapter?.requestDevice({
    requiredLimits: objLikeToObj(adapter.limits),
    requiredFeatures: adapter.features,
  });
  if (!device) {
    fail('need webgpu');
    return;
  }

  const canUse128KUniformsBuffers = device.limits.maxUniformBufferBindingSize >= 128 * 1024;
  const canStoreToBGRA8Unorm = device.features.has('bgra8unorm-storage');
  const canIndirectFirstInstance = device.features.has('indirect-first-instance');
}
```

这看起来似乎是一种简单又清晰的检查方式[^objliketoobj]。
问题在于，这种模式很容易让你在不知不觉中依赖了超出最低要求的能力。
例如，假设你创建了一张 `'rgba32float'` 纹理，
并且还对它用了 `'linear'` 过滤。
在你的桌面机器上，它可能“神奇地”就能正常工作，
只是因为你碰巧把这个 feature 也一起启用了。

[^objliketoobj]: 这个 `objLikeToObj` 是什么？为什么需要它？
这是一个比较偏门的 Web 规范细节。
规范中把 `requiredLimits`
定义成 `record<DOMString, GPUSize64>`。
而 Web IDL 规范规定，
当把一个对象转换成 `record<DOMString, GPUSize64>` 时，
只会拷贝这个对象“自身拥有”的属性。
但是 adapter 上的 `limits`
对象本身在规范里是一个 `interface`。
那些看起来像属性的东西，
其实并不是对象自身属性，
而是定义在原型链上的 getter。
也就是说，它们不是对象的 own properties。
因此在转换成 `record<DOMString, GPUSize64>` 时，
它们不会被自动拷贝。
所以你必须自己把它们复制出来。

可是一旦用户换到手机上运行你的程序，
就会莫名其妙失败，
因为 `'float32-filterable'`
这个 feature 在那台设备上根本不存在，
而你当时又没有意识到自己已经依赖了一个可选 feature。

或者你也可能在不知不觉中，
分配了一个大于最低 `maxBufferSize`
限制的 buffer，
自己却完全没意识到已经超限了。
结果程序发布之后，
很多用户都打不开你的页面。

## 推荐的 features 和 limits 请求方式

推荐做法是：
先明确你“绝对必须”要用到哪些 feature 和 limit，
然后只请求那些真正必需的内容。

例如

```js
  const adapter = await navigator?.gpu.requestAdapter();

  const canUse128KUniformsBuffers = adapter?.limits.maxUniformBufferBindingSize >= 128 * 1024;
  const canStoreToBGRA8Unorm = adapter?.features.has('bgra8unorm-storage');
  const canIndirectFirstInstance = adapter?.features.has('indirect-first-instance');

  // if we absolutely need these one or more of these features then fail now if they are not
  // available
  if (!canUse128kUniformBuffers) {
    alert('Sorry, your device is probably too old or underpowered');
    return;
  }

  // Request the available features and limits we need
  const device = adapter?.requestDevice({
    requiredFeatures: [
      ...(canStorageBGRA8Unorm ? ['bgra8unorm'] : []),
      ...(canIndirectFirstInstance) ? ['indirect-first-instance']),
    ],
    requiredLimits: [
      maxUniformBufferBindingSize: 128 * 1024,
    ]
  });
```

这样做的话，
如果你后来不小心去请求一个超过 128KB 的 uniform buffer，
就会得到错误提示。
同样，如果你不小心使用了一个你根本没请求过的 feature，
也会得到错误。

这样你就能有意识地做决定：
到底要不要提高所需的 limit
（这也意味着程序将无法在更多设备上运行），
还是继续保持现有限制；
或者你也可以重构代码，
让它在 feature 或 limit
可用与不可用时走不同分支。

<!-- keep this at the bottom of the article -->
<link rel="stylesheet" href="webgpu-limits-and-features.css">
<script type="module" src="webgpu-limits-and-features.js"></script>
