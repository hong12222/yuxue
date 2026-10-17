语学（Yuxue）— 代码与架构介绍
一个把自己做成「刷题外壳」的安卓学习 App：出题、判分、学情报告全部走用户自己配置的 AI 接口，
没配 AI 时用内置离线题库顶上，保证打开就能刷。
当前版本 2.7.0，包名 com.yuxue.app。

1. 一句话概括
维度	事实
形态	安卓 APK（原生 WebView 外壳） + 单文件网页版（双击即用）
前端	原生 JavaScript（无框架、无打包器、无 npm 运行时依赖），20 个模块脚本 + 2 个数据文件
后端	没有后端。AI 请求由用户在设置里填的 Base URL / Key 直连，数据存在本机 KV
规模	src/www 约 918 KB；JS 约 10 200 行，CSS 约 640 行
知识点库	962 个知识点、5 个学段（小学 / 初中 / 高中 / 大学 / 考研）、5 套考试方案
离线题库	194 道内置题（31 个科目）+ 150 道可本地判分的扩展填空题 + 6 个预设题库
激励系统	24 项成就 + 经验值 / 等级
测试	5 个 jsdom 无头测试套件，198 条断言，当前全绿
2. 产品结构：四个 Tab
底栏固定四页（js/app.js 的 TABS），每页首次进入时才挂载：

Tab	视图	模块	做什么
刷题	#v-home	feed.js	抖音式无限流：上下滑动出题，选择题点选即判、主观题交 AI 阅卷；报错 / 收藏 / 斩 / 我不会 / 暂时不做 / 划走重刷
题库	#v-bank	bank.js	按学段-年级-科目-知识点挑题源，主流考试一键建库，自定义建库，把 AI 题缓存到本机离线刷，举一反三
学情	#v-stats	statsview.js	正确率趋势、科目/知识点分布、薄弱点、AI 学习报告与下一步推荐
我的	#v-me	profile.js	个人信息、错题本、收藏夹、历史记录、成就墙、学习画像、数据备份、设置
设置是三级结构（settings.js）：设置 → 通用 / AI 设置 / 个性化 → 各自整页。

3. 总体架构
┌─────────────────────────── Android 外壳（预编译 APK，不在源码里） ───────────────────────────┐
│  MainActivity（WebView + @JavascriptInterface 方法，以 window.YX 暴露给网页）                 │
│  能力：KV 存储 / 异步 HTTP / 触感 / TTS 朗读 / 剪贴板 / 分享 / 导出导入 / 主题 / 返回键      │
└───────────────────────────────────────────┬────────────────────────────────────────────────┘
                                            │ window.YX（网页版由 websingle.js 注入等价实现）
┌───────────────────────────────────────────▼────────────────────────────────────────────────┐
│  纯前端单页应用（src/www）                                                                   │
│                                                                                             │
│  index.html ──> 22 个脚本按依赖顺序串联（20 个模块 + 2 个数据文件），全局命名空间 w.X       │
│                                                                                             │
│  bridge.js ── 桥接层（有原生走原生，无原生降级 localStorage / fetch，即浏览器预览模式）      │
│  store.js  ── 单一状态树 DB.state（设置 / 题库 / 记录 / 错题 / 收藏 / 统计），防原型污染合并  │
│  ai.js     ── AI 层：出题、判分、追问、纠错、变式、报告、推荐、连通性测试                     │
│  offline.js── 出题源调度 SRC：内置题库 + 用户缓存题 ⇄ AI，在线/离线自动切换与自愈            │
│  reco.js   ── 默认推荐：题库 / 最近刷过 / 薄弱点 / 学段 加权，每个知识点都带「为什么」        │
│  feed.js   ── 刷题流（最重的一个模块，1 859 行）                                             │
│  bank.js / statsview.js / profile.js / settings.js / guide.js / persona.js / ach.js / ...    │
│  util.js（DOM、md 渲染、Toast、BottomSheet） fx.js（Web Audio 音效 + 分级触感 + 朗读）        │
│  icons.js（24×24 线性矢量图标，无 emoji）  app.css（主题 token + 响应式）                    │
└───────────────────────────────────────────┬────────────────────────────────────────────────┘
                                            │ HTTPS（直连）
                                  ┌─────────▼──────────┐
                                  │ 用户自己的 AI 接口   │
                                  │ DeepSeek / 通义 /    │
                                  │ Kimi / 智谱 / OpenAI │
                                  │ / Claude（Anthropic）│
                                  └────────────────────┘
