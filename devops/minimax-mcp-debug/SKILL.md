---
name: minimax-mcp-debug
description: MiniMax web search & image understanding via mmx CLI — 替代MCP方案的CLI实现，包含search query和vision describe命令。配额450次/周期。
triggers:
  - minimax web search 搜索
  - minimax vision 图片理解
  - minimax 图片分析
  - mmx cli minimax
  - minimax api key format
  - MINIMAX_API_SECRET config.yaml interpolation bug
---

# MiniMax MCP Web-Search Debugging

## Quick Diagnosis Flow

```
mcp_minimax_web_search fails
├─ "login fail: Please carry the API secret key in Authorization" 
│   → API key format wrong (sk-api-... vs sk-cp-...) OR key not loaded
├─ "1008-insufficient balance"
│   → Auth works, account has no credits
└─ Token error
    → env var not passed to MCP server
```

## Key Finding: minimax-coding-plan-mcp Auth Mechanism

Client (`minimax_mcp/client.py`) uses standard Bearer token:
```python
self.session.headers.update({
    'Authorization': f'Bearer {api_key}',
    'MM-API-Source': 'Minimax-MCP'
})
```
The "Please carry the API secret key" error is misleading — it actually means the key format is wrong or key not loaded.

## Key Format
- `sk-cp-...` — MiniMax Coding Plan key (correct for MCP)
- `sk-api-...` — OpenAI-compatible format (wrong for MiniMax MCP)

## hermes config set MINIMAX_API_SECRET bug

`hermes config set` writes **directly to config.yaml** for top-level keys, NOT to .env.

This means `${MINIMAX_API_SECRET}` interpolation in `mcp_servers.minimax.env` won't resolve, because the value is stored as a literal in config.yaml rather than as an env var reference.

**Workaround**: Use `printf` append to write directly to .env:
```bash
printf '\nMINIMAX_API_SECRET=sk-cp-...\n' >> ~/.hermes/.env
```

After editing .env manually, remove the erroneous literal entry from config.yaml if it was created.

## Verification Steps
1. Check .env has both `MINIMAX_API_KEY` and `MINIMAX_API_SECRET`
2. Check config.yaml mcp_servers.minimax.env references `${MINIMAX_API_SECRET}` correctly
3. Ensure no orphaned `MINIMAX_API_SECRET: sk-cp-...` literal at top-level of config.yaml
4. `hermes gateway restart` after changes
5. Test: `mcp_minimax_web_search` with a simple query

## ⚠️ MCP Approach: Abandoned in Favor of CLI

The `minimax-coding-plan-mcp` MCP server could not reliably receive `MINIMAX_API_SECRET` from the Hermes gateway env interpolation. The gateway's `hermes config set` command writes top-level keys directly to config.yaml as literals, breaking `${}` interpolation in mcp_servers config.

**Recommended Solution: Use `mmx-cli` instead of MCP.**

## mmx-cli Installation & Setup

```bash
# Install
npm install -g mmx-cli

# Authenticate with Coding Plan key (sk-cp-...)
mmx auth login --api-key "sk-cp-yyvU-9pPYQCV3FyuZTMPNOLdPihmzB-vKcd_gLpyxIRxguMFQVDa5jHKGn4WE8kMKavDy0lSuKdh6HLLRXedr6PKflCcNBhdr7E1P3yhUBW_UQH0GyGmWro"

# Verify
mmx auth status
```

## mmx-cli Commands

### Web Search
```bash
mmx search query --q "搜索关键词"
# 注意：必须用 "query --q"，不能用 "mmx search \"...\""
```

### Image Understanding
```bash
mmx vision describe --image <本地路径或URL> --prompt "问题"
# 示例
mmx vision describe --image photo.jpg --prompt "描述这张图片"
mmx vision describe --image "https://example.com/photo.jpg" --prompt "这张图里有什么？"
```

### Other Capabilities
```bash
mmx text chat --message "你好"          # 文本对话
mmx image "一只猫"                      # 图片生成
mmx speech synthesize --text "你好" --out hello.mp3  # 语音合成
mmx music generate --prompt "流行音乐" --out song.mp3  # 音乐生成
mmx video generate --prompt "日落" --download video.mp4  # 视频生成
mmx quota                                  # 查看配额
```

## Quota (2026-04-26 Cycle)
- `coding-plan-search`: 450次/周期
- `coding-plan-vlm` (vision): 450次/周期
- `MiniMax-M*` (text): 4,500次/周期

## Legacy: MCP Server Env Vars (Abandoned)
```yaml
mcp_servers:
  minimax:
    command: uvx
    args: [minimax-coding-plan-mcp, -y]
    env:
      MINIMAX_API_KEY: ${MINIMAX_API_KEY}
      MINIMAX_API_HOST: https://api.minimaxi.com
      MINIMAX_API_SECRET: ${MINIMAX_API_SECRET}  # broke: hermes config set writes literal to config.yaml
```

The MCP approach is DEPRECATED. Use `mmx-cli` for all MiniMax capabilities.
