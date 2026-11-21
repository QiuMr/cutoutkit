# 抠图工坊 (CutoutKit)

点选抠图 → 透明 PNG → 图色模板。用于解决传统图色（模板自带背景）一换背景就失效的问题。

界面完全参考 `E:\工作区\yolo-trainer-lab`（Tauri + React）的版式：白底、黑色唯一强调色、
40px 自绘标题栏、236px 可拖拽侧边栏、`max-w 1080` 内容区、Ctrl+K 命令面板。

---

## 它解决什么问题

传统图色用「带背景的 RGB 小图」做模板匹配，屏幕背景一变，相关性直接崩：

| 场景 | 传统图色（TM_CCOEFF_NORMED） | 掩膜匹配（本工具的下一步） |
| --- | --- | --- |
| 原图 | 1.00 | 0.98 |
| 换背景 | 0.24 ~ 0.69 | 0.97 ~ 0.98 |
| 缩放 1.3× | 偏 40px | 误差 0.0px |

所以模板必须先「只剩目标」。这一版工具负责把目标抠成 RGBA，下一版把掩膜匹配算法
做成 32 位 DLL 给易语言调用。

---

## 技术结构

```
cutout-app/
├─ src/                    React 19 + Vite 8 前端（版式与参考项目一致）
│  ├─ components/          TitleBar / Sidebar / CommandPalette / Toaster
│  ├─ pages/               CutPage / LibraryPage / SettingsPage
│  ├─ fonts.css            本地 Inter + Noto Sans SC 的 @font-face（unicode-range 切片）
│  ├─ assets/fonts/        105 个 woff2 子集（4.49 MB，只在需要时才加载对应块）
│  └─ lib/api.ts           与 Rust 的唯一接口面
├─ src-tauri/              Tauri v2 后端（Rust）
│  ├─ src/engine.rs        两段式 ONNX 推理 + 掩膜后处理
│  └─ src/lib.rs           命令层与目录解析
└─ models/                 sam_encoder.onnx (26.7MB) + sam_decoder.onnx (19.6MB)
```

### 为什么把 MobileSAM 拆成两段

TinyViT 编码器是一张图最贵的部分，点选时却要反复解码。拆开后：

- **编码**：每张图只跑一次（长边缩到 1024 + 右/下补零，和 SAM 原生 `ResizeLongestSide` 一致），
  得到 `[1,256,64,64]` embedding 并缓存；
- **解码**：每次点击/改框只跑 MaskDecoder，输入 embedding + 8 个提示槽，输出 3 个候选。

实测（Ryzen 5 5560U，纯 CPU）：编码约 0.8~1.5s，缓存命中 0.3ms，解码约 100ms。

### GPU 加速（DirectML，可选）

同一个 `onnxruntime.dll` 里 **CPU 与 DirectML 两个 EP 都在**，所以切引擎只是重建一次会话，
不换 DLL、不重启进程。DirectML.dll 走 **Windows 自带**（`C:\Windows\System32\directml.dll`，
Win10 1903+ 就有），分发包里不再多塞 18MB。

本机实测（MobileSAM，1024 长边，发布版二进制，两张图）：

| 引擎 | 第 1 张编码 | 第 2 张编码 | 解码器 | 会话加载 |
| --- | --- | --- | --- | --- |
| CPU（Ryzen 5 5560U 8 线程） | 683 ms | 728 ms | 114~135 ms | 338 ms |
| DirectML（Radeon 核显） | **487 ms** | **434 ms（1.6×）** | 106~125 ms | 646 ms |

两边导出的 PNG 逐字节相同（107×99，24183 B），掩膜面积/bbox 完全一致 ——
这就是「加速不能改变结果」的验证方式（`engine.rs` 里有一条 `directml_matches_cpu` 单测在对拍）。