一句话：原生外壳只提供「手机能力」，一切业务逻辑都是可以被 jsdom 无头跑起来的前端代码。
这也是整套测试能在没有手机的情况下跑真实业务链路的原因。

4. 目录与文件职责
语学/
├── base.apk              原始外壳 APK（可直接解压；dex 里只有 MainActivity + R 共 7 个类）
├── extract/              展开后的外壳（AndroidManifest.xml / classes.dex / res / assets）
├── keys/yuxue-local.jks  本地签名密钥（别名 yuxue）
├── src/www/              ← 唯一的业务源码目录
│   ├── index.html        骨架：启动页、四个 view、底栏、sheet/toast/busy 容器
│   ├── css/app.css       主题 token + 组件样式 + 宽屏分栏
│   └── js/
├── build/                构建脚本、中间产物、历史版本 APK / 网页版
└── test/                 jsdom 无头测试（自带 package.json，只有 jsdom 一个依赖）
模块清单（按加载顺序）
文件	行数	职责
taxonomy.js	数据	962 个知识点 + 5 学段 + 5 套考试方案（自动生成，勿手改）
offline-bank.js	数据	194 道内置离线题（自动生成，勿手改）
offline-extra.js	166	内置扩展题库：150 道可本地判分的填空题（单词拼写 / 口算等）
presets.js	101	6 个预设题库：把扩展题库里某一组题一次性缓存进题库，建库后完全离线可刷
bridge.js	121	原生桥 window.B：KV、异步 HTTP（Promise 化 + 320 s 兜底超时）、触感、TTS、导出导入，浏览器下自动降级
util.js	320	U：$/$$/h() 建元素、md() 轻量 Markdown、toast()、sheet() 底部弹层、主题、格式化
store.js	666	DB：状态树、迁移、容量上限、记录/错题/收藏/划走/纠错、统计（trend、bySubject、byPoint、weakPoints、streak、goal）、导出导入
fx.js	217	答题反馈：Web Audio 合成音效（零资源文件）、6 级触感、读题朗读、去重节流
ai.js	831	AI：OpenAI 兼容 + Anthropic 双格式、端点候选探测、JSON 模式回退、出题/判分/追问/纠错/变式/报告/推荐/连通性测试
offline.js	351	SRC：出题源调度与在线/离线状态机（unknown→checking→online/offline）+ 自动重试
reco.js	331	RECO：默认推荐打分（题库、最近、薄弱、学段），输出「为什么推荐」
feed.js	1859	Feed：无限流刷题、出题队列、答题交互、自动切源、定题模式（错题重练/薄弱点强化）
bank.js	1171	Bank/BankView/TAX：知识点检索、考试方案建库、自定义建库、离线缓存、并行补充、举一反三
statsview.js	392	StatsView：学情页与报告独立页，生成过程非阻塞
ach.js	153	ACH：24 项成就 + 经验/等级（答对 10 点、答过 2 点）
persona.js	465	PERSONA：用真实记录请 AI 写「你是什么样的学习者」画像
remind.js	103	REMIND：应用内学习提醒（前台可见时才检查，一天一次）
profile.js	712	Profile：我的页、错题本、收藏、历史、关于与更新日志（VERSION = '2.7.0'）
settings.js	685	SET：三级设置页（通用 / AI / 个性化）
guide.js	459	GUIDE：8 屏新手引导，可跳过可回看
app.js	220	入口：底栏、主题、全局触感委托、返回键双击退出、生命周期
5. 关键运行时链路
5.1 启动
app.js:boot() →

DB.load() 读 KV，merge() 合并进默认值（防 __proto__/constructor 注入），跑 migrate()
先落地主题（避免闪白/闪黑）→ 画启动页 logo → 建底栏 → 绑全局触感 → 应用界面偏好
420 ms 后隐藏启动页，进入「刷题」，首启弹欢迎引导，2.2 s 后检查学习提醒
900 ms 后写日志：知识点数、离线题数、bridge= 是否原生、AI 是否已配置
5.2 一次出题的来源决策（offline.js）
SRC.fetch(spec, n, exclude)
   │
   ├─ SRC.ai.state === 'online'  ──> AI.genQuestions()  ──失败/超时──┐
   │                                                                │
   └─ 其余情况 ──> SRC.offline(spec, n)  内置 194 题 + 题库缓存题 <──┘ 自动降级
