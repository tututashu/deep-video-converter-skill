# Deep Video Converter Tool · Deploy & Operate Skill

**"深度视频转换器"类工具的通用部署/运维 Agent Skill** —— 让 AI 助手能一键完成这类工具的定位、环境搭建、启动、五种特效模式的使用、免浏览器验证与故障排查。

> 本技能是**通用模式**，不绑定任何特定 GitHub 仓库/主项目。目标项目由使用者提供（自己的 clone、fork 或已有目录），skill 按约定结构识别并操作。

## 这是什么

本仓库遵循 **Agent Skills（SKILL.md）开放标准**（顶层 `SKILL.md` + `references/`），凡是支持该标准的 agent 都能直接识别使用。已兼容：

- **Claude Code** → `~/.claude/skills/`
- **OpenAI Codex CLI** → `~/.codex/skills/`
- **WorkBuddy** → `~/.workbuddy/skills/`
- 其他 SKILL.md 标准实现（Cursor、OpenClaw 等）

### 操作对象的结构约定

skill 面向满足以下布局的"深度视频转换器"项目：

```
<你的项目>/
├── backend/app.py + pipeline.py + face_worker.py + config.py   # Flask + 双环境编排
├── frontend/index.html                                          # 单页 UI
├── scripts/setup.sh|bat  run.sh|bat  run_e2e_all.py  recomposite_test.py
├── models/            # 权重（通常不入库，setup 下载）
├── venvs/env-main     # torch/transformers/opencv/flask（py3.12）
├── venvs/env-face     # mediapipe（py3.11，进程隔离）
└── jobs/              # 每任务中间帧与结果
```

skill 触发后按流程执行：**定位项目 → 一键 setup（双 venv + 模型）→ 启动 `http://localhost:8000` → 五种模式使用 → 免浏览器验证 → 排障**。

## 安装

按所用 agent 选择用户级技能目录（所有项目通用）：

```bash
# Claude Code
git clone https://github.com/tututashu/deep-video-converter-skill.git ~/.claude/skills/deep-video-converter

# Codex CLI
git clone https://github.com/tututashu/deep-video-converter-skill.git ~/.codex/skills/deep-video-converter

# WorkBuddy
git clone https://github.com/tututashu/deep-video-converter-skill.git ~/.workbuddy/skills/deep-video-converter
```

或从 [Releases](https://github.com/tututashu/deep-video-converter-skill/releases) 下载 `deep-video-converter.zip`，解压后把 `deep-video-converter/` 文件夹放到上述任一技能目录。

### 使用方式

1. 打开 agent 会话，进入你的 deep-video-converter 项目目录（或把路径告诉 agent）
2. 提问触发，例如："启动深度视频转换器"、"帮我把这个项目跑起来"、"跑一下 CLI 端到端验证"、"人脸点云是空的，查一下"
3. agent 自动加载本 skill：先做**环境预检**（Python 3.11/3.12、ffmpeg），再按结构定位你的项目并执行对应流程

**没有代码时的一键安装**：若你手头没有该项目代码，skill 内置一个**可配置默认源**（`git clone https://github.com/tututashu/deep-video-converter.git`），agent 会用它自动克隆后 setup。你可以在对话中指定自己的仓库/fork 地址覆盖它；维护者 fork 本 skill 后，把 `SKILL.md`「Locate or obtain the project」一节的默认源 URL 换成自己的仓库，即可变成你的专属一键安装技能。

## 前置条件（目标项目运行本身需要）

- Python 3.11（人脸环境）+ Python 3.12 / 3.10+（主环境）
- ffmpeg 在 PATH
- macOS / Linux / Windows x64
- 首次 setup 需联网下载模型（数百 MB），之后完全离线可用

**缺失时可自动安装**，两条路径任选：

1. **skill/agent 路径**：触发本 skill 后自动做环境预检，缺失时询问你；选择"自动安装"后 agent 按系统执行（macOS `brew`、Windows `winget`、Linux `apt`），装完重跑预检。
2. **纯脚本路径**（无 agent）：`setup.sh` / `setup.bat` 检测到缺失会打印安装指引；设 `AUTO_INSTALL_PREREQS=1` 后重跑即自动安装（macOS/Homebrew 与 Windows/winget 免 sudo 全自动，Linux 需免密 sudo）：

   ```bash
   AUTO_INSTALL_PREREQS=1 bash scripts/setup.sh     # Mac / Linux
   set AUTO_INSTALL_PREREQS=1 && scripts\setup.bat  # Windows
   ```

   > 模型下载同样自动降级：官方源 → `MIRROR_MEDIAPIPE`/`MIRROR_PYTORCH` 镜像 → GitHub Releases 分卷（[models-v1](https://github.com/tututashu/deep-video-converter/releases/tag/models-v1)）。

## 维护

- 本技能为通用模式：修改 `SKILL.md` 与 `references/project-notes.md` 后提交即可；可适配任意同构项目，无对外"主项目"同步关系
- 想把自己的同构项目发布为开箱即用技能：fork 本仓库，把"定位项目"一节改为指向你的仓库即可

## License

MIT © tututashu
