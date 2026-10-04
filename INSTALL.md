# native-glass（原生玻璃）· 安装 / 排障 / 原理

给 DSH 桌面版做的一层皮肤：**官方外观 + 毛玻璃输入框 + 通透底图**。
纯声明式 —— 只有 CSS 和图片，**没有 `hooks.mjs`、不执行任何代码**。

> 版本 **1.9.2**。本文件记录这套皮肤踩过的几个大坑的完整因果链：
> §六（Windows 外壳挡死底图）、§七（皮肤管线的选择器 / token 规则 —— "改了没反应"的最大来源）、
> §十三（毛玻璃失效的两层原因）。**改皮肤前请先读这三处。**

---

## 一、包内容

| 路径 | 说明 |
|---|---|
| `native-glass/` | 皮肤本体（**目录名 = `skin.json` 里的 `id`**，别改名） |
| `linxin666-dsh-client-ui-skin-center-0.4.4.tgz` | 皮肤中心插件本体（离线 / 锁版本安装用） |
| `INSTALL.md` | 本文件 |

## 二、依赖

- DSH 桌面版（本皮肤在 **0.2.0-rc.2** 上验证：macOS 与 Windows 11 各一台）
- 皮肤中心插件 `@linxin666/dsh-client-ui-skin-center`，版本 **0.4.4**

## 三、三步安装

### 1) 装插件（三选一）

```sh
# 有网、锁版本
dsh plugin --profile desktop add @linxin666/dsh-client-ui-skin-center@0.4.4
# 离线 / 从本包安装
dsh plugin --profile desktop add ./linxin666-dsh-client-ui-skin-center-0.4.4.tgz
```

或者用 GUI：**设置 → 插件**（不保证能锁版本）。

`dsh` CLI 位置：macOS `"/Applications/DeepSeek Harness.app/Contents/Resources/runtime/cli/bin/dsh"`；
Windows 在安装目录 `resources\runtime\cli\bin\`。
⚠️ **profile 名必须是那台机器 GUI 实际在用的**（本机是 `desktop`）。

### 2) 放皮肤

| 系统 | 目标路径 |
|---|---|
| macOS | `~/.dsh/skins/native-glass/` |
| Windows | `%USERPROFILE%\.dsh\skins\native-glass\` |

**目录名必须正好是 `native-glass`**（= `skin.json` 的 `id`）。若设过
`DSH_HOME` / `DSH_SKINS_HOME` / `DSH_SKINS_DIR`，以那些为准。
放好后**不用重启**：打开 设置 → 皮肤中心，重开卡片即收录。

### 3) 应用

设置 → 皮肤中心 → 「原生玻璃」→ **应用**。

---

## 四、推荐滑杆值（这套参数是调出来的）

| 滑杆 | 值 | 作用 |
|---|---|---|
| 背景遮蔽 | **0** | **联动旋钮**：画布 + **左侧栏** + 输入卡厚薄的共同系数 |
| 背景模糊（空对话 / 有内容） | **20 / 20** | 把底图的细颗粒糊成柔光 |
| 输入卡模糊 | **14** | 驱动"真磨砂层"（糊的是从输入框下面滚过去的正文） |
| 气泡不透明度 | **0** | 0 = 消息没有底色（纯文字）；60~80 = 一层淡磨砂底 |
| 气泡模糊 | **0** | ⚠️ **自 1.8.1 起此滑杆不再生效**（见 §十一 changelog）：气泡的 `backdrop-filter` 层已被移除 |

- 底图浓淡公式：`alpha = 遮蔽 × 0.50 + 0.22`，画布与侧栏共用 → 遮蔽 0 = 最透，100 = 最实。
- **玻璃材质有两档**（1.9.0 起分开，都是"遮蔽"滑杆的线性函数）：

  | 用在哪 | 公式 | 遮蔽=0（当前设置） | 遮蔽=100 |
  |---|---|---|---|
  | 输入卡 / 提问卡 | `遮蔽 × 0.28 + 0.50` | **0.50** | 0.78 |
  | 浮层（`/` 命令面板、配件弹层、悬浮卡） | `遮蔽 × 0.20 + 0.72` | **0.72** | 0.92 |

  浮层天生压在正文上，所以地板更高。想单独调浮层厚薄，改 `skin.css` 里两行
  `--dsh-glass-fill-popover`（亮/暗各一行）；`.20` 是灵敏度、`.72` 是地板。
- 想换底图：替换 `native-glass/assets/frost-dark.jpg`（深色）/ `frost-light.jpg`（浅色）。
- `assets/backdrop-*.jpg` 是早期**无颗粒版**底图，当前未被引用（备用）。

---

## 五、像素级验收（怎么确认底图真的在）

肉眼看"有没有背景"在深色下很不可靠（差 2 个色阶也可能是"被遮住"）。用这个方法：

```sh
python3 - <<'PY'
from PIL import Image; import numpy as np
im = Image.open("截图.png").convert("RGB"); a = np.asarray(im).astype(np.float32)
lum = lambda x: 0.2126*x[...,0] + 0.7152*x[...,1] + 0.0722*x[...,2]
best = None
for y in range(int(a.shape[0]*.2), int(a.shape[0]*.8), 20):
    band = a[y:y+80, int(a.shape[1]*.28):int(a.shape[1]*.95)]; lb = lum(band)
    tf = float((lb > 55).mean())                      # 文字占比，挑最干净的一条
    if best is None or tf < best[0]: best = (tf, band, lb)