状态机：unknown → checking → online / offline，SRC.probe()/forceProbe() 探测，SRC.configChanged() 改配置后重置
断网时只发能本地判分的选择题和填空题（主观题离线判不了，避免卡在「评分失败」）
每批题带 genEpoch 代际号，过期响应直接丢弃，避免旧批次污染新队列
5.3 一次 AI 请求（ai.js:AI.chat）
AI.status() 先做配置体检（缺 Base URL / 模型 / Key 直接短路，不发网络）
按 format 组包：openai → POST {base}/chat/completions；anthropic → POST /v1/messages（system 提出来单独放）
JSON 模式：response_format: {type:'json_object'}，若 provider 以 400 拒绝则去掉该字段重发一次
端点候选：路径猜错（404/405 等）自动换下一个候选 URL，成功的地址记进 EP_OK 缓存
超时：请求自身 timeoutMs（默认 90 s），桥层 320 s，offline.js 外层 60 s
返回统一成 {ok, text, usage, ms, url}，上层模块从不直接碰 HTTP
5.4 判分分工
题型	判分者
选择题	本地比对选项字母
填空题 / 数值题	本地归一化判分（AI.gradeFill / gradeLocal）
主观题	AI 阅卷（AI.grade），支持「追问」继续对话
6. 数据模型（store.js）
单一状态树，单键 state 落 KV（原生走文件，网页版走 localStorage['yuxue.kv.state']）：

{
  v: 1, createdAt,
  profile: { nickname, avatar, stage, grade, age, target, bio },
  settings: {
    api:    { baseUrl, apiKey, model, format, temperature, maxTokens, jsonMode, extraHeaders, preset, timeoutMs },
    gen:    { batch, choiceRatio, difficulty, preferWrong, concurrent },
    quiz:   { tapSubmit, autoNext, autoWrong, vibrate, sound, keepScreen, shuffleOptions,
              showTimer, showExplain, readAuto, readRate, redoWeak },
    ui:     { theme, font, density, wide, anim, showStat, showChip, reco: {...} },
    goal:   { on, daily },
    remind: { on, time, onlyIfGoal }
  },
  banks: [],      // 用户题库（含 cached 离线缓存题）
  wrong: [],      // 错题本
  fav: [],        // 收藏
  chopped: [],    // 划走的题（按题干指纹）
  reports: [],    // 题目纠错
  records: [],    // 答题记录
  sessions: [],   // 刷题会话
  feed: {}, skips: {}, report, days: {}
}
设计要点：

容量上限：records ≤ 4000、sessions ≤ 300、wrong ≤ 800、fav ≤ 500，超了从头部丢，防止状态无限膨胀拖慢启动
写盘节流：DB.save() 是 350 ms 防抖，DB.flush() 立即写；visibilitychange/beforeunload 强制 flush，强杀进程也不丢设置
防御性合并：merge() 拒绝 __proto__/constructor/prototype，且不触碰「原型链上有、自己没有」的键 —— 因为导入备份是外部不可信数据
一次性迁移：mig.batchMigrated 之类标记，保证「每批 5 题」这种合法选择不会每次启动被打回默认值
7. UI 与适配
主题 token：:root 定义暗色（默认，#0B1419 底 + #4CC2FF 主色）与 [data-theme="light"] 全套变量，color-scheme 同步，B.setTheme() 联动状态栏
安全区：env(safe-area-inset-*) 处理刘海与手势条
宽屏：#app[data-wide="auto|single|split"]，min-width:700px 时选择题/主观题卡片左右分栏（题干在左、作答与判定在右）
个性化：data-font（字号）、data-density（密度）、.no-anim（关动画 + prefers-reduced-motion）、.hide-stat/.hide-chip（隐藏首页元素）、data-nohaptic（豁免触感）
零图片资源：图标全是 icons.js 里的 24×24 描边路径，音效由 Web Audio 现场合成，启动 logo 也是矢量
8. 构建与发布
外壳是预编译的，构建只做「换 www + 改版本号 + 对齐 + 重签」：

# 单包
bash build/build.sh <版本名> <versionCode>     # 例：bash build/build.sh 2.8.0 12

# 双产物：APK + 单文件网页版，并复制到 /sdcard/Download/
bash build/release.sh <版本名> <versionCode>
build.sh 六步：