结论：收益只在**每张图跑一次**的编码器上；256×256 的解码器太小，GPU 反而要付调度开销。
所以默认仍是 CPU，设置页里给一个开关，点一下就用 GPU —— 显卡跑不动、驱动太老、
系统 DirectML 版本不够、或者显卡被别的进程吃满时，**自动退回 CPU 并把原因写在设置页**，
不会出现「设了 GPU 就打不开」。

`--engine cpu|dml` 可在命令行里直接指定（自检/排障用），覆盖配置文件里的值。

### 8 个固定槽位是硬约定

`PromptEncoder._embed_points` 里的 `point_embedding[labels == -1] = 0` 是布尔索引赋值，
导出 ONNX 会变成 `GatherND/ScatterND + Where`，onnxruntime 加载时报
`Incompatible dimensions` / `Attempting to broadcast an axis by a dimension other than 1`。

导出脚本 `../cutout/export_onnx.py` 把它换成了等价的 one-hot 乘加，代价是槽位数必须固定：

- 槽 0/1：框的两个角点，`label = 2`（左上）、`3`（右下）
- 其余槽：点，`label = 1`（正）、`0`（负）
- 空槽：`label = -1`，坐标随便填

改模型时这两个契约（输入名、槽位数）不能变，否则 Rust 侧要同步改。

---

## 使用

| 操作 | 效果 |
| --- | --- |
| 左键点 | 加正点（目标在这里） |
| 右键 / Shift+左键 | 加负点（这里不要） |
| 按住左键拖 | 框选 |
| `1` `2` `3` | 切换 3 个候选 |
| `Delete` | 撤掉最后一个提示 |
| `Esc` | 清空全部提示 |
| `Ctrl+S` | 直接存进模板库 |
| `Ctrl+V` | 把剪贴板里的图当输入图（QQ/微信截图后直接按） |
| `Ctrl+↵` | 用刚存的模板去「找图」页试找 |
| `Ctrl+K` | 命令面板 |
| `Ctrl+B` | 折叠侧边栏 |
| `Ctrl+1..4` | 切页面（抠图 / 找图 / 模板库 / 设置） |

「叠加」看边缘是否贴合，「透明」看真正切掉背景后的效果 —— 两者共用同一张 alpha，
所以看到的就是导出的结果。

导出时可选羽化（0~3px）和「裁剪到内容」。存入模板库的文件名即模板 ID。

---

## 设置页：引擎与更新

设置页是「跑得怎么样 + 要不要升级」这两件事的落点，不是配置大杂烩。

| 卡片 | 能做什么 |
| --- | --- |
| 推理引擎 | CPU / GPU 二选一（GPU 走 DirectML）；显示 DirectML 是否可用、上次编码耗时；显卡起不来时在这里写清原因 |
| 分割模型 | 两个 onnx 是否就绪、大小、当前运行时 |
| 版本更新 | 检查 GitHub Releases、显示新版本号与更新说明、一键下载并重启到新版本 |
| 模板库 | 目录路径 + 一键打开 |
| 关于分发 | 程序形态、运行依赖、配置文件位置 |

引擎选择存在 `%APPDATA%\com.cutoutkit.desktop\config.json`（`{"engine":"cpu"|"dml"}`），
放这里而不是程序目录：更新是整目录覆盖，放程序目录会被新版本顺手抹掉。

### 更新怎么做的（绿色版手写更新器）

`tauri-plugin-updater` 只认**安装器**产出的签名包，本项目发的是免安装 zip，接不上。
参考 `D:\Python测试\OCR-Studio`（同为绿色版）的做法，自己走一遍：

1. 拉 `https://github.com/QiuMr/cutoutkit/releases/latest/download/latest.json`
   （`version / notes / url / size / sha256`）；
