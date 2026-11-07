# overleaf-sync-skill

一个 Claude Code skill：让 Claude 直接读写并双向同步你的 Overleaf 项目，
不依赖浏览器自动化、不需要付费 Overleaf 或 Git bridge。

内容就是这次实机跑通后总结的操作手册，包括登录陷阱、命令清单、安全纪律和排错表。

## 它解决什么问题

Claude Code 的会话通常没有操作外部网站的能力：应用内浏览器预览只服务本地开发
服务器，挂 `overleaf.com` 会被拒绝。于是"帮我改 Overleaf 上的论文"就没有下手点。

[overleaf-sync](https://github.com/moritzgloeckl/overleaf-sync)（MIT）用 Overleaf
自己的 Web 接口做文件级双向同步。它在 Overleaf 的登录页上完成登录，**不接触密码**，
只保存会话 cookie。免费账号可用。

这个 skill 把它的安装、登录、日常命令、冲突语义、安全边界和常见故障整理成一份
可直接执行的流程。

## 安装

### 1. 装工具

```bash
python -m venv ~/.venvs/ols
~/.venvs/ols/bin/python -m pip install "overleaf-sync[gui]"
```

Windows 把 `bin/python` 换成 `Scripts/python.exe`，命令是 `ols.exe`。

有 `uv` 的话用它建 venv 更好：能选一个**非 conda** 的 Python，直接绕开下面的陷阱。

```bash
uv venv ~/.venvs/ols --python 3.12
uv pip install --python ~/.venvs/ols/Scripts/python.exe "overleaf-sync[gui]"
```

> **已知陷阱**：venv 如果基于 Anaconda 的 Python，图形登录窗会报
> `DLL load failed while importing QtCore`。清理 PATH 无效，改用 `ols login --cookie`。

### 2. 装 skill

把 `SKILL.md` 复制到 Claude 的用户级 skill 目录：

```bash
mkdir -p ~/.claude/skills/overleaf-sync
cp SKILL.md ~/.claude/skills/overleaf-sync/SKILL.md
```

Windows 对应 `C:\Users\<you>\.claude\skills\overleaf-sync\SKILL.md`。

之后在会话里提到 Overleaf、论文同步、`.tex` 项目时会被自动选用。

## 最小用法

```bash
cd <同步目录>                                    # 同步根是「当前目录」，不是 -n
ols login --cookie --auth-file ./.olauth        # 粘贴浏览器里的 overleaf_session2
ols list --auth-file ./.olauth                  # 找到项目全名
ols -r -n "<项目全名>" --auth-file ./.olauth    # 首次：拉下来
# 在本地改 .tex
ols -l -n "<项目全名>" --auth-file ./.olauth    # 推回 Overleaf
ols download -n "<项目全名>" --auth-file ./.olauth --download-path ./out   # 云端编译验证
```

不在同步目录里时用 `-p <同步目录>` 代替 `cd`。自建 Overleaf 在登录时加
`--server https://你的域名`，之后所有命令都会记住。查不出原因就加 `-v`。

## 四条必须记住的规则

1. `.olauth` 是账号会话凭据，别提交、别分享。作废：Overleaf → Account Settings → Clear sessions。
2. 同步目录里的所有非隐藏文件都会上传。脚本、笔记、日志放目录外面，或者写进 `.olignore`。
3. 项目按**名称**匹配，首次同步先 `-r`。
4. 同步根由**当前目录**决定；不在同步目录里时必须显式 `-p`，否则会把当前目录整个传上去。
5. 长命令要有时间上限和输出上限。`ols download`、批量同步都可能跑很久，
   命令本身不带超时——详见 [LONG-RUNNING-COMMANDS.md](LONG-RUNNING-COMMANDS.md)。

## 长命令

`ols download` 要在 Overleaf 云端编译，大项目会慢；批量同步同理。这些命令**没有自带上限**，
跑起来就一直跑。一次真实事故里，一个没有终止条件的管道任务在前台超时后被自动转后台，
停止工具报成功但子进程继续写，75 分钟后写掉了约 98 GiB。

建议的形状：

```bash
timeout -k 5 300 ols download -n "$NAME" --auth-file "$AUTH" --download-path ./out
```

退出码 124 表示超时被杀。被强制终止之后，先 `ols list` 核对项目、再 `-r` 回一次远端基线，
不要直接接着改——中途被杀可能留下写了一半的文件。

完整的三条规则和 Claude Code 的相关行为见 [LONG-RUNNING-COMMANDS.md](LONG-RUNNING-COMMANDS.md)。

## 边界

- `overleaf-sync` 是第三方工具，靠模拟网页接口工作，Overleaf 改接口时可能失效。
- Overleaf Premium 用户可以用官方 Git bridge，长期项目更稳。
- 本仓库只包含操作文档，不包含 `overleaf-sync` 的源码，也不复制其许可证内容。

## 许可

本 skill 文档按 MIT 使用。`overleaf-sync` 本身是 MIT，版权归 Moritz Glöckl；
使用前请以其仓库为准。