# 一键安装 prompt（丢给 DSH 用）

把下面代码块里的**整段话**复制，粘贴给你的 DSH（桌面版）发出去，它就会自己去装、自己验证、最后回报结果。
macOS / Windows 都适用，不需要你手动敲命令。

> 前提：DSH **桌面版**。皮肤是纯 CSS + 图片的声明式皮肤（无 `hooks.mjs`、不执行代码），来源即本仓库。
> 想自己动手装（4 步图文）看 [README](README.md)；想了解原理和排障看 [INSTALL.md](INSTALL.md)。

---

```text
请帮我在本机的 DSH 桌面版上安装皮肤「原生玻璃（native-glass）」，按下面的步骤执行，每一步都先验证再往下走，最后把结果汇报给我。

【要装的东西】
- 皮肤：native-glass v1.9.3，来源仓库 https://github.com/NianwYue/native-glass-skin
- 依赖插件：@linxin666/dsh-client-ui-skin-center 0.4.4
- 皮肤是纯声明式（只有 CSS 和图片，没有 hooks.mjs，不执行任何代码）
- 关键约束：皮肤在皮肤中心里的 id 是 native-glass，安装目录名必须正好是 native-glass

【步骤】

1. 先探测环境并告诉我结论：
   - 操作系统与 DSH 版本
   - DSH_HOME 的实际取值（默认 ~/.dsh；Windows 为 %USERPROFILE%\.dsh），若设过 DSH_SKINS_HOME / DSH_SKINS_DIR 以它们为准
   - GUI 实际在用的 profile 名（常见是 desktop）
   - 皮肤中心插件是否已安装、版本是多少

2. 若插件未安装或版本不是 0.4.4，用 dsh CLI 安装：
   dsh plugin --profile <实际 profile 名> add @linxin666/dsh-client-ui-skin-center@0.4.4
   CLI 位置：macOS 在 "/Applications/DeepSeek Harness.app/Contents/Resources/runtime/cli/bin/dsh"；
   Windows 在 DSH 安装目录的 resources\runtime\cli\bin\ 下。

3. 获取皮肤并放置：clone 或下载仓库（git clone --depth 1 https://github.com/NianwYue/native-glass-skin），
   把仓库里的 native-glass/ **整个目录**放到 <DSH_HOME>/skins/native-glass/。
   如果该目录已存在，先把旧目录改名备份（例如 native-glass.bak-<时间戳>）再拷贝。

4. 落盘核验（缺一不可）：
   - <DSH_HOME>/skins/native-glass/ 下存在：skin.json、skin.css、patches.css
   - assets/ 下有 frost-light.jpg 与 frost-dark.jpg；preview/ 下有 light.jpg、dark.jpg
   - skin.json 里 "id": "native-glass"、"version": "1.9.3"
   - 皮肤不需要重启 DSH 即可被收录

5. 让我在「设置 → 皮肤中心 → 原生玻璃 → 应用」点一下应用；你也可以直接读 <DSH_HOME>/skin-center-active.json 确认 "active" 是不是 native-glass，不是就告诉我。

6. 帮我把皮肤中心里的滑杆设成推荐值（若你无法直接设置，就把这张表给我，我自己拖）：
   背景遮蔽 = 0；背景模糊（空对话/有内容）= 20/20；输入卡模糊 = 14；气泡不透明度 = 0；气泡模糊 = 0。

7. 验证真的生效，并把证据给我：
   - GET {DSH地址}/api/skin-center/v2/catalog → 应出现 native-glass、version 1.9.3、warnings 为空
   - GET {DSH地址}/api/skin-center/v2/skins/native-glass/assets/frost-dark.jpg → 应返回 200（404 说明目录名不对）
   - 界面看起来没变化就是前端缓存，让我强刷一次（Ctrl+F5 / Cmd+Shift+R）
   - Windows 上若画布/侧栏是纯色（底图被挡住），按仓库 INSTALL.md 的 §六 检查 [class*="_frame"] 规则是否已下发

【边界】
- 只做上面这些事：不要修改皮肤文件内容，不要动其它插件与配置，不要改 DSH 本体
- 写入 ~/.dsh 目录若被沙箱拦下（路径在工作区之外），需要提权后再执行，不要就此跳过
- 遇到任何一步失败，把原始报错贴给我，不要猜着继续

最后请汇报：装的是哪个版本、皮肤目录的实际路径、上面每条验证的实际结果、以及我需要手动做的剩余动作（如果有）。
```