2. 下载 zip 到 `%TEMP%\cutoutkit-update\`，**先比大小再比 sha256**（下到半截就丢弃，
   绝不拿半截包去覆盖还能用的程序）；
3. 用系统 `C:\Windows\System32\tar.exe -xf` 解压（Win10 1803+ 自带 bsdtar，认 zip，
   所以不用为更新多编一个 zip 依赖）；
4. 把当前 exe **复制一份**到 TEMP 当"更新助手"再拉起（运行中的 exe 不能覆盖自己），
   主进程退出；
5. 助手轮询旧 PID：`OpenProcess` + `QueryFullProcessImageNameW` **校验进程镜像路径**
   （Windows 会复用 PID，只看 PID 消失会傻等到超时），确认真的退出了才覆盖文件，
   然后拉起原来的 exe 路径。

助手在 `tauri::Builder` **之前**分流 —— 否则任何"单实例/托盘"插件都会把它当成第二个实例踢掉，
表现为"点了更新，程序关了，再也没起来"。全程写 `%TEMP%\cutoutkit-update\helper.log`。

仓库还没发过 Release 时，前端会显示"更新源还没有发布记录"而不是报错。

### 发版（把更新链路打通）

已发布：<https://github.com/QiuMr/cutoutkit/releases/tag/v0.2.2>
（前一版 <https://github.com/QiuMr/cutoutkit/releases/tag/v0.2.1>、
更早 <https://github.com/QiuMr/cutoutkit/releases/tag/v0.2.0>）

包的形态就是**整个 CutoutKit 目录压成 zip**，`latest.json` 与 zip 一起上传到 Release：

```powershell
# 1. 打包，资产名用 ASCII（见下面那条坑）
Compress-Archive -Path .\CutoutKit\* -DestinationPath .\CutoutKit-portable-v0.2.0.zip -Force
# 2. 算 sha256（大小和哈希都要，更新器两个都校验）
$h = (Get-FileHash .\CutoutKit-portable-v0.2.0.zip -Algorithm SHA256).Hash.ToLower()
(Get-Item .\CutoutKit-portable-v0.2.0.zip).Length
# 3. 发版
gh release create v0.2.0 .\CutoutKit-portable-v0.2.0.zip .\latest.json `
  -R QiuMr/cutoutkit --target main --title "CutoutKit v0.2.0" --notes "..."
```

`latest.json` 的格式（仓库里 `latest.json` 已经是 v0.2.0 的样子）：

```json
{
  "version": "0.2.0",
  "notes": "1. 设置页可切 GPU（DirectML）加速；2. 设置页支持检查/一键更新",
  "url": "https://github.com/QiuMr/cutoutkit/releases/download/v0.2.0/CutoutKit-portable-v0.2.0.zip",
  "size": 54728811,
  "sha256": "..."
}
```

**坑：资产名必须是 ASCII。** 第一次发版时 zip 叫 `CutoutKit-便携版.zip`，
`gh release create` 上传后 GitHub 上变成了 `CutoutKit-.zip`（中文被吃掉），
于是 `latest.json` 里那条下载地址直接 404。改名（`PATCH /releases/assets/{id}`）
或一开始就用 ASCII 名都能解决，**别想着在 URL 里 percent-encode 中文然后指望它一直有效**。

`url` 用 `releases/download/<tag>/<asset>` 直链。而 `latest.json` 自己走
`releases/latest/download/latest.json`（固定资产路径，永远指向最新一个正式 Release），
所以程序里的检查地址一次写定，发新版本只改 `latest.json` 里的 `version` 与 `url`。

### 三个候选怎么选

SAM 的三个候选是**粒度递增**的三档，实测（1024×576 的背包界面，点在药水瓶上）：

| 候选 | 面积 | 尺寸 | 是什么 |
| --- | --- | --- | --- |
| 1 | 10,503 | 115×123 | 瓶身（不含软木塞） |
| 2 | 12,825 | 116×156 | **整个瓶子** ← 默认选这个 |
| 3 | 45,863 | 217×218 | 瓶子所在的整张卡片底 |

