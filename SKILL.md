---
name: overleaf-sync
description: >
  Read, edit and two-way sync an Overleaf LaTeX project from the local filesystem
  using the overleaf-sync CLI (ols), without paid Overleaf, Git bridge or browser
  automation. Use when the user names an Overleaf project or shares a public
  read-only link, asks to view / edit / sync / push changes to an Overleaf
  project, wants LaTeX compile errors found and fixed for an Overleaf paper, or
  when operating the Overleaf web editor is not possible.
  Collaborator invite links are not readable anonymously; use the account
  cookie instead. Triggers: overleaf, overleaf 同步, 打开我的 overleaf,
  改论文, 推回 overleaf, .tex 项目协作.
---

# Overleaf 本地双向同步（overleaf-sync）

## 先明确边界

Claude Code 会话通常**没有**操作外部网站的能力：应用内 Browser 预览只服务
`localhost` 开发服务器，挂外部域名会被拒（`external URL preview is not enabled`）。
因此：

- 不要承诺"帮你点 Overleaf 的界面"。
- 不要试图用 cookie 去跑浏览器自动化。
- 正确做法是 `overleaf-sync`：走 Overleaf 自己的 Web 接口做**文件级**双向同步。

`overleaf-sync` 是第三方 MIT 工具（PyPI 包名 `overleaf-sync`，命令名 `ols`），
支持免费账号，**不接触密码**——登录在 Overleaf 自己的页面上完成，只保留会话 cookie。
它不是官方功能，Overleaf 改接口时可能失效。

## 一、安装

要求 Python ≥ 3.11。**必须用独立 venv**，不要装进 conda base。

```bash
python -m venv <VENV_DIR>
<VENV_DIR>/Scripts/python -m pip install --upgrade pip
<VENV_DIR>/Scripts/python -m pip install "overleaf-sync[gui]"
<VENV_DIR>/Scripts/ols --help
```

`[gui]` 会拉 PySide6（约 250 MB）。如果只想用 cookie 登录，装 `overleaf-sync`
不带 `[gui]` 即可，体积小很多。

本机有 `uv` 时更省事，还能拿到**非 conda** 的 Python，顺带绕开下面那个 Qt 陷阱：

```bash
uv venv <VENV_DIR> --python 3.12
uv pip install --python <VENV_DIR>/Scripts/python.exe "overleaf-sync[gui]"
```

装完**不要假设 `ols` 在 PATH 上**——工具 shell 的 PATH 可能在进程启动时就冻住了，
新装的命令会报 `command not found`。一律用绝对路径调用：
`<VENV_DIR>/Scripts/ols`（Linux/macOS 是 `<VENV_DIR>/bin/ols`）。

## 二、登录（关键陷阱）

### 陷阱：图形登录窗在 conda 环境里加载失败

如果 venv 基于 Anaconda 的 Python，`ols login` 会报：

```
Error: The login window could not be loaded: DLL load failed while importing QtCore
This usually means another Qt installation (e.g. from Anaconda) conflicts with PySide6.
```

清理 PATH **不能**解决。直接改用 cookie 登录，不要在这上面耗时间。

### cookie 登录

1. 浏览器登录 `https://www.overleaf.com`（自建服务器换成自己的域名）。
2. 打开开发者工具：
   - Chrome / Edge：Application → Storage → Cookies → `https://www.overleaf.com`
   - Firefox：Storage → Cookies → `https://www.overleaf.com`
3. 复制 `overleaf_session2` 的值（自建 Overleaf 是 `overleaf.sid`；5.0 以前的
   版本叫 `sharelatex.sid`）。
4. 执行：

```bash
ols login --cookie --auth-file <SYNC_DIR>/.olauth
```

在隐藏提示处粘贴，回车。屏幕上不显示任何字符是正常的。非交互场景可以先
`export OLSYNC_COOKIE=<值>`，再跑同一条命令。

成功输出：`Login successful. Cookie persisted as ...`

### 自建 Overleaf

把服务地址在登录时一并交进去，之后所有命令都会记住它：

```bash
ols login --server https://overleaf.example.com --auth-file <SYNC_DIR>/.olauth
ols list --auth-file <SYNC_DIR>/.olauth          # 不用再重复 --server
```

`OLSYNC_SERVER` 环境变量可以代替 `--server`。

### `.olauth` 的版本

`.olauth` 格式在 overleaf-sync 2.0 变更过。升级后会报
`was created by an older version of overleaf-sync`，重新登录即可，别去手工改文件。

## 三、日常命令

### 在哪个目录执行（最容易踩的坑）

同步的本地根目录由**当前目录**决定，不由 `-n` 决定——`-n` 只负责在账号里挑项目。
两种写法等价，二选一，别混着用：

```bash
cd <SYNC_DIR> && ols -n "$NAME" --auth-file ./.olauth          # 写法一：cd 进去
ols -p <SYNC_DIR> -n "$NAME" --auth-file <SYNC_DIR>/.olauth    # 写法二：-p 指定
```

在别处裸跑 `ols -n "论文"` 会把**整个当前目录**当成同步根往 Overleaf 上传，包括
你根本不想传的东西。`-p/--path` 就是为这种情况准备的。

