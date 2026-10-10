# RedBookSkills 评论能力接入

## 依赖与环境

从当前环境已配置的技能根目录或本技能同级目录定位 `RedBookSkills/SKILL.md`，操作前读取。Codex 通常使用 `$CODEX_HOME/skills`，未设置时使用 `~/.codex/skills`。缺失时仍可拟写草稿，不能假称已读取或回复。

复用 `scripts/cdp_publish.py` 的 `check-login`、`get-feed-detail`、`get-notification-mentions` 和 `respond-comment`。不调用 `publish_pipeline.py`、`post-comment-to-feed` 或私信能力。本任务是回复已有评论，不能以发布一级评论代替。

Windows PowerShell 命令环境示例：先将 `$commentPython` 绑定到当前项目可用的 Python 3 绝对路径，`$redbookRoot` 绑定到已定位的 RedBookSkills 目录，`$commentCacheRoot` 绑定到用户指定的缓存根目录。不要覆盖已配置的有效路径。

```powershell
$env:PYTHONUTF8 = '1'
$env:PYTHONIOENCODING = 'utf-8'
$env:PYTHONDONTWRITEBYTECODE = '1'
$env:TEMP = Join-Path $commentCacheRoot 'temp'
$env:TMP = $env:TEMP
$env:PIP_CACHE_DIR = Join-Path $env:TEMP 'pip'
$env:UV_CACHE_DIR = Join-Path $env:TEMP 'uv'
$env:npm_config_cache = Join-Path $env:TEMP 'npm'
$env:HF_HOME = Join-Path $env:TEMP 'huggingface'
$env:XDG_CACHE_HOME = Join-Path $env:TEMP 'xdg'
$redbookScript = Join-Path $redbookRoot 'scripts/cdp_publish.py'
```

先检查 Python 3 和上游 requirements.txt 中的依赖（当前包括 requests、websockets）；不假定 `py` 或 `python` 一定指向正确环境。目录按需建立，所有变量只作用于助手进程。macOS/Linux 使用同样的 CLI 参数及本机实际路径，不照搬 PowerShell 语法。复用已登录浏览器，不盲目重启或更换默认账号。上游硬编码缓存或 Profile 路径优先复用现有兼容联接；不能声称 TEMP 已覆盖全部缓存，不修改 LOCALAPPDATA 来强制搬家。

## 读取公开评论

```powershell
& $commentPython $redbookScript --reuse-existing-tab check-login
& $commentPython $redbookScript --reuse-existing-tab get-notification-mentions
& $commentPython $redbookScript --reuse-existing-tab get-feed-detail --feed-id $feedId --xsec-token $xsecToken --load-all-comments --limit 20 --click-more-replies --reply-limit 10
```

按任务选用命令，不必每次获取通知。全局参数如 `--account` 在子命令前，详情参数在子命令后。变量必须来自真实页面或工具返回，不能伪造 ID/token。`get-notification-mentions` 用于评论与 @ 通知，不是私信接口；仅纳入授权范围内的公开评论，不把每条 @ 通知都当作待回复评论。

读取笔记、父评论和已有回复，核对账号与回复对象。`--limit 20` 是当前读取范围示例，不代表用户授权回复 20 条；加载标志也不证明已加载所有评论。用户要求全部时按实际可读范围继续加载，说明任何不可访问的部分。

## 目标定位的重要限制

当前 `respond-comment` 会定位、填入并直接提交，没有草稿预览参数。**不要为了测试或预览运行该命令。** 草稿保存在本地即可。

当前源码的匹配是打分机制：comment ID 相等得 100 分，作者包含匹配得 30 分，文本片段包含匹配得 20 分。若不提供任何定位字段，可能选中第一条评论；同时提供 ID、作者和片段也不保证严格同时匹配。

因此发送时只传已验证的 `--comment-id`，不额外附加模糊字段，以免 ID 失配后回退到另一个人。先确认该 ID 对应的作者、原文、笔记及唯一目标。没有稳定 ID 时继续做只读定位和草稿；无法可靠定位就暂停该条，不以作者昵称或常见短句猜测目标。

当前回复函数自动加载约 30 条评论并展开部分楼中楼；目标未加载时不要换一个目标发送，也不要把这种失败当作“该评论不存在”。需要时采用可核验的浏览器操作路径，并保持同样的授权、精确定位与发送后核验要求。

## 授权、发送与去重

用户要求拟稿时只交付草稿；明确要求在指定账号和范围回复时可以执行。已有有效授权无需每条重复确认。账号或范围不明确时，先完成可读上下文与候选回复，再问缺少的关键项。上游发布笔记的“标题、正文、图片确认”不套用到评论任务。

发送前读取已有回复，检查同一账号是否已回应目标评论。用账号标识、笔记 ID 和评论 ID 作为目标键，记录草稿、待发送、已发送、失败或待核验；已回复后出现新问题，仅在当前授权范围内处理新的目标评论。执行记录不能替代平台实时检查。

将实际定稿写入任务缓存中的 UTF-8 文本文件，使用绝对路径传入，避免多行文本和特殊字符被 shell 解析：

```powershell
# 此命令会直接发送：仅在已有明确回复授权且目标核验通过后执行。
& $commentPython $redbookScript --reuse-existing-tab respond-comment --feed-id $feedId --xsec-token $xsecToken --comment-id $commentId --content-file $replyFile
```

逐条发送并读取对应楼中楼，验证回复属于预期账号、挂在正确父评论下、文本与定稿相符。尽可能保存回复 ID 或可用链接。提交返回值不能单独证明可见成功。

超时、网络断开或结果不明：先重读目标评论，已出现该回复就不重发。仍无法确认则标记待核验并暂停该条；页面故障、登录失效或平台限制影响整批时暂停整批发送。没有发送发生且原因明确可修复时才重试；同一错误修复后仍重复出现则停止对应操作并报告。不要循环提交或换账号绕过限制。

最终报告区分已发送、仅草稿、跳过、失败及待核验，不把“已调用命令”当作“已回复成功”。本技能不在未经请求的情况下继续后台巡查。