默认落在**面积居中**的那个，也就是用户指的「整个物体」。不按模型给的分值排：
ONNX 里导出的 `iou_prediction` 是未归一化的原始值（会 >1，例如 0.802 / 0.953 / 1.052），
跨图不可比，而且它常常把最大的那张「卡片底」排到第一 —— 按分值选会导出一张带整块背景的模板。
分值仍会随候选回传，只作参考，界面上不拿它当结论。

### 批量抠图

同一类按钮散在几十张截图里同一个位置时，不必逐张点：

1. 在画布上点一下目标（定好提示）；
2. 右下「批量」卡片点「选图」，多选那批截图；
3. 点「开始」——同一套点/框套到每张图，逐张导出到模板库（同名自动加 `-2`、`-3`）。

单张失败不会中断整批，失败的会在控制台日志里列出来。

---

## 找图页（用模板去屏幕里找）

这是把 matcher 算法变成「能看见」的一页：命中框直接画在截图上，右边给分数和坐标，
拖阈值立刻重算 —— 不用「改参数 → 跑脚本 → 看日志」那一圈。

| 控件 | 说明 |
| --- | --- |
| 模板 | 从模板库下拉选，或「打开 PNG」直接选一张透明图；抠图页存的模板会自动带过来 |
| 打开场景图 | 拿一张现成截图当搜索范围 |
| 准星截屏（主按钮） | 打开时先把桌面截成一张快照当底图，**拖框选一块**（松手即得），或者**单击选一个窗口**（鼠标指着的窗口会有黑框提示），Esc / 右键取消 |
| ▾ 下拉 | 「截全屏（多显示器合并）」；或直接从窗口列表点一个（最小化窗口会自动还原一下再截回原状） |
| 最低分 | 拖动即时重算。≥0.95 直接点，0.75~0.95 建议复核，<0.6 别用 |
| 快速模式 | 开 = 只先试原尺寸档（1080p 约 10~26ms）；关 = 六档全扫（约 60~90ms），不漏缩放实例 |
| 复制易语言代码 | 生成带模板路径、阈值、坐标的调用片段，直接粘进易语言 |

命中框：第一个命中是粗黑框 + 分数标签，其余是细灰框；点结果行复制中心坐标。

### 截图：全屏 / 窗口 / 框选，统一进内存（v0.2.2）

三种来源走同一个捕获层（`src-tauri/src/capture.rs` 的 `Target::{Screen, Window, Region}`），
出口都是内存里的 RGBA —— **没有临时文件**。

为什么非要"不落盘"：以前的路线是「截图 → 存 `%TEMP%\CutoutKit\shot-xxx.png` → 再读回来找图」，
多一次磁盘往返，还留下了"清理软件把临时图扫了、页面上还拿着旧路径"的坑（v0.2.1 那条
「读图失败 E:\...」就是这么来的）。现在截图直接进内存的 `ShotStore`（留最近 4 张），
拖阈值重算只发一个图片 id，不再重新截屏，也不怕临时目录被清。

窗口截图的两条硬经验：

- **`PrintWindow(PW_RENDERFULLCONTENT=2)`**：对 WebView2 / Chromium / Electron 这类窗口，
  即使被别的窗口挡住、甚至不可见，也能拿到它自己的完整渲染；窗口矩形用
  `DwmGetWindowAttribute(DWMWA_EXTENDED_FRAME_BOUNDS)` 取，比 `GetWindowRect` 准
  （后者把外面的阴影和圆角透明边一起算进去，截出来四边各多一圈白）。
- **最小化窗口会返回全黑**：抽样 2000 个点判"几乎全黑"后走兜底 —— `ShowWindow(SW_RESTORE)`
  + 置顶 + 等 220ms + `BitBlt` 抓那块矩形 + 还原成最小化。用户看到的是"闪一下"，拿到的是对的图。

