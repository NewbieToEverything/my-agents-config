# OpenCode 全局配置统一管理

将 OpenCode 的配置文件集中到同一项目目录下管理，通过符号链接同步到 OpenCode、Codex 和 pi（派）的默认路径，实现一套配置，三个 Agent 共享。

三者的全局指令与 skill 扫描位置（均支持软链接）：

| Agent | 全局指令文件（默认扫描） | 全局 skill 目录 |
|---|---|---|
| OpenCode | `~/.config/opencode/AGENTS.md` | `~/.agents/skills/`（兼容）、`~/.config/opencode/skills/` |
| Codex | `~/.codex/AGENTS.md`；若存在 `~/.codex/AGENTS.override.md` 则优先且只用它 | `~/.agents/skills/` |
| pi | `~/.pi/agent/AGENTS.md`（同目录也认 `AGENTS.override.md`、`CLAUDE.md`） | `~/.agents/skills/` |

`~/.agents/skills/` 是三者共用的 skill 入口，因此 skill 只需链接一次；全局指令没有共用目录，必须分别为每个 Agent 建一条软链接。

## 管理内容

| 文件/目录 | 说明 |
|---------|------|
| `skills/` | 自定义全局 Skill 源文件目录。存放在此的自定义 skill 通过软链接注入 `~/.agents/skills/<name>/`（OpenCode、Codex、pi 均扫描） |
| `AGENTS.md` | 全局指令文件，通过软链接分别供 OpenCode（`~/.config/opencode/AGENTS.md`）、Codex（`~/.codex/AGENTS.md`）与 pi（`~/.pi/agent/AGENTS.md`）加载 |
| `opencode.json` | OpenCode 的本地全局配置，保存在 `~/.config/opencode/opencode.json`。可能含 API key，不纳入版本控制 |
| `rules/` | 专项规则文件。OpenCode 通过 `opencode.json` 的 `instructions` 字段引入；Codex 和 pi 通过 `AGENTS.md`「按任务读取专项规则」一节按需读取 |

软链接总览：
| 目标路径 | 源路径 |
|---------|--------|
| `~/.agents/skills/<name>/` | `my-opencode-config/skills/<name>/` |
| `~/.config/opencode/AGENTS.md` | `my-opencode-config/AGENTS.md` |
| `~/.codex/AGENTS.md` | `my-opencode-config/AGENTS.md` |
| `~/.pi/agent/AGENTS.md` | `my-opencode-config/AGENTS.md` |
| `~/.config/opencode/rules` | `my-opencode-config/rules` |


## 管理方式

### 场景 A：从 GitHub 恢复

```bash
# 1. 创建项目目录
cd ~/projects
mkdir my-opencode-config

# 2. 克隆仓库
git clone git@github.com:<username>/my-opencode-config.git ~/projects/my-opencode-config
cd ~/projects/my-opencode-config

# 3. 创建符号链接（配置文件）
rm -f ~/.config/opencode/AGENTS.md
rm -rf ~/.config/opencode/rules
rm -f ~/.codex/AGENTS.md
rm -f ~/.pi/agent/AGENTS.md

ln -sf ~/projects/my-opencode-config/AGENTS.md ~/.config/opencode/AGENTS.md
ln -sf ~/projects/my-opencode-config/rules ~/.config/opencode/rules
ln -sf ~/projects/my-opencode-config/AGENTS.md ~/.codex/AGENTS.md
mkdir -p ~/.pi/agent
ln -sf ~/projects/my-opencode-config/AGENTS.md ~/.pi/agent/AGENTS.md

# 4. 将自定义 skill 链接到 .agents/skills/
ln -sf ~/projects/my-opencode-config/skills/* ~/.agents/skills/

# 5. 验证
ls -la ~/.config/opencode/ | grep -E "^l"
ls -la ~/.codex/ | grep -E "^l"
ls -la ~/.pi/agent/ | grep -E "^l"
ls -la ~/.agents/skills/ | grep -E "^l"
```

### 场景 B：现有配置迁移（首次执行）

```bash
# 1. 备份现有配置
cp ~/.config/opencode/AGENTS.md ~/.config/opencode/AGENTS.md.backup
cp ~/.config/opencode/opencode.json ~/.config/opencode/opencode.json.backup
cp ~/.codex/AGENTS.md ~/.codex/AGENTS.md.backup

# 2. 创建项目目录并复制配置
mkdir -p ~/projects/my-opencode-config
cd ~/projects/my-opencode-config
cp ~/.config/opencode/AGENTS.md ./
[ -d ~/.config/opencode/rules ] && cp -r ~/.config/opencode/rules/ ./
[ -d ~/projects/my-opencode-config/skills ] || mkdir -p skills

# 3. 创建符号链接（配置文件）
rm -f ~/.config/opencode/AGENTS.md
rm -rf ~/.config/opencode/rules
rm -f ~/.codex/AGENTS.md
rm -f ~/.pi/agent/AGENTS.md

ln -sf ~/projects/my-opencode-config/AGENTS.md ~/.config/opencode/AGENTS.md
ln -sf ~/projects/my-opencode-config/rules ~/.config/opencode/rules
ln -sf ~/projects/my-opencode-config/AGENTS.md ~/.codex/AGENTS.md
mkdir -p ~/.pi/agent
ln -sf ~/projects/my-opencode-config/AGENTS.md ~/.pi/agent/AGENTS.md

# 4. 将自定义 skill 链接到 .agents/skills/
ln -sf ~/projects/my-opencode-config/skills/* ~/.agents/skills/
```

### 场景 C：新增自定义全局 Skill

```bash
# 1. 创建 skill 目录
mkdir -p ~/projects/my-opencode-config/skills/<skill-name>
# 2. 编辑 SKILL.md ...

# 3. 同步到 .agents/skills/（OpenCode、Codex、pi 都会扫描，无需分别为每个 Agent 链接）
ln -sf ~/projects/my-opencode-config/skills/* ~/.agents/skills/

```

### 场景 D：只给某个 Agent 单独加规则

全局指令没有共用目录，只能指向同一个源文件。需要 Agent 专属规则时：

- Codex：新建 `~/.codex/AGENTS.override.md`（Codex 全局层只加载一个文件，写了它就不再读 `~/.codex/AGENTS.md`，因此内容需包含共享规则 + Codex 专属段）。
- pi：新建 `~/.pi/agent/APPEND_SYSTEM.md` 追加到系统提示，或启动时用 `--append-system-prompt <path>`。
- OpenCode：在 `opencode.json` 的 `instructions` 数组里追加文件、glob 或 URL。

注意：Codex 的系统提示词已内置官方 GPT-6.2 prompting 建议（任务推进、最终回复自包含等），无需在 `AGENTS.md` 重复。
