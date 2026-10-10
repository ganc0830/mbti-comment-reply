# mbti-comment-reply

结合五种 MBTI 表达风格的小红书**公开评论回复 Skill**，通过本地 [RedBookSkills](https://github.com/Angiin/Post-to-xhs) 读取评论，并按用户授权发送回复。

ENFP 提供热情与趣味，ENFJ 提供温暖鼓励，INFJ 提供理解与边界，INFP 提供真诚细腻，ISFJ 提供具体可靠的回答。它们用于调节语气，不用于判断评论者的人格；评论主题不必与 MBTI 有关。

## 功能与范围

- 根据笔记、父评论和已有回复拟写自然的评论回复。
- 支持先看草稿，或在明确授权的账号和范围内逐条发送。
- 发送前核验评论 ID，发送后核验结果，防止错人和重复回复。
- **不读取或回复私信，不发布笔记，不自动点赞、收藏、关注。**
- 本仓库是供助手执行的技能说明，复用 RedBookSkills 的脚本，不是独立运行的机器人服务。

## 安装

先安装并配置 RedBookSkills、其 Python 3 依赖以及可用的小红书浏览器登录会话。本仓库不包含上游代码、账号资料或浏览器凭据。

将本仓库克隆到 Codex 的技能目录。PowerShell 示例：

```powershell
$skillRoot = if ($env:CODEX_HOME) { Join-Path $env:CODEX_HOME 'skills' } else { Join-Path $env:USERPROFILE '.codex/skills' }
git clone https://github.com/ganc0830/mbti-comment-reply.git (Join-Path $skillRoot 'mbti-comment-reply')
```

macOS/Linux 示例：

```sh
git clone https://github.com/ganc0830/mbti-comment-reply.git "${CODEX_HOME:-$HOME/.codex}/skills/mbti-comment-reply"
```

若目标目录已经存在，请更新现有安装，不覆盖本地修改。技能使用环境实际的 Python、技能目录及缓存位置；个人存储偏好可写入工作区 AGENTS.md。

## 使用

先看草稿：

> 使用 $mbti-comment-reply，结合这篇笔记的上下文，为这些评论拟写回复，先给我看，不发送。

授权回复：

> 使用 $mbti-comment-reply，回复我指定账号这篇笔记下尚未回复的评论，逐条检查，不处理私信。

需要定时运行时，在支持调度的助手中另行创建定时任务，明确账号、时间、范围和发送授权。**安装本技能不会自动创建定时任务，也不会自行发送评论。**

## 文件

- [SKILL.md](SKILL.md)：入口、范围及工作流程。
- [references/reply-styles.md](references/reply-styles.md)：五种风格与评论示例。
- [references/redbook-operations.md](references/redbook-operations.md)：环境、CLI、目标定位与去重。
- [agents/openai.yaml](agents/openai.yaml)：技能展示信息。

当前接入依据本地 RedBookSkills 的 `respond-comment` 命令：它会直接发送，没有草稿模式。草稿必须保留在本地；平台页面或上游脚本变化后，需要重新核验接口和定位行为。