准星那层不快照就不行：Windows + WebView2 的**真透明窗口在这台机器上做不到**
（`transparent(true)` + `background_color(0,0,0,0)` + CSS 全透明都试过，整屏被那层白底糊成
均匀灰板，只有提示条能正常渲染）。所以改成「打开准星那一刻先把桌面截成快照当底图」，
上面压 26% 黑，拖框时恢复亮度、框外继续压暗；**区域直接从这张快照裁**（坐标就是虚拟桌面坐标，
底图原点即虚拟桌面原点），好处是框选期间画面不闪、框到的百分百就是用户看到的那块。

坐标一律物理像素：捕获前把线程切到 `PER_MONITOR_AWARE_V2`，准星窗口按物理像素铺满整个虚拟桌面，
前端量到的是逻辑像素，所以乘 `devicePixelRatio` 再叠虚拟桌面原点。150% 缩放下也不会偏。

顺手修掉的三个坑（都会让"看起来很简单"的功能直接不可用）：

| 现象 | 原因 | 做法 |
| --- | --- | --- |
| 快照里有个半透明的自己 | Win11 隐藏窗口有淡出合成，截太快把正在消失的主窗口糊进去了 | 隐藏后等 420ms 再截 |
| 从界面点「准星截屏」窗口停在 800×600、里面全黑、右键没反应 | 从 IPC 线程直接建 WebView2 窗口，`set_position/set_size/show` 全部静默失效 | 整个开窗过程回主线程跑，几何交给 builder 一次定好（builder 只认逻辑像素，按所在显示器缩放换算） |
| 鼠标指着窗口却不高亮 | `WindowFromPoint` 命中的永远是铺满全屏的准星自己，再被"跳过自己"过滤成 None | 改成从 z 序顶端往下逐个命中测试（跳过自己 / 桌面 / 任务栏 / cloaked 窗口） |

在抠图页点了准星也没关系：截完会自动切到找图页，并把那张图一起带过去（主窗口自己监听
`shot://ready`，找图页不在前台时由 App 转交）。

### 文件不在了会说人话（v0.2.1）

找图页会记住「这一页用过的模板 / 场景图」路径，但**只记在页面内存里**，关掉软件就清空，
`%APPDATA%\com.cutoutkit.desktop\config.json` 里只有引擎选择和耗时，没有任何图片路径。
所以不存在「写死了某个盘」这回事；真正会发生的是：**那个文件后来被移动、改名、删掉**，
而页面上还拿着旧路径。（v0.2.2 之后截屏不再写临时文件，所以"临时图被清理"这类失效来源整个消失了。）

以前的报错是 `读图失败 E:\...\面板.png：系统找不到指定的文件。(os error 2)` —— 一串原生
错误码 + 一个用户没法处理的路径。现在按角色分开说：

| 情况 | 界面上的说法 | 顺手做了什么 |
| --- | --- | --- |
| 场景图被移走/删除 | 「场景图不在了」+ 原路径 + 重新打开一张 / 截屏找图 | 清掉失效场景图，不再拿旧路径重试（拖阈值不会连环弹错） |
| 截图过期（只留最近 4 张，或重开过软件） | 「那张截图已经过期了，重新截一次」 | 内存截图没有"文件"，这条只剩"截得太多"一种来源 |
| 模板被移走/删除 | 「模板文件不在了」+ 从模板库重选 | 清掉失效模板，回到「从模板库选择」 |
| 后缀被改过（jpg 存成 .png） | 「不是能识别的图片，可能只是改了扩展名」 | — |
| 选中的是个文件夹 / 没权限读 | 各自单独提示 | — |

右侧参数卡多了一行「场景文件」，显示当前场景的文件名（鼠标悬停看全路径），
排查时一眼能看出它在用哪张图。


---

## 这一版修掉的两个坑（都影响「点哪抠哪」）

