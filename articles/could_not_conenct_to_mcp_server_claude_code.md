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
![Could not connect to MCP serverエラー](/images/error.png)

ネットを調べると、`command`をフルパスで指定することで解決できるという情報がありましたが、自分の環境では問題が解消しませんでした。
ログを見ると、nodeが見つからないそうでした。

## 解決方法

最終的に、`env`に`PATH`を追加することで問題を解決できました。具体的には以下の設定を行いました：

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

