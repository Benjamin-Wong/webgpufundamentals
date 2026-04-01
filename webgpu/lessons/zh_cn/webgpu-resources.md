Title: WebGPU 资源
Description: 各类 WebGPU 资源
TOC: 资源 / 参考资料

这里整理了一些其他的 WebGPU 资源。

# 文章

* [你的第一个 WebGPU 应用](https://codelabs.developers.google.com/your-first-webgpu-app#0)
* [WGSL 导览](https://google.github.io/tour-of-wgsl/)
* [WebGPU Unleashed：实用教程](https://shi-yan.github.io/webgpuunleashed/)
* [高效渲染 glTF 模型](https://toji.github.io/webgpu-gltf-case-study/)
* [WebGPU 最佳实践](https://toji.dev/webgpu-best-practices/)
* [Rosetta](https://toji.github.io/rosetta/) - 对比多种着色器语言中的 shader 写法
* [3D 图形入门教程与项目](https://shrekshao.github.io/3d-graphics-beginner-projects/)
* [WebGPU Render Bundle 最佳实践](https://toji.dev/webgpu-best-practices/render-bundles)

# 示例

* [WebGPU Samples](https://webgpu.github.io/webgpu-samples/)
* [WebGPU 延迟渲染](https://github.com/toji/burrow)
* [WebGPU 阴影演练场](https://toji.github.io/webgpu-shadow-playground/)
* [WebGPU Metaballs](https://toji.github.io/webgpu-metaballs/)
* [WebGPU 中的 Shadertoy 示例](https://jsgist.org/?src=a17b03b88c86c08ac621298dae50e30b)
* [Shader Graph WGSL](https://deepkolos.github.io/shader-graph-wgsl/)
* [Compute Toys](https://compute.toys)

# 调试 / 性能分析

* [使用 PIX 分析 WebGPU（仅限 Windows）](https://toji.dev/webgpu-profiling/pix)

# 工具

* [WebGPUReport.org](https://webgpureport.org) - 显示你的 WebGPU 功能支持情况

---

# <a id="a-libraries"></a>库

## 3D 库

* [three.js](https://threejs.org) [示例](https://threejs.org/examples/?q=webgpu)
* [babylon.js](https://www.babylonjs.com/)

## 实用工具

* [webgpu-utils](https://github.com/greggman/webgpu-utils) - 帮助设置 uniform buffer 数据等内容
* [wgsl-struct-buffer](https://github.com/deepkolos/wgsl-struct-buffer) - 在 TypeScript 中以带类型检查与推断的方式查看 WGSL 结构体缓冲区
* [TypeGPU](https://typegpu.com) - 以类型安全、声明式的方式管理 WebGPU 资源

## 调试 / 性能分析库

* [webgpu_inspector](https://github.com/brendan-duncan/webgpu_inspector) - 检查 WebGPU 资源和命令
* [webgpu-dev-extension](https://github.com/greggman/webgpu-dev-extension) - 作为扩展提供多种调试和开发辅助工具
* [webgpu-helpers](https://github.com/greggman/webgpu-helpers) - 提供多种调试和开发辅助工具
* [webgpu-memory](https://github.com/greggman/webgpu-memory) - 告诉你当前使用了多少内存
* [webgpu-avoid-redundant-state-setting](https://github.com/greggman/webgpu-avoid-redundant-state-setting) - 检查代码中的冗余 WebGPU 调用
* [webgpu_recorder](https://github.com/brendan-duncan/webgpu_recorder) - 将 WebGPU 录制为一个 `.HTML` 文件，便于制作可独立复现 bug/问题的仓库

## 游戏开发

[Web Game Dev](https://www.webgamedev.com/)