### 命令

```bash
OLS=<VENV_DIR>/Scripts/ols        # Windows；Linux/macOS 为 <VENV_DIR>/bin/ols
AUTH=<SYNC_DIR>/.olauth
NAME="<EXACT PROJECT NAME>"

$OLS list --auth-file $AUTH                      # 列出账号下项目（最近修改在前）
$OLS -r -n "$NAME" --auth-file $AUTH             # 首次：只从远端下载
$OLS -l -n "$NAME" --auth-file $AUTH             # 改完：只把本地推到远端
$OLS    -n "$NAME" --auth-file $AUTH             # 双向，双侧都改过会逐文件询问
$OLS download -n "$NAME" --auth-file $AUTH --download-path <OUT_DIR>   # 云端编译 + 下载 PDF
```

规则：

- **项目按名称匹配**。本地目录名与项目名不同时，必须 `-n "<全名>"`，包括
  `(2)`、空格、`^` 这类后缀和字符。
- **第一次永远先 `-r`**，让空目录从远端建立基线。
- `-l` / `-r` 只覆盖"另一侧自上次同步后没动过"的文件；两侧都动过的会询问。
- 双向 `ols` 不会删除文件；删除要走 `-l`/`-r` 并确认。
- **`ols download` 是本地没有 LaTeX 时的唯一编译验证手段**：它在 Overleaf
  云端编译一次并取回 PDF。用 PDF 页数与本地读取内容核对产物是否真的更新。
- 出任何异常，先加 `-v/--verbose` 再跑一遍。默认输出会把"为什么失败"藏起来。

## 四、安全纪律

| 文件 | 性质 | 处理 |
|---|---|---|
| `.olauth` | 会话 cookie = 账号临时访问权 | 不提交、不粘贴、不打印。失效用 Overleaf 账号设置里的 Clear sessions |
| `.olsync-state.json` | 上次同步快照，决定哪边改了 | 别删。删了下一次会把所有差异当冲突逐条询问 |
| `.olignore` | 上传排除清单（fnmatch 语法） | 用来挡 `*.log`、`out/*` 这类构建产物 |

- 以 `.` 开头的文件和 `.git` **永远不会**被上传。
- 同步目录里的**一切非隐藏文件都会被推到 Overleaf**。分析脚本、评审笔记、
  临时导出的 PDF 一律放在同步目录**外面**；确需放里面就先写进 `.olignore`。

## 五、排错

| 现象 | 原因 | 处理 |
|---|---|---|
| `403 Restricted` / `Your Overleaf session has expired` | 未登录 / 会话过期 / 无权限 | 重新 cookie 登录 |
| `/read/<token>` 返回 404 | 该链接不是公开只读链接，是邀请链接 | 别试匿名访问，走 cookie 登录 |
| `DLL load failed ... QtCore` | conda 与 PySide6 的 Qt 冲突 | 用 `--cookie` 登录，或改用非 conda 的 venv |
| `Project could not be queried` / `Project not found` | `-n` 与项目名不一致 | `ols list` 核对全名 |
| `Multiple active projects are named ...` | 账号里有同名项目，工具不替你猜 | 在 Overleaf 里改名或归档其中一个 |
| 首次同步逐文件询问 | **没有同步历史**，不是故障 | 逐条确认；想要静默基线就保证本地为空再 `-r` |
| 删掉 `.olsync-state.json` 后仍逐条询问 | 历史已丢，`-r` 一样会问 | 删除没有好处，别删 |
| `Overleaf's rate limit was reached` | 短时间内请求过多 | 等一分钟；工具已自动重试 |
| 刚在网页改的内容没同步过来 | Overleaf 尚未保存完 | 等几秒再跑一次 |
| 推上去了不该传的文件 | 同步目录有杂物 | 云端删除 + 写 `.olignore` |
| `ols` 命令名被占用 / `no description file specified` | 环境里有同名程序 | 改用 `overleaf-sync`，二者等价 |
| 其他查不出原因 | 默认输出不显示细节 | 加 `-v/--verbose`，把输出原样报告 |

## 六、替代方案

- **Overleaf Premium**：`git clone https://git.overleaf.com/<PROJECT_ID>`，
  凭据来自 Account Settings → Git Integration。不支持分支、tag、子模块、LFS。
  `git push` 即同步，适合长期项目。
- **无凭据**：让用户下载 Source ZIP 或 PDF，手工往返。这是最后手段，别当默认。

## 七、验证清单

首次接入一个项目时，按顺序确认：

0. 将要上传的目录里**没有**无关文件——`ls -a` 看一遍。这一步做漏了，后面很难收拾。
1. `ols list` 能看到目标项目，名称与 `-n` 一致。
2. `ols -r` 后本地文件数与远端目录树一致，主 `.tex` 存在。
3. 抽查主文件的标题/摘要与用户所述一致，确认没拉错项目。
4. `ols download` 成功，PDF 页数合理（与云端编译一致）。
5. 第一次真实修改后跑 `ols -l`，核对输出里列出的文件名**正是**你改的那几个。