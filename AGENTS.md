# AGENTS.md — dsh-whale-widget

给在本仓库改代码的 AI / 维护者。用户文档以 `README.md` 为准；完整需求草稿 `whale-widget-prompt.md` **已过期**（仍写 token 模式、具名导出等旧实现），**以本文件 + 当前源码为准**。

## 这是什么

DeepSeek Harness（DSH）Web 界面右下角的标准 bundle 插件：余额 / 今日已用 / 每轮消耗 / 自定义泡泡。npm 包名 `dsh-whale-widget`（当前版本见 `package.json`）。

两条产品线**互不兼容、无共享代码**：

| 分支 | 挂哪里 | 不要做什么 |
|---|---|---|
| `main`（本工作区） | DSH Web | 不要往这里塞 Codex 桌面安装物 |
| `For-Codex` | Codex 桌面 | 不要用 `dsh plugin add` 装进 DSH Web |

改 `main` 时不要「顺便统一」另一条线。

## 源码地图

| 路径 | 职责 | 改完怎么生效 |
|---|---|---|
| `lib/index.js` | 宿主：路由、余额、会话计费、自定义 API、角色/音效/图库 | **重启 `dsh web`** |
| `lib/accounting.mjs` | 定点金额 + 余额观测/校正账本（不读写文件） | 随宿主重启 |
| `assets/whale-widget.js` | 前端挂件（原生 JS IIFE，按文件 mtime 热读） | **Ctrl+F5 / 硬刷新** |
| `cordis.patch.yml` | 把插件插入 Web profile | 改完重启 |
| `package.json` | 包名、版本、`dsh.bundle.patch`、npm `files` | 发版才动 `version` |
| `assets/*` | 图/音效；**不在 MIT 内**，见 `PROVENANCE.md` | 缺文件则对应功能静默降级 |

运行时数据在 `$DSH_HOME`（默认 `~/.dsh`），**永不入库**：`.dshw-*.json`、`whale-roles/`、`whale-audio/`、`whale-bubble-imgs/`、凭据。密钥只走 DSH `ctx.credentials`，配置里只存凭据名（如 `DEEPSEEK_API_KEY`）。

本地开发（仓库根目录就是插件包，没有 `dsh-whale-widget/` 子目录）：

```bash
dsh plugin --profile web add link:.
# 然后重启 dsh web；改前端硬刷新即可
```

不要写成 `link:./dsh-whale-widget`。npm 名与 `github:` 源二选一，混装会冲突。

## 宿主约定（`lib/index.js`）

- 导出是 **default object**：`name: 'whale-balance-widget'`，`inject: ['webServer', 'credentials', 'connection']`，`apply(ctx)`。包名 / patch 名是 `dsh-whale-widget`，Cordis 插件名是 `whale-balance-widget`——**不要为了「统一命名」去改其中一边**。
- 新路由必须走已有的 `registerRoute`（内部先 `connection.requestRejection`）。不要直接 `webServer.register`。栅栏拒绝时返回 401/403 空 body 是预期。
- `tapIndex` 注入 `<script defer src="/dsh-whale/widget.js">` 必须幂等；所有 `register` / `on` / timer 的 disposer 推进 `disposers`，由 `ctx.effect` 在 HMR/卸载时清理。定时器要清句柄，否则 DSH 退不干净。
- JSON 路由用 `JSON_HEADERS`（`no-store`）。`/dsh-whale/balance.json` **任何情况 200 + JSON**，禁止空响应或让 handler 抛到 dispatcher。
- 路径一律 `fileURLToPath(import.meta.url)` 推 `PACKAGE_ROOT`，资源优先包内 `assets/`，再回退旧绝对路径。用户可写数据只落 `$DSH_HOME`。
- 账本写入：tmp + `renameSync`，Windows 共享冲突用 `ledgerIo` 重试；读失败的已有账本**禁止当成空账本覆盖**。旧格式首次迁移才写 `.before-recharge-fix.bak`（已存在不覆盖）。
- PUT 配置（尤其 `/dsh-whale/size.json`）：缺字段**沿用已有值**，不要用默认值把用户设置洗掉。
- 余额校正只允许内置 DeepSeek（`id === 'deepseek' && builtin && provider === 'deepseek'`）。其它厂商模板不要「顺便」打开这条能力。
- 计费：`reasoningTokens ⊆ outputTokens`，**不要再把 reasoning 加进输出**。高峰 = 工作日北京时间 9–12、14–18；`2026-08-23` 起周末全天谷价。调价只改文件顶部 `PRICING` / `BASE_PRICE` / `PRO_PRICE`。内置 DeepSeek **不支持**自定义单价。账本金额统一 CNY；美元单价必须带汇率再折算。
- `normalizeUsageMode()` 恒为 `'ledger'`。不要恢复 `DEEPSEEK_PLATFORM_TOKEN` / 实时·令牌模式。
- 自定义 API：模板只是默认值；用户留空字段继续用模板，不要当成「关掉」。`kind:'quota'` 与余额接口不是同一件事。新增中文厂商时给 `TPL_PINYIN_INITIAL` 补首字拼音，否则排序会甩到列表末尾。
- Node `fetch` 保持 `dns.setDefaultResultOrder('ipv4first')`。

