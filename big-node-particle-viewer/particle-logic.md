# 模型粒子转换逻辑

本文档记录独立查看器使用的模型转粒子效果实现方式。它提取自原来的大节点粒子算法，但现在不绑定任何具体项目节点，可以用于任意 GLB / GLTF 模型预览。

## 总流程

```text
GLB 模型
  -> 遍历 mesh 顶点
  -> 按预算计算 STEP 采样
  -> 提取第一张可用贴图
  -> 将贴图渲染到 512 x 512 RenderTarget
  -> 按 UV 读取像素颜色
  -> 写入 position / normal / phase / color 属性
  -> 居中并统一缩放到最大尺寸 15
  -> 设置 uYBounds
  -> 使用 ShaderMaterial 渲染 THREE.Points
```

## 采样预算

转换逻辑按总顶点数计算采样步长：

```js
const STEP = Math.max(1, Math.floor(totalVerts / budget));
```

独立查看器默认预算是 `1200000`，适合更稳定地查看普通模型。需要更细时可以在面板里调高，最高到 `3000000`。

## 贴图颜色采样

当前逻辑只取模型中第一张找到的 `material.map`：

```text
first material.map -> 512 RenderTarget -> readRenderTargetPixels -> UV sample
```

这意味着它不是逐材质精确采样。优点是简单、性能稳定；缺点是多材质模型可能无法完全还原每个材质的原贴图色彩。

## 点云属性

每个采样点写入这些属性：

- `position`：mesh 顶点经过 `matrixWorld` 后的世界坐标，最后统一居中缩放。
- `aNormal`：法线经过 normal matrix 转换，用于 shader 光影。
- `aPhase`：随机相位，用于轻微呼吸漂浮。
- `aColor`：从贴图 UV 采样得到的颜色，或者无贴图时使用暖灰 fallback 色。

## 统一缩放

转换后会计算点云包围盒，并把最大边缩放到 `15`：

```js
const scale = 15 / Math.max(size.x, size.y, size.z);
```

这一步让不同 GLB 的尺度接近，便于自动取景和使用统一 shader。

## Shader 效果

查看器 shader 主要由这些部分组成：

- 呼吸扰动：用 `uTime + aPhase` 给每个点做很轻的 x/y/z 位移。
- 三方向光：key light、fill light、rim light 结合 `aNormal` 计算层次。
- 原贴图墨化：`aColor` 与灰度色按 `uInkMix` 混合，形成克制的水墨感。
- 高光和边缘光：让模型轮廓和迎光面更有星光雕塑感。
- 底部淡入：用 `uYBounds` 计算高度归一化，再做 `smoothstep`，减少底部硬边。
- 透视粒径：点大小随相机距离变化，并受高光、边缘光影响。
- 深度淡出：远处点通过 `uDepthNear / uDepthFar` 逐渐变淡。

## 可调参数

- `uSize`：基础粒径。
- `uOpacity`：整体透明度。
- `uInkMix`：贴图色彩转水墨灰度的比例。
- `uPtMax`：点大小最大上限。
- `uPtPersp`：透视粒径强度。
- `uYBounds`：当前点云自身最低点和最高点，用于底部淡入。

## 后续可拓展方向

- 改成逐 mesh / 逐 material 贴图采样，让多材质模型颜色更准。
- 给每个模型保存独立粒径、透明度、墨化参数。
- 加入 hover 热点，让鼠标靠近模型局部时局部变亮。
- 加入性能档位，比如低配电脑自动降低预算和像素比。
