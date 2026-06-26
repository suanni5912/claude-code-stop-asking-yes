# Claude Code 一直问 yes？精确配 `settings.json` 才是正解

> 刚用 Claude Code 的人几乎都被这个卡住：干点啥都弹一句
> “Do you want to proceed? (yes/no)”，效率全耗在按 yes 上。
>
> 网上的办法基本两类，**都不太行**。真正好用的是改 `settings.json` 的权限白名单——
> **该免的免、该拦的拦、该问的还问**。这个仓库讲清楚怎么配。

---

## 一、网上两种常见办法，为什么都不够好

| 办法 | 怎么做 | 问题 |
|---|---|---|
| **切换 mode** | `Shift+Tab` 切到 auto-accept / acceptEdits 模式 | 只免掉**改文件**的确认，跑命令照样问；而且**只在本次会话有效**，重开又得切。治标不治本。 |
| **一键全放行** | `claude --dangerously-skip-permissions` | 确实再也不问了——**但任何命令都不确认**，`rm -rf`、读密钥、联网下载执行全部畅通无阻。**风险太大，日常机器别用**。 |
| ✅ **配 `settings.json`（本仓库）** | 在权限白名单里精确写明放行/拦截 | **持久生效**、**按命令粒度控制**、**危险操作仍然拦**。一次配好，长期省心。 |

切 mode 太轻（管不住命令、还不持久），全放行太重（等于裸奔）。
**中间这条精确配权限的路，才是兼顾省心和安全的做法。**

---

## 二、放行规则写在哪

权限由 **`settings.json` 的 `permissions` 字段**控制。放哪生效不同：

- **全局**（对所有项目生效）：`~/.claude/settings.json`
  - Windows：`C:\Users\你的用户名\.claude\settings.json`
- **项目级**（只对当前项目，可提交进 git 团队共享）：`<项目>/.claude/settings.json`
- **个人项目级**（不进 git，覆盖项目设置）：`<项目>/.claude/settings.local.json`

优先级：`settings.local.json` > 项目 `settings.json` > 全局 `settings.json`。

---

## 三、核心：allow / ask / deny 三层

```jsonc
{
  "permissions": {
    "allow": [ /* 直接放行，不再问 */ ],
    "ask":   [ /* 强制每次都问，哪怕命中了 allow */ ],
    "deny":  [ /* 直接拒绝，最高优先级 */ ]
  }
}
```

- **`deny` 优先级最高**：命中就拒，谁都盖不过它 —— 兜底危险操作。
- **`ask` 次之**：命中就强制确认 —— 保护“对外 / 不可逆”的操作。
- **`allow` 最低**：命中就放行不问。
- **三个名单都没命中** → 回落默认行为：**还是会问 yes**。

> 记法：**deny 禁 > ask 问 > allow 放**；没列到的默认问。
>
> 这正是它比“切 mode / 全放行”强的地方——**不是一刀切，而是你自己划线**：
> 信得过的免确认，对外的强制问，危险的直接禁。

---

## 四、放行了什么 / 还会问 yes（对照表）

以本仓库的 [`settings.example.json`](./settings.example.json) 为例：

| 行为 | 还问 yes 吗 | 说明 |
|---|---|---|
| 读文件、搜索 (`Read` `Glob` `Grep`) | ❌ 不问 | 只读，无副作用 |
| 看 git 状态 / 改动 / 历史 | ❌ 不问 | `git status` / `diff` / `log` |
| 改文件 (`Edit`) | ❌ 不逐个确认 | 仍会展示改了什么 |
| 跑测试 / 构建 / lint | ❌ 不问 | 精确匹配的 `npm run` 命令 |
| 本地提交 (`git add` / `commit`) | ❌ 不问 | 改动留本地，可回退 |
| **推送 `git push`** | ✅ 仍会问 | 放进 `ask`，对外操作每次确认 |
| **`npm publish`** | ✅ 仍会问 | 同上，发布不可逆 |
| **`rm` 删除 / `sudo`** | 🚫 直接拒绝 | 放进 `deny` 兜底 |
| **读 `.env` / 密钥 / 私钥** | 🚫 直接拒绝 | 防泄露 |
| 网络下载 (`curl` / `wget`) | 🚫 直接拒绝 | 防拉取执行未知内容 |
| 名单外的任意新命令 | ✅ 默认问 | 没命中就回落到询问 |

---

## 五、怎么用（3 步）

1. 打开（没有就新建）你的 `~/.claude/settings.json`。
2. 把本仓库 [`settings.example.json`](./settings.example.json) 里的 `permissions` 片段**粘进去**。
   - ⚠️ **不要整个文件替换**，否则会覆盖掉你已有的 `model`、`env`、`hooks` 等配置。只合并 `permissions` 这一段。
3. 重启 Claude Code 生效。

> 更省事：在 Claude Code 里直接说「把 xxx 命令加进放行白名单」，它会帮你改 `settings.json`。

---

## 六、语法速记：`Bash(...)` 匹配

- `"Bash(git status)"` —— 精确匹配这条命令。
- `"Bash(git diff:*)"` —— `:*` 是通配，匹配 `git diff` 开头的任意命令。
- `"Read"` / `"Edit"` —— 整类工具放行。
- `"Read(./.env)"` —— 针对具体文件路径。

⚠️ **通配要克制**：`"Bash(git:*)"` 会把 `git push --force`、`git reset --hard` 一起放行。
越宽松越方便，也越危险，**自己看懂每一条再加**。

---

## 七、安全提醒（重要）

- 别直接抄陌生人（包括本仓库）的 `allow` 列表就闭眼用。**放行 = 授权 Claude 不经确认就执行这些命令模式**。
- 加任何 `Bash(...)` 前，先想清楚“最坏情况它能干出什么”。
- 涉及对外（push / publish / 发邮件 / 调 API）和不可逆（删除 / 覆盖）的，放 `ask` 或 `deny`，别图省事放 `allow`。

---

## License

MIT —— 随便用、随便改、随便转。出了事自己负责。