## 前端约定（`assets/whale-widget.js`）

- 保持 **单文件 IIFE**。不要引入打包器、React、TypeScript 或拆成 ESM 模块，除非维护者明确要求。
- 双重闸门：`window.__dshWhaleWidget`（脚本级）+ `window.__dshWhaleInit`（初始化）。只在主聊天界面启动（`#root` 里有 `textarea` 或 `[contenteditable="true"]`）。市场等 SPA 页不要插 DOM、不要注册全局捕获——否则 React portal 会炸掉。未就绪则 `MutationObserver` **无限等待**，不要加「几秒后永久放弃」。
- 音效用 Web Audio shim（`dshwvSound`），不要改回 `<audio>`（会进系统「正在播放」）。
- 定位**永远 `left`/`top` 像素**。禁止 `right:0` / `left:auto` 吸附，CSS 不能对 `auto` 插值。`settle()` 按锚点重算；缩放时以鲸鱼所在角为固定点。
- 翻转、按压、文字出现：过渡必须按属性拆分，需要时对根元素临时 `transition:none`（滑块/缩放），否则会抖或文字滞后。
- 泡泡：最多 6 行、每行最多 6 模块；`image` / `randimg` 独占一行且**每个泡泡只能有一个**图片类模块。编辑器与渲染层规则必须一起改。
- 样式单位跟 `--dshw-u`（画布宽 / 1026），不要写死 px，除非是菜单/管理窗口这类独立 UI。

## 记账内核（`lib/accounting.mjs`）

- 金额 8 位小数（内部 ×1e8 整数）。余额下降记消费，上升记 `credit`，**充值不得冲减已观测消费**。校正公式用显式到账 / 非调用扣减，带 `revision` 防并发覆盖。
- 本模块只算账，不碰文件系统。观测窗口按 `scope + currency` 隔离。
- 「已观测消费」与「余额校正」是 DeepSeek 账户口径；「本机模型费用」才是全模型本地估算。不要把两套数字混成一个「今日已用」。

## 发布与验证

- 发版：只改 `package.json` 的 `version` 再推进 `main`。workflow **不会自动 bump**；同版本已在 npm 则跳过。不要在无关 PR 里改版本号。
- `package.json` → `files` 决定进 npm 的内容。不要把开发笔记、本机路径、运行时 json 加进去。`AGENTS.md` 默认不进包，保持这样。
- 裸 `curl http://127.0.0.1:3080/dsh-whale/*` 现在应是 **401/403**（浏览器信任栅栏），不是接口坏了。带 DSH 会话的浏览器访问才是 200。验证「路由还在」看 401/403；验证业务看浏览器或已登录会话。
- 改宿主后重启 `dsh web`；只改 `whale-widget.js` 则硬刷新。自测至少覆盖：挂件仅在主聊天出现、拖拽四边吸附、左吸附翻转、点击泡泡序列、余额刷新、每轮消耗泡泡。

## 不要做的事

- 不要把密钥、`.dshw-*.json`、用户图库/音效提交进 git。
- 不要改 `assets/` 里 PNG 的像素或重新导出而不剥离文本/EXIF 元数据（见 `PROVENANCE.md`）。不要把素材改成 MIT 或声称原创。
- 不要为了「现代化」重写前端或把宿主改成 TypeScript。
- 不要用动态 Cordis 插件（`cordis_define`）替代 bundle：那样无法随页面打开自启。
- 不要在 `cordis.patch.yml` 里写 `name: ./xxx.mjs?v=N`（那是旧手动安装热更写法，发布包会启动失败）。
- 不要在 issue/PR 里用本机绝对路径当默认路径；代码里的旧路径只允许当 fallback。
