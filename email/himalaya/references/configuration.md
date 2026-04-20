# Himalaya Configuration Reference

Configuration file location: `~/.config/himalaya/config.toml`

## Minimal IMAP + SMTP Setup

```toml
[accounts.default]
email = "user@example.com"
display-name = "Your Name"
default = true

# IMAP backend for reading emails
backend.type = "imap"
backend.host = "imap.example.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "user@example.com"
backend.auth.type = "password"
backend.auth.raw = "your-password"

# SMTP backend for sending emails
message.send.backend.type = "smtp"
message.send.backend.host = "smtp.example.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "user@example.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.raw = "your-password"
```

## Password Options

### Raw password (testing only, not recommended)

```toml
backend.auth.raw = "your-password"
```

### Password from command (recommended)

```toml
backend.auth.cmd = "pass show email/imap"
# backend.auth.cmd = "security find-generic-password -a user@example.com -s imap -w"
```

### System keyring (requires keyring feature)

```toml
backend.auth.keyring = "imap-example"
```

Then run `himalaya account configure <account>` to store the password.

## Gmail Configuration

```toml
[accounts.gmail]
email = "you@gmail.com"
display-name = "Your Name"
default = true

backend.type = "imap"
backend.host = "imap.gmail.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "you@gmail.com"
backend.auth.type = "password"
backend.auth.cmd = "pass show google/app-password"

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.gmail.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "you@gmail.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "pass show google/app-password"
```

**Note:** Gmail requires an App Password if 2FA is enabled.

## QQ邮箱配置

**关键问题：** QQ邮箱IMAP在APPEND消息到Sent文件夹时会失败（`cannot find UID of appended IMAP message`），原因是QQ邮箱的已发送文件夹名"Sent Messages"含空格，且IMAP实现有特殊性。

**解决方案：** 用 maildir 本地存储作为后端存已发送，SMTP用于发送。

**完整配置示例：**

```toml
[accounts.qq]
email = "78080114@qq.com"
display-name = "多宝道人"
downloads-dir = "~/Downloads"

# 本地 maildir 存储（用于存已发送等）
backend.type = "maildir"
backend.root-dir = "/Users/lirui/.local/share/himalaya/qq"

# SMTP 发送（QQ邮箱用 start-tls + 587）
message.send.backend.type = "smtp"
message.send.backend.host = "smtp.qq.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "78080114@qq.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "echo <your-authorization-code>"
```

**maildir目录结构（手动创建，否则报错`cannot find maildir matching name Sent`）：**

```bash
mkdir -p ~/.local/share/himalaya/qq/Sent/{cur,new,tmp}
mkdir -p ~/.local/share/himalaya/qq/INBOX/{cur,new,tmp}
mkdir -p ~/.local/share/himalaya/qq/Drafts/{cur,new,tmp}
mkdir -p ~/.local/share/himalaya/qq/Trash/{cur,new,tmp}
mkdir -p ~/.local/share/himalaya/qq/Junk/{cur,new,tmp}
```
**注意：**
- 文件夹名不能有`{}`等特殊字符，用裸名如`Sent`而非`{Sent}`
- **macOS 默认配置路径**：`~/Library/Application Support/himalaya/config.toml`（不是`~/.config/himalaya/config.toml`）
- `backend.folders.rename` 对 QQ 邮箱无效，rename 后仍然报`cannot find UID of appended IMAP message`，maildir 是唯一可靠方案

**发送测试：**
```bash
echo 'From: you@qq.com\nTo: you@qq.com\nSubject: Test\n\nBody' | himalaya template send --account qq
```

## iCloud Configuration

```toml
[accounts.icloud]
email = "you@icloud.com"
display-name = "Your Name"

backend.type = "imap"
backend.host = "imap.mail.me.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "you@icloud.com"
backend.auth.type = "password"
backend.auth.cmd = "pass show icloud/app-password"

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.mail.me.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "you@icloud.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "pass show icloud/app-password"
```

**Note:** Generate an app-specific password at appleid.apple.com

## Folder Aliases

Map custom folder names:

```toml
[accounts.default.folder.alias]
inbox = "INBOX"
sent = "Sent"
drafts = "Drafts"
trash = "Trash"
```

## Multiple Accounts

```toml
[accounts.personal]
email = "personal@example.com"
default = true
# ... backend config ...

[accounts.work]
email = "work@company.com"
# ... backend config ...
```

Switch accounts with `--account`:

```bash
himalaya --account work envelope list
```

## Notmuch Backend (local mail)

```toml
[accounts.local]
email = "user@example.com"

backend.type = "notmuch"
backend.db-path = "~/.mail/.notmuch"
```

## OAuth2 Authentication (for providers that support it)

```toml
backend.auth.type = "oauth2"
backend.auth.client-id = "your-client-id"
backend.auth.client-secret.cmd = "pass show oauth/client-secret"
backend.auth.access-token.cmd = "pass show oauth/access-token"
backend.auth.refresh-token.cmd = "pass show oauth/refresh-token"
backend.auth.auth-url = "https://provider.com/oauth/authorize"
backend.auth.token-url = "https://provider.com/oauth/token"
```

## Additional Options

### Signature

```toml
[accounts.default]
signature = "Best regards,\nYour Name"
signature-delim = "-- \n"
```

### Downloads directory

```toml
[accounts.default]
downloads-dir = "~/Downloads/himalaya"
```

### Editor for composing

Set via environment variable:

```bash
export EDITOR="vim"
```
