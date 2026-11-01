# dsh-eto-boot · 地球三体组织（ETO）安全终端

DeepSeek Harness 的启动登录屏插件：启动时整屏接管，徽记描线 → `SECURE TERMINAL · CLEARANCE REQUIRED` → 触屏自动敲口令 → 验证序列逐行 `OK` → `你并不孤单。` → CRT 压成亮线关屏，然后交还给应用。

```
版本 1.0.1
```

> 1.0.1 是基于 1.0.0 的修复版：修掉「幕布右侧少一条」「刷新页面被空幕布盖死」，并把画布后备分辨率改成跟随真实屏幕密度、横竖屏切换自动重排。改动明细见 [1.0.1 修了什么](#101-修了什么)。原始文档保留在 [`README.upstream.md`](README.upstream.md)。

---

## 效果一览

| 阶段 | 画面 | 声音 |
| --- | --- | --- |
| 开机 | CRT 亮线横展开 → 徽记外环描线 → 刻度逐格亮 → 三条轨道与三星淡入 | 低频“嗡” + 通电噪声 |
| 登录 | 徽记 + `// EARTH-TRISOLARIS ORGANIZATION //` + 口令行 | 「安全终端。请输入访问口令。」 |
| 自动输入 | 点屏幕即逐字填入口令（密文解出、以圆点显示） | 打字滴答 |
| 验证 | `> 正在与节点 ████-0734 建立握手 …` 逐行打出，右侧绿色 `OK` | 逐句中文播报 |
| 收尾 | 扫带 + 闪白 → `你并不孤单。` → 宣言 | 「你并不孤单。消灭人类暴政，世界属于三体。」 |
| 转场 | 画面压扁成亮线 → 收点 → 消失 | 断电“啪” |

失败时：抖动 + 干扰条纹 + 红色暗角脉冲 + 右下角 `ATTEMPT 1/3 · ●○○`。Esc / 右上角「跳过 SKIP」随时退出；验证序列 14 秒无操作自动放行。

---

## 安装

### 推荐：解压后用 `link:` 装目录

```bash
# 1) 解压到固定位置（以后别移动/删除）
mkdir -p /root/dsh-eto-boot && tar xzf dsh-eto-boot-1.0.1.tar.gz -C /root/dsh-eto-boot
# 装的是「含 package.json 的那一层」，不要多套一层同名目录

# 2) 装进 web profile
dsh plugin --profile web add link:/root/dsh-eto-boot

# 3) 重启 dsh web，然后浏览器 F5
```

`link:` 是软链安装，改了源目录立即生效；**但装完之后不要移动/重命名该目录**，挪了要重新 `add` 一次。想改成拷贝安装用 `dsh plugin --profile web add file:/root/dsh-eto-boot`（此后源目录再改不会同步）。

### 不要直接 `dsh plugin add <包>.tar.gz`

实测（pnpm 10.34）：直接装 tarball 会把包**摊平**——`lib/` 和 `locale/` 被拆到包根目录，而 `package.json` 的 `main` 仍是 `lib/index.js`，结果插件加载失败且不报错（右下角什么都没发生）：

```text
node_modules/dsh-eto-boot/index.js      # ← package.json 说主入口是 lib/index.js
node_modules/dsh-eto-boot/splash.js     # ← lib/index.js 里 LIB_ROOT 也算到 node_modules
```

先解压再 `link:`（或 `file:`）能避开。

### 与其它启动动画共存

同一 profile **只保留一个**整屏启动动画插件，装本插件前先移除 `dsh-550c-boot`、`dsh-threebody-splash` 之类的同类插件：

```bash
dsh plugin --profile web remove <同类插件名>
```

---

## 1.0.1 修了什么

### 1. 幕布右侧露一条（Android 真机复现）

**现象**：登录幕只铺到屏幕宽的约 90%，右边漏一条，露出的是应用底色 `#151517` 加贴着右缘的小鲸鱼挂件。桌面端不复现，只在 DSH App（Android WebView）上出现。

**测量**（OnePlus PKG110 / Android 16 / dpr 3.5 / 竖屏 iw=361 ih=754）：

```text
采样    幕布宽   100vw 探针   inline style   transform / zoom   最右缘最上层
0.2s     361        361          空            none / 1         #eto-gate（幕布自己）
0.6s     361        361          空            none / 1         #eto-gate
1.2s 起  329        361          空            none / 1         应用的 _uV2eYG_root
```

同一时刻 `100vw` 探针仍是 361、`sbW = 0`（没有滚动条）、`transform` 与 `zoom` 都是默认值，也没有 inline 尺寸——幕布是**自己少了 32px**，而且发生在应用启动完成之后。

**根因**：DSH App 在启动后（约 1s）加载/生效的某条 CSS 改写了 `#eto-root` 的盒子，把整屏接管的幕布挤窄了 32px。与缩放、滚动条、视口单位都无关。

**修复**：[`lib/splash.css`](lib/splash.css) 把盒子钉死，谁也别想再改：

```css
#eto-root.eto {
  position: fixed !important;
  left: 0 !important; top: 0 !important;
  right: auto !important; bottom: auto !important;
  width: 100vw !important; height: 100dvh !important;
  max-width: none !important; max-height: none !important;
  margin: 0 !important; transform: none !important;
  box-sizing: border-box !important;
}
```

外加底色兜底：`html.eto-lock` / `html.eto-lock body` / `html.eto-first body` 在登录幕期间压成 `#08090a`（body 背景会传播到 canvas、铺满整屏），万一再漏也只会是「更黑的一条」，不会露出应用底色。

### 2. 刷新页面被空幕布盖死

**现象**：`mode` 为 `session` / `daily` 时，本会话已经播过、再刷新页面，会被一张没有脚本驱动的空登录幕永久盖住。

**根因**：注入的 `#eto-root` 是常驻 DOM，而 `splash.js` 在「这次不该播」的分支里只 `return`，没把幕布摘掉。

**修复**：两条跳过分支都先 `__etoHideCurtain()`（移除 `#eto-root`、清掉 `eto-first` / `eto-lock`）。

### 3. 画布后备分辨率与屏幕不匹配（横屏尤其明显）

**根因**：`bgLayout()` 里 `BD = min(devicePixelRatio, 2)` 把上限钉死在 2x。这块屏是 **dpr 3.5**，横屏可视区 754×321 css（物理 2640×1124），2x 的 canvas 只有 1508×642，被放大 1.75 倍绘制——星空和 46px 战术网格发虚。

**修复**：按真实 dpr 取，上限 3x，并加 600 万像素预算（超预算按 0.25 递减）：

```js
var d = Math.min(window.devicePixelRatio || 1, 3)
while (d > 1 && w * h * d * d > 6e6) d -= 0.25
BD = Math.max(1, d)
```

竖屏 361×754 → 1083×2262（2.45M 像素），横屏 754×321 → 2262×963（2.18M 像素），都在预算内。

### 4. 旋转后不重排

Android 旋转时 `innerWidth/innerHeight` 上报有延迟，只重排一次会停在中间态（尺寸已横屏、canvas 还是竖屏分辨率）。

**修复**：`relayout()` 一次做完「钉盒子 + 重算 canvas 后备分辨率 + 重绘 + 软键盘避让」，挂在 `resize` / `orientationchange` / `visualViewport.resize`，并在旋转后 **120 / 400 / 900ms 各补一次**。

### 其它

- 跳过时同时清掉 `html` 上残留的 `eto-first`。
- 首帧覆盖补 `html.eto-first body{background:#08090a}`（body 背景会盖过 html 的）。

---

## 设置页

设置侧栏会多出一栏：**地球三体ETO组织启动动画设置**（不占用「通用」）。

### 启动行为

| 项 | 说明 |
| --- | --- |
| 关闭 / 会话一次 / 每次 | 是否在启动时拦一道登录 |
| 语音 / 音效 | 分别开关 |
| 预览 | 新开 `/dsh-eto/preview`，随时重看 |

配置文件在 `~/.dsh/dsh-eto-boot.json`，字段：`enabled` / `mode`（`session` `always` `daily` `off`）/ `allowSkip` / `voice` / `sound` / `mimoKey` / `mimoVoice` / `mimoTone` / `mimoSpeed` / `space`。

### MiMo 语音合成（可选）

| 项 | 说明 |
| --- | --- |
| API Key | 只存 Host（`~/.dsh/dsh-eto-boot.json`），接口只回「已配置」，不把明文发给浏览器 |
| 音色 | 冰糖 / 茉莉 / 苏打 / 白桦 |
| 音调 | 低沉 / 标准 / 明亮 |
| 语速 | 慢 / 正常 / 快（映射成 MiMo 的风格指令） |
| 空间感 | 关 / 轻 / 中 —— 播放时在 WebAudio 里加一路 110ms 反馈延迟，不依赖 ffmpeg |
| 试听 / 生成全部语音 | 合成 9 句缓存到 `~/.dsh/eto-tts/<音色>-<音调>/` |

没填 Key 或没生成时回退到内置预生成女声（`lib/voice.js`，9 段 MP3 base64 内联）。语音走 **WebAudio**，不依赖系统 TTS；同时只播一段、其余排队；首次交互前挂起的那句会在第一次触摸时补播。

---

## 口令

口令以**密文**存放，源码里没有明文：

- 自动输入：异或密文 `PHRASE_CIPHER` + 密钥 `PHRASE_KEY`，运行时解出后逐字敲入
- 校验：`PHRASE_HASH`（SHA-256）与输入现算的摘要比对

换口令要同时改这两处（摘要 + 密文），二者必须指向同一个口令。

> 这是**主题化的登录屏**：脚本随页面下发、摘要也在里面，能被人离线爆破。它拦的是误触与仪式感，真正的访问控制仍由 DSH 自己的 token 负责。忘了口令随时「跳过 SKIP」或 Esc。

---

## 路由

| 路径 | 用途 |
| --- | --- |
| `/dsh-eto/config` | 读写配置（Key 不回显） |
| `/dsh-eto/preview` | 独立预览页（带 INITIALIZE 手势层） |
| `/dsh-eto/tts/status` | 语音是否已生成、当前音色/音调 |
| `/dsh-eto/tts/preview?text=…` | 合成一句返回 MP3 |
| `/dsh-eto/tts/generate` | POST，合成全部 9 句 |
| `/dsh-eto/tts/clip/<key>.mp3` | 取已生成的片段 |
| `/dsh-eto/voice.js` `/logo.js` `/starfield.js` `/splash.js` `/splash.css` | 静态资源 |

自检（浏览器已登录，带 cookie）：

```bash
curl -s -b cookie.txt "http://127.0.0.1:3080/dsh-eto/config"                        # 200 JSON
curl -s -o /dev/null -w "%{http_code}\n" -b cookie.txt "http://127.0.0.1:3080/dsh-eto/splash.js"   # 200
```

---

## 文件结构

```text
lib/
  index.js       Host：首帧覆盖、index 注入行、/config、/preview、/tts/*、静态资源
  client.js      设置页（settings.section）「地球三体ETO组织启动动画设置」
  root.html      登录幕骨架：徽记、口令行、验证序列、宣言、开机线/扫带/闪白
  splash.css     全部样式：CRT、星场叠层、横屏布局、动效、盒子钉死
  splash.js      口令校验、自动输入、验证序列、语音队列、CRT 关屏、键盘与旋转适配
  voice.js       内置语音包（9 段 MP3，base64 内联）
  logo.js        动态徽记组件（任何 [data-eto-logo] 自动填充）
  starfield.js   通用动态星空背景（任何 canvas[data-eto-bg] 自动挂载）
locale/          zh.json / en.json（插件元信息）
icon.svg         插件列表图标
cordis.patch.yml bundle 挂载声明
```

发布包里**不含** `lib/audition/`（语音试听页）、`test/`、`tools/`、`preview/`。

---

## 已知限制

- **横屏两侧的黑边不是幕布**：Android 上 DSH 的 WebView 不是 edge-to-edge，横屏时导航栏占左侧 40 css、状态栏占上边 40 css（实测纯 `#000`，在 WebView 之外）。幕布已经铺满 WebView（横屏 754×321 css，右/下齐边）；要真·沉浸得 App 侧开 edge-to-edge，插件做不到。
- **`engines.dsh` 声明是 `>=0.2.0-rc.1`**，但 `webserver/index-inject` 的 `global/style/script/html/script-src` 五个 kind 在 `0.1.7-rc.2` 上就有实现，实测可跑；声明偏保守。
- **只支持 `web` profile**：官方桌面端（Electron）读的是 `desktop` profile，用 `--profile web` 装进去它读不到，桌面端要走它自己的插件管理。
- 与其它整屏启动动画插件**互斥**，同一 profile 只留一个。

---

## 卸载与回滚

```bash
dsh plugin --profile web remove dsh-eto-boot    