tf, band, lb = best
print("std=%.3f  唯一色值=%d  B-R=%+.2f" % (
    lb.std(), len(np.unique(band.reshape(-1,3), axis=0)),
    band[...,2].mean() - band[...,0].mean()))
PY
```

**判据（三个都要满足）**：

| 指标 | 健康 | 被遮住 |
|---|---|---|
| 局部 `std` | **> 0**（典型 2.9，Windows 修好后可达 8） | **0.000** |
| 唯一色值数 | 几十~几百 | **1** |
| `B−R`（底图是偏蓝的） | **正**（+2.6 ~ +8.6 都有） | **+1.0** 左右（官方中性色） |

⚠️ 别拿 `std` 跨窗口比 —— 两台机器窗口宽高比不同时，`object-fit: cover` 裁到的底图区域不同，
`std` 不可比，**`B−R` 才是稳的判据**。

---

## 六、Windows 专属坑：**外壳把底图挡死**（已修，但你要知道原因）

症状：皮肤应用成功、输入卡毛玻璃正常，**唯独画布和侧栏是纯色**。

**四层因果链（全部实测）**：

1. **底图没问题**：DOM 里 `<img>` 存在、`complete=true`、`naturalWidth 2560×1440`、铺满视口、`object-fit: cover`
2. **画布层是半透明的**：`centerCol` = `rgba(21,21,23,0.22)` ← 正是皮肤的 `--dsw-alias-bg-base`
3. **但下面压着一层不透明外壳**：`frame` = `rgb(27,27,28)`
4. **这层壳的颜色来自皮肤自己锁的 token**：前端有一条**平台特化**规则

```css
/* 基础规则：两个平台都有 */
._6Qf49G          { background: var(--dsw-alias-bg-base); … }
/* 平台特化覆盖：只在这个属性存在时匹配 */
[data-windows-titlebar] ._6Qf49G_frame        { background: var(--dsw-specific-sidebar-fill); … }
[data-windows-titlebar] ._6Qf49G_frame:before { background: var(--dsw-specific-sidebar-fill); … }  /* 标题栏拖拽条 */
```

数值验证：`0.22×(21,21,23) + 0.78×(27,27,28) = (25.7,25.7,26.9)`，与实测 `(25.25,25.25,26.25)` 吻合；
把外壳临时改 `transparent` 后画布 `B−R` 从 **+0.58 → +2.59**（底图蓝偏回来了）。

**修法**（`patches.css` 第 6 节）：让外壳透明 + **侧栏 token 跟随底图**（等于"这个 token 不再是硬不透明"，同类碰撞从此只会变淡、不会挡死）：

```css
[class*="_frame"]         { background-color: transparent !important; }
[class*="_frame"]::before { background-color: transparent !important; }  /* 只清色，保留 -webkit-app-region:drag */
```

> 这是**防御纵深**：即使将来前端又用这个 token 画别的全幅表面，也只会是一层薄纱，不会再"整个挡死"。
> 已做全量审计：官方共 **36 处**消费本皮肤锁过的 token，其中**平台特化的只有 `frame` 与 `frame:before` 这两处**，现已全部覆盖。

---

## 七、皮肤管线的选择器规则（"改了没反应"的根源）

皮肤管线会**自动重写选择器**，规则如下（实测）：

| 源码里写的 | 服务端实际吐出 |
|---|---|
| `:root { … }` | `html[data-dsh-skin="native-glass"] { … }` ← **`:root` 被替换成作用域本身** |
| `body { … }` | `html[data-dsh-skin="native-glass"] body { … }` |
| `body[data-ds-dark-theme] { … }` | `html[data-dsh-skin="native-glass"] body[data-ds-dark-theme] { … }` |
| `[class*="_frame"] { … }` | `html[data-dsh-skin="native-glass"] [class*="_frame"] { … }` |
| `html[data-dsh-skin="x"] [class*="_frame"]` | `html[data-dsh-skin="x"] html[data-dsh-skin="x"] [class*="_frame"]` ← **两个 html 嵌套 → 永久失配、静默失效** |

**三条铁律**：

1. **源码里永远不要自己写 `html[data-dsh-skin=…]` 前缀**（管线的插入方式是 **descendant**，写重就废）
2. 想覆盖 token 用 `:root` / `body` 是安全的（`:root` 会被替换成作用域本身）
3. **匹配不了"挂在 `<html>` 自身上的属性"** —— 例如 `[data-windows-titlebar] [class*="_frame"]` 会被拼成
   `html[data-dsh-skin=…] [data-windows-titlebar] …`，即"要求 html 的**后代**带这个属性"，
   而前端是把该属性设在 `document.documentElement` 上（`document.documentElement.hasAttribute("data-windows-titlebar")`）
   → 同样静默失效。**这类规则只能裸写、靠类名后缀命中。**
4. **`--dsw-alias-*` / `--dsw-specific-*` 的 `:root` 声明会被"搬到 body"**，其它名字不会（1.9.0 实测确认）：
   管线把匹配这两个前缀的 token 在作用域元素上重置为 `initial`，再以
   `html[data-dsh-skin="…"] body { … }` 的形式克隆到 **body**（顺便**剥掉 `!important`**）。
   于是：
   - 想覆盖 `--dsw-alias-*` / `--dsw-specific-*`：写 `:root` 或 `body` 都行，会被送到 body，**有效**；
   - 想覆盖**其它**名字（如 `--dsw-menu-surface-fill`、`--dsw-hovercard-bg`）：写 `:root` 只会落在
     `<html>` 上，而官方是在 **body** 上定义同名 token 的 ⇒ 元素自身声明胜过继承 ⇒ **等于没写**。
     **必须写进 body 级选择器**。
   - 想给 token 加 `!important` 压过官方，**不能写在 `:root`**（会被剥掉）；写在
     `body[data-ds-dark-theme]` / `body:not([data-ds-dark-theme])` 这类块里可原样保留。
   - 官方 macOS 还有一条平台兜底会盖住皮肤：`html[data-platform="darwin"] body{--dsw-specific-menu:#303136f0}`
     （特异性 (0,1,2)，且官方主题是运行时 append 的 `<style>`、排在皮肤的 `<link>` 之后 ⇒ 同分时官方赢）。
     破法：用 `body:not([data-ds-dark-theme])` / `body[data-ds-dark-theme]`，加作用域前缀后是 **(0,2,2)**。

---

## 八、排障清单（按顺序）

1. **目录名是否正好 `native-glass`**

   ```powershell
   dir "$env:USERPROFILE\.dsh\skins"          # Windows
   ```
   ```sh
   ls ~/.dsh/skins                            # macOS
   ```

   不是 → 改名（`Rename-Item … native-glass` / `mv … native-glass`），重开卡片或刷新。
   **机制**：样式表按**目录名**加载、`backgroundMedia` 的图片按 **manifest 的 `id`** 拼 URL → 目录名不对时
   出现"样式生效但背景图 404"。

2. **图片在不在**：`native-glass/assets/` 下要有 `frost-dark.jpg`（约 830 KB）与 `frost-light.jpg`（约 196 KB）

3. **资产 URL 直测**（应 200）：
   `GET {DSH地址}/api/skin-center/v2/skins/native-glass/assets/frost-dark.jpg`
   → 200 但界面无变化 = 前端缓存，**强刷**（Ctrl+F5 / Cmd+Shift+R）
   → 404 `unknown-skin-resource` = 目录名仍不对

4. **「背景遮蔽」拉到 100%** 时底图几乎被全遮 —— 看起来就像"没背景"。按 §四 的值设回 **0**。

5. **浅色 / 深色用的是两张不同图** —— 只有一种主题没背景 = 缺对应的那一张。

6. **Windows 上画布/侧栏仍是纯色** → 回到 §六（外壳 token 冲突），并确认 `patches.css` 里那两条
   `[class*="_frame"]` 规则在服务端下发中确实存在：

   ```sh
   curl -s {DSH地址}/api/skin-center/v2/skins/native-glass/patches | grep _frame
   ```

7. **用 §五 的像素验收**给出结论，别靠肉眼。

---

## 九、退出方式

设置 → 皮肤中心 → 「官方默认」→ 应用。或直接删掉 `skins/native-glass/`。
删除皮肤不会影响 DSH 本身（皮肤只覆盖它自己声明的文件）。

---

## 十、已知取舍

- 气泡「不透明度」仍有效；**「气泡模糊」自 1.8.1 起不再生效**（该层会创建背景根，hover 时会令界面抖动，已移除）。
- 皮肤**不使用任何 `:hover` 规则**，也**不使用 `mask`** —— 这是刻意的性能约束，改皮肤时请遵守。
- 消息正文的磨砂**只在有消息内容的会话里生效**（空首页不生效，这是插件设计的 gate）。
- 「背景遮蔽」是**联动**旋钮，动它同时影响画布、侧栏与输入卡厚薄 —— 想要只改一个面就得改皮肤 CSS。
- 皮肤走的是插件的 **L3（`patches.css` 自由选择器）**层；插件文档明确声明 L3 不构成安全边界。
  本皮肤是纯声明式、无 `hooks.mjs`，但**从市场随手装的皮肤若有 `hooks.mjs` 属可执行代码**，装前值得看一眼。


---

## 十一、Changelog

> 最新在最上面。**1.8.6 → 1.9.2 这一段全是"浮层毛玻璃"主题**，完整因果见 §十三。

### 1.9.2（2026-10-05）
- 按用户要求把浮层再调透一档：`--dsh-glass-fill-popover` 系数 `.18+.78` → **`.20+.72`**（遮蔽 0 时 0.72）。

### 1.9.1（2026-10-05）
- 撤掉 1.9.0 的临时诊断探针（已确认皮肤 CSS 确实命中 `[data-menu-material]`）。
- 浮层系数 `.12+.86` → `.18+.78`（用户反馈 0.86 仍偏实）。

### 1.9.0（2026-10-05）
- **浮层与输入卡拆成两档玻璃材质**：新增 `--dsh-glass-fill-popover`（`.12+.86`）供浮层使用；
  输入卡 / 提问卡维持 `--dsh-glass-fill`（`.28+.50`）。理由：浮层压在正文上，地板天然该更高。
- 加临时诊断探针 `[data-menu-material]{outline:3px solid #d946ef}`（1.9.1 删）。

### 1.8.9（2026-10-05）
- **磨砂层从 `[data-composer-card]` 本体搬到它的 `::before`**（真凶修复，见 §十三 第二层）。
  本体同时改 `background: none`（填充跟着搬，避免叠两层）并显式补 `position: relative; isolation: isolate`
  （原来靠 `backdrop-filter` 当包含块；现在面板定位行为不变）。
  附带收益：卡片不再是 `position: fixed` 后代的包含块，官方 issue #1724 那类 tooltip 抖动前提消失。

### 1.8.8（2026-10-05）
- 统一材质那批 token 全部加 `!important`（写在非 `:root` 块里才不会被管线剥掉）。
- 新增 `patches.css` 第 10 节：按稳定属性锚点把 MenuSurface 材质层
  `[data-menu-material] > [aria-hidden="true"]:first-child` 按统一玻璃材质**再画一遍**。

### 1.8.7（2026-10-05）
- **统一浮层材质**：在 `body:not([data-ds-dark-theme])` / `body[data-ds-dark-theme]` 块里把
  `--dsw-menu-surface-fill` / `--dsw-specific-menu` / `--dsw-hovercard-bg` 全部指向同一份玻璃填充，
  糊度跟随「输入卡模糊」滑杆（`blur(var(--dsh-input-card-blur,20px)) saturate(1.06)`）。
  必须写 body 级的原因见 §七 铁律 4（要压过官方 macOS 的 `--dsw-specific-menu:#303136f0` 兜底）。
- 提问卡 / 计划评审卡接入同一套材质（`patches.css` 第 9 节）。

### 1.8.6（2026-10-04）
- 中和皮肤中心插件给输入区配件强加的 `backdrop-filter`：数据面把 `--dsh-composer-accessory-blur` 定为 `0px`，
  选择器面用 `[data-phase][data-phase]` 双写属性提高特异性（第 8 节）。

### 1.8.5（2026-10-04）
- **修复「输入卡右侧多出一块模糊斑」+「hover 上下文用量指示器闪烁」的真凶** ——
  **根因不在本皮肤**，而在 `@linxin666/dsh-client-ui-skin-center@0.4.4`（`lib/client.js:4643-4663`）注入的规则：
  它给输入区配件（`[data-slot="conversation.composer.dock"] > * / + *` 等）强加
  `background(不透明) + backdrop-filter: blur(var(--dsh-composer-accessory-blur, var(--dsh-input-card-blur, 10px)))`。
  ⇒ 那条 blur 的**回退值正是本皮肤的「输入卡模糊」滑杆**，所以用户把滑杆拖到 0 时它才消失（用户实测确认）。
  三重问题：① 不透明底下的 blur 视觉零收益；② 命中元素既带后代又是 hover 目标 ⇒ 背景根反复重算（闪烁）；
  ③ 违反本皮肤性能约束。
- 本版新增第 8 节，用更高特异性把那条 blur 中和为 `none`（6 条选择器：带 `[data-dsh-backdrop-active]` 的 3 条，
  以及**不带它**的 3 条 —— 后者覆盖用户关闭底图时的情形）。观感不变（背景色/圆角/阴影全部保留）。
  （定位与首版补丁来自对端机器实测；本版在其 3 条之外补齐了另外 3 条。）

### 1.8.4（2026-10-04）
- **把磨砂从「座位外框」搬到「输入卡本体」**：`[data-composer-seat]::before` **整块删除**（连同它的 `mask-image`
  与 1.8.2 加的层提升）。
  原因：座位比输入卡更宽，铺在座位上的 `backdrop-filter` 会连 padding 区一起模糊 ⇒ 糊到输入卡右侧 ✗。
  `backdrop-filter` 只放输入卡本体后，它跟随自身 border-box + 圆角，物理上不可能越界 ✓。
- 副作用（正面）：顶部那条硬分界线消失（用户反馈「毛玻璃上面的透明部分修好了」✓）。

### 1.8.3（2026-10-04）
- 给座位磨砂层补 `border-radius: inherit`（当时的误判修复；该层已在 1.8.4 整块删除，此条仅存档）。

### 1.8.2（2026-10-04）
- **修复「hover 到下方上下文用量指示器时画面闪烁」**：输入卡渐变描边原用 `mask-composite: exclude` 挖空，
  hover 卡内元素会令 mask 反复重算 ⇒ 改为**等价的 inset box-shadow 方向性描边**（上缘高光 / 左右衰减 / 下缘暗边），观感一致、成本极低。
- 磨砂层加 `will-change: transform; transform: translateZ(0)` 做**层提升**（该层已在 1.8.4 删除）。

### 1.8.1（2026-10-04）
- **修复「hover 到用户消息的复制按钮时界面鬼畜」**：移除气泡的 `position: relative` + `::before{backdrop-filter}` 层。
  机制：**`backdrop-filter` 即使 `blur(0px)` 也会创建「背景根」**，按钮的 hover 过渡会令其反复重算/重合成。
  代价：**「气泡模糊」滑杆自本版起不再生效**（气泡仍由 `--dsw-specific-bubble` 的 alpha 提供透明底）。

### 1.8.0（2026-10-04）
- 并入对端机器侧实测修复：`[class*="_frame"]` 与 `::before` 透明化（Windows 平台特化规则 `[data-windows-titlebar]` 会吃掉 `--dsw-specific-sidebar-fill`）。
- 侧栏改为**跟随底图**（与画布同一公式，联动「背景遮蔽」滑杆）。

## 十二、本皮肤的性能约束（改之前务必读）

1. **不使用任何 `:hover` 规则**；**不使用 `mask`**（两者都会被官方 hover 触发重算 ⇒ 闪烁）；
2. `backdrop-filter` 只放在**小面积、无后代、不会被 hover 反复失效**的元素上。
   本皮肤现在有 3 处，全部是"无后代的 ::before"：输入卡 `[data-composer-card]::before`、
   提问卡/计划评审卡 `> section::before`、以及浮层材质层 `[data-menu-material] > [aria-hidden]:first-child`
   （官方自己就是这么做的，那个 div 没有子元素）；
3. 任何"铺满宿主"的伪元素必须 `border-radius: inherit`；
4. **绝不把 `backdrop-filter` 放在"里面还会浮出别的面板"的容器本体上**（见 §十三）。

---

## 十三、毛玻璃失效的**两层**原因（1.8.7→1.9.2 的全部结论）

「面板看着还是透的」有两层独立原因，**要分别修**，1.9.x 一路就是这么拆出来的。

### 第一层：材质 token 被别人盖住（1.8.7 / 1.8.8 修）

`/` 命令面板的底板不在宿主本体上，而在 MenuSurface 塞进根节点里的一个**无后代子 div**：

```html
<div data-menu-material="translucent" class="_surface_…">   <!-- 面板本体 -->
  <div aria-hidden="true" class="_material_…"></div>        <!-- 真正的底板 -->
  …菜单项…
</div>
```
```css
._material_… { background: var(--dsw-menu-surface-fill); backdrop-filter: var(--dsw-menu-backdrop-filter); }
```

所以给浮层统一材质有两条路：**改 token**（`--dsw-menu-surface-fill` / `--dsw-specific-menu` / `--dsw-hovercard-bg`），
或者**按属性锚点直接重画这一层**。皮肤两条都做了（前者在 `skin.css` 的 body 块，后者在 `patches.css` 第 10 节）。
⚠️ token 必须写在 **body 级选择器**上，原因见 §七 铁律 4。

### 第二层（真凶）：祖先的 `backdrop-filter` 形成「背景根」（1.8.9 修）

**`backdrop-filter` 会给它的所有后代造成「背景根(backdrop root)」——后代自己的 `backdrop-filter`
只能糊到该根以内的内容，页面真正在后面的东西一律糊不到。**

而 **`/` 命令面板恰恰是输入卡的后代**：

- `ui-input-trigger` 源码自己用 `listRef.current.closest("[data-composer-card]")` 判断点击是否落在卡内；
- 面板 CSS 是 `bottom: calc(100% + 4px); left:0; right:0` —— 贴着卡的上沿、用卡的宽度；
- 截图实测：面板左右边缘 x≈34/1458，与输入卡的 34/1458 **完全一致**。

而 1.8.4 起磨砂是放在 `[data-composer-card]` **本体**上的 ⇒ 面板的 `backdrop-filter` 只能糊到卡片自己
那块纯色底板 ⇒ 等于没糊，下面的正文原样透出来。

Chrome 最小复现（同结构、面板放在祖先盒子之外）：

| 磨砂挂在哪 | 面板区域条纹扩散 | 结论 |
|---|---|---|
| 容器**本体** | **178** | 完全没糊 ❌ |
| 容器 **`::before`** | **0** | 完全糊上 ✅ |

**结论（写新皮肤时的通用规则）**：容器里还会浮出别的面板时，容器的 `backdrop-filter` 必须挂在
**它的 `::before`**（无后代）上，绝不能挂在容器本体上。

### 顺带：一个用来定位「修错地方没有」的探针

怀疑"样式到底打没打到这个元素"时，加一条显眼的

```css
[data-menu-material] { outline: 3px solid #d946ef !important; }
```

- 看得到描边 ⇒ CSS 确实命中该元素，问题只在材质属性（数值/属性）上；
- 看不到 ⇒ 选择器压根没命中，别在材质上调参了，先去拿真实 DOM。

1.9.0 就是靠它一次性确认了"位置从头到尾是对的、只是不透明度数值给低了"。（验证完已删。）
