# tool-vscode-tunnel 设计文档指针

本 Maven 子模块 `tools/tool-vscode-tunnel` 属于 kai-toolbox monorepo，设计文档不单独建库，统一落在 umbrella 项目下：

→ [VS Code Tunnel-current.md](../../kai-toolbox/design/VS%20Code%20Tunnel/VS%20Code%20Tunnel-current.md)
→ [VS Code Tunnel-coding.md](../../kai-toolbox/design/VS%20Code%20Tunnel/VS%20Code%20Tunnel-coding.md)

本文件仅用于让 `check-design-doc.js` hook 在按子模块识别 project root 时也能定位到对应设计文档，避免误判为"无设计"。任何实质内容更新一律在 umbrella 路径下进行。