1. **掩膜回映射压扁了 y 方向。** 早期为了省一步，把 256×256 的低分辨率掩膜直接缩放成
   `nw×nh`，等于把整块补零画布压进了有效区：1024×576 的图上 y 被压到 0.5625 倍，
   点 (835,260) 抠出来的是上方 100px 处的卡片顶边。正确顺序是
   256 → 1024 画布 → **裁掉右下补零区**（取左上 `nw×nh`）→ 缩回原图。
   修完后 bbox 从 `[772,97,888,194]` 变成 `[772,189,888,345]`，正好贴着瓶子。
2. **点开图片这条路上有个变量名写错**（`setMaskRef` → `maskRef`），抛出的 ReferenceError
   被启动链路的 `catch` 静默吞掉，表现是「图打开了，但点什么都没反应」。
   现在启动链路的错误会弹 toast，不再静默。

还有一个纯 UI 的修复：候选缩略图之前传的是整图尺寸的 alpha，塞进 34px 的方格里
物体只剩几个像素，看着像一块空棋格。现在按 bbox 单独裁一份（`thumb` 字段），
并且把底色压深 —— mask 是「白=物体、透明=背景」，白形状要有深色底才看得清。

---

## 分发

便携版 = 一个目录，拷走即用，不需要 Python / torch / CUDA：

```
CutoutKit/
├─ CutoutKit.exe                    11.9 MB  （Tauri 壳 + 内嵌前端/字体 + 匹配算法 + 更新器）
├─ onnxruntime.dll                  16.5 MB  （ONNX Runtime 1.24.3，含 CPU + DirectML 两个 EP）
├─ onnxruntime_providers_shared.dll  22 KB   （ORT 的 provider 共享库）
└─ models/
   ├─ sam_encoder.onnx              26.7 MB
   └─ sam_decoder.onnx              19.6 MB
```

解压后约 78 MB（含模型），压成 zip 约 52 MB。`ORT_DYLIB_PATH` 会自动指向程序目录里的 DLL，
避免加载到 system32 里的旧版本。

两点和上一版的差别：

- **DLL 从 15.4 MB 涨到 16.5 MB（+1.2 MB）**：换成了带 DirectML 的那份。
  它里面 **CPU EP 仍然在**，所以不必为「GPU 版 / CPU 版」打两个包 —— 一个 DLL 两个引擎，
  运行期按设置决定是否挂 DML。省下的是「再发一个 16 MB 的 GPU 专用包」。
- **没有额外塞 DirectML.dll（18.5 MB）**：Windows 10 1903+ 的 `System32` 里自带
  （本机是 1.15.6，和 nupkg 里的 1.15.4 同一代）。少 18 MB，而且不会盖掉系统后续更新的版本。

exe 涨了约 1.8 MB（reqwest + rustls + sha2 + windows crate 的更新助手）。

找图用的是 **同一份 Rust 算法源码**（`matcher` crate）直接编进 exe，不走 DLL：
所以「找图」页里看到的分数/坐标，和易语言调 `Matcher32.dll` 得到的是逐位一致的结果。

### 字体（和参考项目同一套）

参考项目用的是 Google Fonts 上的 **Inter + Noto Sans SC**，并且是本地自带（离线、不受墙影响）。
这一版直接把同一套字体搬过来，`src/fonts.css` 里按 `unicode-range` 切片声明，
字体栈是 `"Inter", "Noto Sans SC", ...`：拉丁字母与数字走 Inter，中文走 Noto Sans SC。

原来的版本字体栈只写了 `Microsoft YaHei UI`（系统雅黑），所以整屏是雅黑 —— 这就是"看着不像"的原因。

另外参考项目那份字体目录有个问题：它是把**一份可变字体**按 400/500/600/700/800 存了 5 遍
（同名文件字节完全相同，237 个文件里只有 105 份不同内容，9.63 MB），
`@font-face` 里还把 `font-weight` 写死成单个值，等于把 wght 轴锁住。
这一版用 `scripts/build_fonts.py` 处理成：每个切片只留一份、`font-weight: 100 900`，
真走可变字体的 wght 轴 —— 400/700 都是真字重，体积 4.49 MB（省掉一半）。

