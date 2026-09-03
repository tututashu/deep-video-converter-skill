# deep-video-converter · WorkBuddy Skill

[Deep Video Converter](https://github.com/tututashu/deep-video-converter) 的 WorkBuddy 部署/运维技能包 —— 让 AI 助手能一键完成该工具的克隆、环境搭建、启动、五种特效模式的使用、免浏览器验证与故障排查。

> 对应项目仓库：https://github.com/tututashu/deep-video-converter

## 这是什么

本仓库本身就是标准 WorkBuddy skill 结构（顶层 `SKILL.md` + `references/`），clone 后直接可被识别为 skill，无需二次组装。

skill 触发后会自动按流程执行：

1. **定位或克隆** deep-video-converter 项目（本地已有则直接使用）
2. **一键 setup** — 创建双 Python 虚拟环境（torch 主环境 + mediapipe 人脸环境）、自动修复 macOS pyexpat、下载 4 个离线模型
3. **启动服务** — `http://localhost:8000`，浏览器拖拽视频 → 选择模式 → 下载保留原声的结果
4. **验证** — 免浏览器 CLI 端到端跑通 / 复用已算帧快速重合成
5. **排障** — ffmpeg、pyexpat、模型下载、Intel Mac torch 版本、人脸漏检等常见问题

## 安装

### 方式一：clone 到用户级技能目录（推荐）

```bash
git clone https://github.com/tututashu/deep-video-converter-skill.git ~/.workbuddy/skills/deep-video-converter
```

所有 WorkBuddy 项目通用。

### 方式二：clone 到项目级技能目录

```bash
git clone https://github.com/tututashu/deep-video-converter-skill.git <你的项目>/.workbuddy/skills/deep-video-converter
```

仅在该项目目录下生效。

### 方式三：下载 zip 手动解压

从 [Releases](https://github.com/tututashu/deep-video-converter-skill/releases) 下载 `deep-video-converter.zip`，解压后把 `deep-video-converter/` 文件夹放到 `~/.workbuddy/skills/` 或项目 `.workbuddy/skills/`。

## 在 Claude Code / Codex 中使用

本 skill 遵循 Agent Skills（SKILL.md）开放标准，同样适用于 **Claude Code**（`~/.claude/skills/`）与 **OpenAI Codex CLI**（`~/.codex/skills/`），格式零改动，clone 即用：

```bash
# Claude Code（用户级）
git clone https://github.com/tututashu/deep-video-converter-skill.git ~/.claude/skills/deep-video-converter

# Codex CLI（用户级）
git clone https://github.com/tututashu/deep-video-converter-skill.git ~/.codex/skills/deep-video-converter
```

完整说明（项目级安装、验证、卸载、注意事项）：[docs/cross-agent-install.md](docs/cross-agent-install.md)

### 验证安装

新会话中提问（任选其一），若 skill 生效 AI 会直接按流程执行而不是泛泛回答：

- "启动深度视频转换器"
- "帮我装并跑起来 deep-video-converter"
- "把这段视频转成姿态骨架叠加效果"

## 前置条件（运行 deep-video-converter 本身需要）

- Python 3.11（人脸环境）+ Python 3.12 / 3.10+（主环境）
- ffmpeg 在 PATH
- macOS / Linux / Windows x64
- 首次 setup 需联网下载模型（~450MB），之后完全离线可用

## 维护

- 主仓库：本项目 skill 文件与 [deep-video-converter](https://github.com/tututashu/deep-video-converter) 同步维护
- 修改 skill 内容：编辑 `SKILL.md` 与 `references/project-notes.md` 后提交即可

## License

MIT © tututashu
