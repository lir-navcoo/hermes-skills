---
name: himalaya
description: CLI to manage emails via IMAP/SMTP. Use himalaya to list, read, write, reply, forward, search, and organize emails from the terminal. Supports multiple accounts and message composition with MML (MIME Meta Language).
version: 1.0.0
author: community
license: MIT
metadata:
  hermes:
    tags: [Email, IMAP, SMTP, CLI, Communication]
    homepage: https://github.com/pimalaya/himalaya
prerequisites:
  commands: [himalaya]
---

# Himalaya Email CLI

Himalaya is a CLI email client that lets you manage emails from the terminal using IMAP, SMTP, Notmuch, or Sendmail backends.

## References

- `references/configuration.md` (config file setup + IMAP/SMTP authentication)
- `references/message-composition.md` (MML syntax for composing emails)

## Prerequisites

1. Himalaya CLI installed (`himalaya --version` to verify)
2. **macOS**: config at `~/Library/Application Support/himalaya/config.toml`; **Linux**: `~/.config/himalaya/config.toml`
3. IMAP/SMTP credentials configured (password stored securely via `auth.command`)

### Installation

```bash
# Pre-built binary (Linux/macOS — recommended)
curl -sSL https://raw.githubusercontent.com/pimalaya/himalaya/master/install.sh | PREFIX=~/.local sh

# macOS via Homebrew
brew install himalaya

# Or via cargo (any platform with Rust)
cargo install himalaya --locked
```

## Configuration Setup

Run the interactive wizard to set up an account:

```bash
himalaya account configure
```

Or create `~/.config/himalaya/config.toml` manually:

```toml
[accounts.personal]
email = "you@example.com"
display-name = "Your Name"
default = true

backend.type = "imap"
backend.host = "imap.example.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "you@example.com"
backend.auth.type = "password"
backend.auth.cmd = "pass show email/imap"  # or use keyring

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.example.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "you@example.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "pass show email/smtp"
```

## QQ邮箱配置示例

```toml
[accounts.qq]
email = "your_qq@qq.com"
display-name = "你的名字"
default = true

# 收件/本地存储用 maildir（QQ邮箱 IMAP 有 APPEND BINARY 兼容问题）
backend.type = "maildir"
backend.root-dir = "/Users/you/.local/share/himalaya/qq"

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.qq.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "your_qq@qq.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.command = "echo 你的授权码"
```

**maildir 文件夹结构**（必须手动创建，否则报错 `cannot find maildir matching name Sent`）：

```bash
mkdir -p ~/.local/share/himalaya/qq/{Sent,INBOX,Drafts,Junk,Trash}/{cur,new,tmp}
```

**QQ邮箱要点：**
- 授权码在网页版：设置 → 账户 → POP3/IMAP/SMTP/Exchange/CardDAV/CalDAV服务 → 开启 POP3/SMTP
- SMTP 用 port 587 + start-tls（不是 465 SSL）
- IMAP 用 imap.qq.com + port 993 + tls（但存已发送有 BINARY extension 兼容问题，推荐 maildir）
- QQ邮箱已发送文件夹名是 `Sent Messages`（带空格），不是 `Sent`

## Hermes Integration Notes

- **Reading, listing, searching, moving, deleting** all work directly through the terminal tool
- **Composing/replying/forwarding** — piped input (`cat << EOF | himalaya template send`) is recommended for reliability. Interactive `$EDITOR` mode works with `pty=true` + background + process tool, but requires knowing the editor and its commands
- Use `--output json` for structured output that's easier to parse programmatically
- The `himalaya account configure` wizard requires interactive input — use PTY mode: `terminal(command="himalaya account configure", pty=true)`

## Common Operations

### List Folders

```bash
himalaya folder list
```

### List Emails

List emails in INBOX (default):

```bash
himalaya envelope list
```

List emails in a specific folder:

```bash
himalaya envelope list --folder "Sent"
```

List with pagination:

```bash
himalaya envelope list --page 1 --page-size 20
```

### Search Emails

```bash
himalaya envelope list from john@example.com subject meeting
```

### Read an Email

Read email by ID (shows plain text):

```bash
himalaya message read 42
```

Export raw MIME:

```bash
himalaya message export 42 --full
```

### Reply to an Email

To reply non-interactively from Hermes, read the original message, compose a reply, and pipe it:

```bash
# Get the reply template, edit it, and send
himalaya template reply 42 | sed 's/^$/\nYour reply text here\n/' | himalaya template send
```

Or build the reply manually:

```bash
cat << 'EOF' | himalaya template send
From: you@example.com
To: sender@example.com
Subject: Re: Original Subject
In-Reply-To: <original-message-id>

Your reply here.
EOF
```

