---
title: "Could not connect to MCP server: Claude Codeのエラー解決"
emoji: "🛣️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claude", "mcp", "claudecode", "error"]
published: true
---

https://zenn.dev/kazuph/articles/5a6cc61ae21940
上記の記事を読んで、自分もClaude CodeをMCPサーバーにしてClaude Desktopを使おうとしたのですが、うまく行かなかったので、その時の解決方法を残しておきます。


こんな感じのエラーが出ました:
![Could not connect to MCP serverエラー](https://storage.googleapis.com/zenn-user-upload/be9ed2a66222-20250327.png)

## 試したこと(だめだった)
上記の記事内にも書いてあるのですが、`command`にフルパスで指定したら解決したそうです。調べてみると他にもこれで解決したって記事がいくつか出てきました。
こんな感じ
```json
{
  "mcpServers": {
    "claude_code": {
      "command": "/Users/hirofumiomi/.local/share/mise/installs/node/22.13.1/bin/claude",
      "args": ["mcp", "serve"],
      "env": {}
    }
  }
}
```
ただ、自分の場合はそれで解決しませんでした。

ログを見ると、nodeが見つからないようでした。

## 解決方法

`env`に`PATH`を追加することで無事解決しました。

```json
{
  "mcpServers": {
    "claude_code": {
      "command": "claude",
      "args": ["mcp", "serve"],
      "env": {
        "PATH": "/Users/hirofumiomi/.local/share/mise/installs/node/22.13.1/bin:${PATH}"
      }
    }
  }
}
```