### 命令行（批量/脚本/自检）

```
CutoutKit.exe "D:\截图\ui1.png"                  # 直接开这张图，等人点
CutoutKit.exe "D:\截图\ui1.png" --point 835,260  # 打开后自动用这个点试一次
CutoutKit.exe --page match                        # 直接开在「找图」页
CutoutKit.exe --try "D:\模板\药水瓶.png"          # 开在找图页 + 立刻截屏找一次
CutoutKit.exe --engine dml --selftest "ui1.png"   # 用 GPU 跑自检（排障用）
CutoutKit.exe --selftest "ui1.png" "ui2.png"      # 无窗口批量抠图（多张=批量）
CutoutKit.exe --findtest "D:\模板\药水瓶.png"     # 无窗口找图，结果写 findtest.log
CutoutKit.exe --overlay                           # 启动就直接开准星（快捷方式/排障用）
CutoutKit.exe --shot D:\out\屏幕.png              # 截全屏存一张（写 shot.log）
CutoutKit.exe --shot D:\out\窗口.png --hwnd 12345 # 截指定窗口句柄
CutoutKit.exe --shot D:\out\区域.png --rect 300,100,400,260   # 截一块区域
CutoutKit.exe --shotlist                          # 把可截的窗口列进 shotlist.log
```

`--engine cpu|dml` 覆盖配置文件里的引擎选择，只影响这次运行，不会写回配置。
`--selftest` 的报告里会有一行 `请求引擎 dml · 实际引擎 dml`，一眼看出 GPU 是真起了还是退了。

`--selftest` 不建窗口，只跑「加载 DLL → 载模型 → 编码 → 解码 → 导出」，
结果写进 exe 旁边的 `selftest.log`。给多张图就是命令行批量抠图，每张导出 `<原名>-selftest.png`：

```
加载模型 349ms
图片 2 张
[0] ...ui1.png 编码 687ms · 解码 132ms · 候选 3 · 导出 204x176 82160B → ...ui1-selftest.png
[1] ...ui2.png 编码 740ms · 解码 209ms · 候选 3 · 导出 962x641 842378B → ...ui2-selftest.png
完成 2 / 失败 0
```

`--findtest` 也是无窗口：截全屏 → 用给的模板找一遍 → 把命中坐标/分数/耗时写 `findtest.log`，
同时把这次截屏留成 PNG（路径写在日志里）。做回归测试或排查「为什么线上找不到」很方便。

`--shot` / `--shotlist` 是截图能力的命令行入口，**只在排障和脚本里用**（正常使用时截图不落盘）：
`--shot out.png` 默认截整个虚拟桌面，`--hwnd` 换成某个窗口、`--rect x,y,w,h` 换成一块区域，
尺寸/耗时/是否走了黑图兜底都写进 exe 旁边的 `shot.log`；`--shotlist` 把可截的窗口
（含标题、进程、尺寸、是否最小化）列进 `shotlist.log`，用它取 `--hwnd` 的值。

```txt
模板 ...\crop_tpl2.png
场景 1920x1080 建表 14.6ms 搜索 94.5ms 命中 3
  (1200.01,530.01) score=0.9902 scale=1.00 120x200
```

排查启动链路时设 `CUTOUTKIT_TRACE=1`，会把启动参数与关键命令调用追加到 `trace.log`。

也可 `npm run tauri build` 出 NSIS 安装包（模型作为 resources 打进安装目录）。

---

## 已知取舍

- 窗口是无边框自绘标题栏：白画布不被 32px 系统灰条压住，代价是没有 DWM 阴影边。
- 不做「一劳永逸」：模型不会自动适配所有 UI 风格。可持续的做法是资产化 ——
  每个目标 = 一张 RGBA 模板 + 建议阈值，进版本库；分数掉下阈值说明 UI 改版，重新抠一次。