Reply-all (interactive — needs $EDITOR, use template approach above instead):

```bash
himalaya message reply 42 --all
```

### Forward an Email

```bash
# Get forward template and pipe with modifications
himalaya template forward 42 | sed 's/^To:.*/To: newrecipient@example.com/' | himalaya template send
```

### Write a New Email

**Non-interactive (use this from Hermes)** — pipe the message via stdin:

```bash
cat << 'EOF' | himalaya template send
From: you@example.com
To: recipient@example.com
Subject: Test Message

Hello from Himalaya!
EOF
```

Note: `himalaya message write` without piped input opens `$EDITOR`. This works with `pty=true` + background mode, but piping is simpler and more reliable.

**踩坑记录：** `himalaya email write` 不是有效子命令（2026-04-21 发现）。所有邮件发送必须走 `himalaya template send` + stdin 管道。

### Move/Copy Emails

Move to folder:

```bash
himalaya message move 42 "Archive"
```

Copy to folder:

```bash
himalaya message copy 42 "Important"
```

### Delete an Email

```bash
himalaya message delete 42
```

### Manage Flags

Add flag:

```bash
himalaya flag add 42 --flag seen
```

Remove flag:

```bash
himalaya flag remove 42 --flag seen
```

## Multiple Accounts

List accounts:

```bash
himalaya account list
```

Use a specific account:

```bash
himalaya --account work envelope list
```

## Attachments

Save attachments from a message:

```bash
himalaya attachment download 42
```

Save to specific directory:

```bash
himalaya attachment download 42 --dir ~/Downloads
```

## Output Formats

Most commands support `--output` for structured output:

```bash
himalaya envelope list --output json
himalaya envelope list --output plain
```

## Debugging

Enable debug logging:

```bash
RUST_LOG=debug himalaya envelope list
```

Full trace with backtrace:

```bash
RUST_LOG=trace RUST_BACKTRACE=1 himalaya envelope list
```

## Tips

- Use `himalaya --help` or `himalaya <command> --help` for detailed usage.
- Message IDs are relative to the current folder; re-list after folder changes.
- For composing rich emails with attachments, use MML syntax (see `references/message-composition.md`).
- Store passwords securely using `pass`, system keyring, or a command that outputs the password.

## Troubleshooting

| 错误信息 | 原因 | 解决方案 |
|---|---|---|
| `cannot send message without a sender` | piped send缺少From header | 必须加 `From: your_email@domain.com` 且必须匹配配置的账号邮箱 |
| `cannot find UID of appended IMAP message` | QQ邮箱 IMAP BINARY extension 不兼容 | 改用 maildir 后端存储已发送 |
| `cannot find maildir matching name Sent` | maildir 文件夹未创建或命名错误 | 先 `mkdir -p` 创建 Sent/INBOX/Drafts 等文件夹 |
| `TOML parse error: unknown variant 'ssl-tls'` | encryption type 拼写错误 | IMAP 用 `tls`，SMTP 用 `start-tls` |
| `cannot add message: feature not available` | 没有后端配置 | 至少需要一个后端（maildir 或 imap）才能发送 |
| `cannot parse email: empty entries` | maildir格式邮件读取失败 | 用 `envelope list` 替代 `message read` 查看邮件列表 |
| `Foreground command uses '&' backgrounding` | heredoc 管道发送在某些环境失败 | 改用 temp file 中转：`write_file` 写内容到 `/tmp/email.txt`，再 `cat /tmp/email.txt | himalaya template send` |
| `Could not determine home directory` | heredoc 管道在某些环境（如cron/daemon）下无法确定 HOME | 改用 temp file 中转：`write_file` 写内容到 `/tmp/email.txt`，再 `cat /tmp/email.txt | himalaya template send` |

### 发送邮件的关键约束

使用管道发送时 (`cat << EOF | himalaya template send`)：
- **必须包含 `From:` header**，且地址必须与 `[accounts.xxx].email` 配置的地址完全一致
- 必须包含 `To:` header
- 必须包含 `Subject:` header
- `Date:` header 可选（会自动生成）

```bash
# ✅ 正确 — From必须匹配配置
cat << 'EOF' | himalaya template send
From: 78080114@qq.com
To: 78080114@qq.com
Subject: Test

Body here.
EOF

# ❌ 错误 — 缺少From或From不匹配
cat << 'EOF' | himalaya template send
To: 78080114@qq.com
Subject: Test

Body here.
EOF
```

查找配置中的邮箱地址：
```bash
# macOS
grep -A2 '\[accounts' ~/Library/Application\ Support/himalaya/config.toml | grep email

# Linux
grep -A2 '\[accounts' ~/.config/himalaya/config.toml | grep email
```
