# Claude Code 总被问 yes？别改 claude.md，改 settings.json

> 很多人刚用 Claude Code 时被频繁的 “Do you want to proceed? (yes/no)” 卡住，上网搜各种偏方。
> 其中一个流传很广的错误说法是 **“让 Claude 改 `claude.md` 把放行写进去”** —— 这不管用。
> 真正控制权限弹窗的是 **`settings.json`**。这个仓库讲清楚正确做法。

---

## 一、为什么改 `claude.md` 不管用

`CLAUDE.md` 是**给模型看的说明 / 上下文**（项目背景、风格约定、注意事项）。
你在里面写 “不要再问我 yes”，模型会看到，但 **Claude Code 的权限系统根本不读 `CLAUDE.md` 来决定是否放行**。

权限系统读的是 **`settings.json` 里的 `permissions` 字段**。所以放行规则必须写在那里。

| 文件 | 作用 | 能不能控制 yes 弹窗 |
|---|---|---|
| `CLAUDE.md` | 给模型的说明/上下文 | ❌ 不能 |
| `settings.json` → `permissions` | 权限白名单/黑名单 | ✅ 就是它 |

---

## 二、文件放哪

- **全局**（对所有项目生效）：`~/.claude/settings.json`
  - Windows：`C:\Users\你的用户名\.claude\settings.json`
- **项目级**（只对当前项目生效，可提交进 git 团队共享）：`<项目>/.claude/settings.json`
- **个人项目级**（不进 git，覆盖项目设置）：`<项目>/.claude/settings.local.json`

优先级：`settings.local.json` > 项目 `settings.json` > 全局 `settings.json`。

---

## 三、三层规则：allow / ask / deny

```jsonc
{
  "permissions": {
    "allow": [ /* 直接放行，不再问 */ ],
    "ask":   [ /* 强制每次都问，哪怕命中了 allow */ ],
    "deny":  [ /* 直接拒绝，最高优先级 */ ]
  }
}
```

- **`deny` 优先级最高**：命中就拒绝，谁都盖不过它 —— 用来兜底危险操作。
- **`ask` 次之**：命中就强制确认 —— 用来保护“对外/不可逆”的操作。
- **`allow` 最低**：命中就放行不问。
- **三个名单都没命中** → 回落到默认行为：**还是会问 yes**。

> 记法：**deny 禁 > ask 问 > allow 放**；没列到的默认问。

---

## 四、放行了什么 / 还会问 yes（对照表）

以本仓库的 [`settings.example.json`](./settings.example.json) 为例：

| 行为 | 还问 yes 吗 | 说明 |
|---|---|---|
| 读文件、搜索 (`Read` `Glob` `Grep`) | ❌ 不问 | 只读，无副作用 |
| 看 git 状态/改动/历史 | ❌ 不问 | `git status` / `diff` / `log` |
| 改文件 (`Edit`) | ❌ 不逐个确认 | 仍会展示改了什么 |
| 跑测试/构建/lint | ❌ 不问 | 精确匹配的 `npm run` 命令 |
| 本地提交 (`git add` / `commit`) | ❌ 不问 | 改动留在本地，可回退 |
| **推送 `git push`** | ✅ 仍会问 | 放进 `ask`，对外操作每次确认 |
| **`npm publish`** | ✅ 仍会问 | 同上，发布不可逆 |
| **`rm` 删除 / `sudo`** | 🚫 直接拒绝 | 放进 `deny` 兜底 |
| **读 `.env` / 密钥 / 私钥** | 🚫 直接拒绝 | 防止泄露 |
| 网络下载 (`curl` / `wget`) | 🚫 直接拒绝 | 防止拉取执行未知内容 |
| 名单外的任意新命令 | ✅ 默认问 | 没命中就回落到询问 |

---

## 五、怎么用（3 步）

1. 打开（没有就新建）你的 `~/.claude/settings.json`。
2. 把本仓库 [`settings.example.json`](./settings.example.json) 里的 `permissions` 片段**粘进去**。
   - ⚠️ **不要整个文件替换**，否则会覆盖掉你已有的 `model`、`env`、`hooks` 等配置。只合并 `permissions` 这一段。
3. 重启 Claude Code 生效。

> 也可以更省事：在 Claude Code 里直接说「把 xxx 命令加进放行白名单」，它会帮你改 `settings.json`。

---

## 六、语法速记：`Bash(...)` 匹配

- `"Bash(git status)"` —— 精确匹配这条命令。
- `"Bash(git diff:*)"` —— `:*` 是通配，匹配 `git diff` 开头的任意命令。
- `"Read"` / `"Edit"` —— 整类工具放行。
- `"Read(./.env)"` —— 针对具体文件路径。

⚠️ **通配要克制**：`"Bash(git:*)"` 会把 `git push --force`、`git reset --hard` 一起放行。越宽松越方便，也越危险，**自己看懂每一条再加**。

---

## 七、危险但有人会用：一键全放行

```bash
claude --dangerously-skip-permissions
```

这等于**完全跳过所有权限确认**（俗称 YOLO 模式）。
⚠️ 只建议在**一次性沙箱 / 容器 / 没有重要数据的环境**里用。日常开发、有真实代码和密钥的机器上**别这么干** —— 模型一旦执行了 `rm -rf` 或读走密钥，没人拦你。

---

## 八、安全提醒（重要）

- 别直接抄陌生人（包括本仓库）的 `allow` 列表就闭眼用。**放行 = 授权 Claude 不经确认就执行这些命令模式**。
- 加任何 `Bash(...)` 前，先想清楚“最坏情况它能干出什么”。
- 涉及对外（push / publish / 发邮件 / 调 API）和不可逆（删除 / 覆盖）的，放 `ask` 或 `deny`，别图省事放 `allow`。

---

## License

MIT —— 随便用、随便改、随便转。出了事自己负责。