- MobileSAM 在扁平卡通 + 强描边这类素材上已经饱和，换 SAM2/ViT-B 收益很小，
  但框提示在 ultralytics 的 SAM2 里是坏的（`SAM2Predictor._inference_features` 把 `boxes` 写死成 None）。

## 32 位 DLL 用 Rust 还是 C++

**已交付**：`Matcher32.dll`（382 KB，stdcall 裸名，易语言直接可调）/ `Matcher64.dll` 在 `发布\MatcherDLL\`，
和本工具里「找图」页跑的是同一份源码。下面是当初选型时的环境记录，留档用。

结论：**Rust 写，导出成 32 位 MSVC DLL**。这台机器上两条路都通，选 Rust 是因为能「一份算法两处用」，
不是因为性能。

先说环境事实（都是本机实测）：

| 事实 | 结果 |
| --- | --- |
| `rustup target list --installed` | 原本只有 `x86_64-pc-windows-msvc` |
| `rustup target add i686-pc-windows-msvc` | 已加好，`rust-std` 下载正常 |
| VS 2022 Community | 已装，`VC\Tools\MSVC\{14.29,14.43}\lib\x86` 都在 |
| 32 位 linker | `bin\Hostx64\x86\link.exe` 存在，`cl.exe` 不在 PATH 但可 vcvars 引入 |
| MinGW | `D:\mingw64` 是 `x86_64-w64-mingw32`，**没有** i686，帮不上 32 位 |

选 Rust 的理由：

1. **算法只写一遍。** 掩膜加权 NCC + 梯度域 + 多尺度 + 亚像素 + NMS 做成一个独立 crate，
   `x86_64` 编进现在这个 GUI（模板库里点一下就能自测匹配分），`i686` 编成 DLL 给易语言。
   C++ 就得维护第二份实现，两边慢慢漂移 —— 这类数值算法最怕两份实现不一致。
2. **FFI 面可以做到极小，安全性反而更好。** 对外只留 4 个函数
   （`create / destroy / find / free_result`），模板与截图都是裸指针 + 宽高 + 步长，
   Rust 侧把 `unsafe` 收在一个模块里、其余全是安全代码，越界读写在调试构建里直接崩，
   不会像 C++ 那样悄悄读坏内存给出一个"看着正常"的分数。
3. **交叉编译就是一条命令**，不需要维护 `.sln/.vcxproj`，也不用担心 CRT 版本。

C++ 更合适的场合：如果这个 DLL 将来还要被别的原生模块（比如游戏内注入、DirectX 钩子）
链接使用，或者你希望别人 clone 下来用 VS 就能改，那 MSVC + `/MT` 静态 CRT 的 C++ 更"原生"，
体积也更小（纯 SSE2 的 NCC 大概 20~60KB）。

两个和易语言对接时必须提前定死的事：

- **符号名。** 32 位 stdcall 的导出名会被修饰成 `_find@24`。要拿到易语言能直接按名字调的
  裸名，编译时挂一个 `.def`：`find=_find@24`（Rust 用 `-C link-arg=/DEF:...`）。
  或者干脆用 cdecl（`extern "C"`，不修饰），代价是易语言侧的调用约定要跟着改。
- **不要跨 FFI 展开 panic。** 所有导出函数外面套 `catch_unwind`，兜底 `abort`；
  返回码用 `i32`，结果放调用方传进来的结构体，不让 Rust 分配的内存跨边界释放。

算法蓝本已经在 `../cutout/matcher.py` 里验证过（7 个合成场景全部命中、亚像素误差 0.0px），
DLL 就是把它翻译过去 + 加 SIMD（SSE2 保底、AVX2 运行时探测）+ 多线程分块。

阈值沿用 `../cutout/README.md` 的实测表：≥0.95 干净命中；0.75~0.95 可接受需复核；
0.6~0.75 遮挡/形变；<0.6 不要。