extract/ → build/stage（去掉旧 META-INF）
patch-manifest.js 改写清单里的 versionName / versionCode
zipapk.js 复制原包全部条目（跳过 assets/），再把 src/www/* 逐文件映射成 assets/www/*
zipalign -f -p 4
apksigner 用 keys/yuxue-local.jks 签 v1 + v2 + v3
apksigner verify + aapt dump badging 自检
websingle.js 把整站内联成一个 .html，三处必须处理对（脚本里写明了）：

内联 <script> 里的 </script> 要转义，否则提前闭合
内联 <style> 里的 </style> 同理
真机的 window.YX 由 Android 注入，网页里没有 → 内联等价实现走 localStorage，能力缺失时优雅降级
工具链走的是手机上的 Extension Center（android-buildtools + openjdk-17），build.sh 里显式设置了 PATH 和 LD_LIBRARY_PATH（保留 node 自己的 lib，否则 node 起不来）。

9. 测试
test/ 用 jsdom 真实加载 src/www/index.html 和全部脚本，并注入一个假的安卓桥 window.YX，
再喂一个假的 AI 服务（ai 回调决定每次请求返回什么），于是整条业务链路都能在无手机环境跑：

cd test
node smoke.test.js       # 47 条：宿主/数据/出题/答题/断网降级/零 JS 错误
node features.test.js    # 62 条：功能链路
node security.test.js    # 51 条：备份导入、原型污染、越界数据
node bugs.test.js        # 16 条：历史缺陷回归
node webbuild.test.js ../build/语学-2.7.0-web.html   # 22 条：单文件网页版自检
最近一次运行结果：

smoke    47 通过 / 0 失败
features 62 通过 / 0 失败
security 51 通过 / 0 失败
bugs     16 通过 / 0 失败
webbuild 22 通过 / 0 失败   → 合计 198 条
harness.js 提供 createApp()、staticAi()、choiceQ()、subjQ() 等夹具：可控 AI 返回、可预置原生 KV（直接塞 state）、可断言发出的 HTTP 请求。

10. 值得注意的工程细节
主题	做法
不中断刷题	在线/离线双源 + 状态机自愈，AI 慢或报错自动降级，用户只看到卡片上的「离线题库」角标
题目新鲜度	记录已见集合 + genEpoch 代际 + 题干指纹去重；推荐只挑知识点，题目每次现出
可解释推荐	每个被推荐的知识点都带一句 why，刷题页直接显示「为什么推荐它」
划走抑制	skips.pts / skips.subs 累计划走次数，默认推荐据此降权
触感设计	全局事件委托识别控件类型（tap / pick / correct / wrong / warn / done 六档，8–34 ms），FX.recent() 去重避免「震两下」
时间与节奏	「暂时不做」隔两批回归、「划走」更久后进队尾，不打断当前节奏
主题闪屏	启动页 HTML 自带底色，主题在 boot() 最先落地，避免白/黑闪
无后端	Key 只存在本机；导出备份时可选择是否包含 Key
11. 二次开发指引
改界面：src/www/css/app.css 改 token 即可全局换肤；结构在 index.html 的四个 section 里
加一个模块：写成 IIFE，末尾 w.Xxx = Xxx，然后在 index.html 里按依赖顺序加一行 <script>（没有打包器，顺序就是依赖）
加知识点 / 题库：改生成器产出 taxonomy.js、offline-bank.js（这两个文件头部都标了「自动生成，请勿手改」）
加测试：test/*.test.js 里用 ok(name, cond) 断言，section() 分节，跑完打印「结果: N 通过, M 失败」并以退出码表示成败
验收一次改动：node smoke.test.js + node security.test.js（这两套覆盖最广），再 bash build/release.sh <版本> <code> 出包，webbuild.test.js 会自动校验网页版
12. 已知约束（有意为之）
没有后端、也没有内置 Key：不填 API 就只有离线池可用（194 道内置题 + 150 道扩展填空题 + 已缓存的题库）。
外壳是预编译的：原生侧（Java）改动需要重新打外壳；src/ 里只有网页层。
提醒是应用内提醒：到点且当天没达标时，打开或切回 App 才提示一次，不做后台通知（外壳没有通知权限链路）。
数据只在本机：换机靠「设置 → 数据备份」导出 JSON，再在新机导入。
网页版没有原生能力：导出、导入、朗读等会降级或提示不可用，属预期行为。
本文档依据 src/www 源码、build/*.sh、test/*.js 与实测输出整理；版本号以 src/www/js/profile.js 的 VERSION 为准。
