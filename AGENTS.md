# 博客写作约定

## 文章配图

为 `data/blog/` 中的 Markdown / MDX 文章新增或替换本地配图时，使用 **Azure Picbed CLI** 上传，再把成功返回的图片链接插入文章。此约定适用于用户提供的照片、截图和生成的图片。已有图片链接可继续使用；只有用户要求迁移时才批量替换。

### 上传步骤

1. 确定待发布的本地图片及其在文章中的位置，给图片写有意义的 alt 文本。上传目标是公开 Git 仓库，先确认图片适合公开，尤其检查截图中的凭据和个人信息。
2. 使用 Node.js 22 或更新版本。从博客目录运行时，显式传入 CLI 和配置文件路径：

   ```sh
   node /Users/sylvain/Work/azure-picbed-cli/src/cli.js upload \
     '/absolute/path/to/image.png' \
     --config /Users/sylvain/Work/azure-picbed-cli/azure-picbed.json \
     --output markdown \
     --alt '图片内容说明'
   ```

   将示例图片路径和说明替换为实际值；每次上传一张图片。支持 PNG、JPEG、GIF、WebP，默认大小上限为 10 MiB。其他格式先转换为支持的格式，并检查转换结果。

3. 检查退出码。仅在退出码为 **0** 时，将 stdout 返回的完整 Markdown 图片行插入文章；CLI 此时已验证匿名访问的图片内容。
4. 原样保留 URL 的编码和所有查询参数，包括版本信息、`$format` 和 `download`。正文使用普通 Markdown 图片语法。若需要把新上传的图片用于 frontmatter 的 `images` 等字段，使用同一次上传返回的 URL。
5. 完成后检查每张新增配图是否已上传成功、alt 文本是否合适、文章是否仍引用本机路径或占位链接。有可用预览时，检查文章中图片的实际显示效果；无法预览时在交付说明中写明。

### 结构化输出与失败处理

- 需要单独取得 URL 时，将 `--output markdown` 改为 `--output json`。仅在退出码为 0 时解析 stdout，使用 `rawUrl`；`webUrl` 是仓库浏览页面。
- 退出码 **1**：读取 stderr JSON 中的 `error.code` 和 `error.message`，修正输入或报告配置、权限、网络问题。
- 退出码 **2**：图片已保存或复用，但匿名链接验证失败。保留待插图的位置，在交付说明中报告问题，不把候选链接当作可用图片插入。
- 遇到 `UPLOAD_UNCERTAIN`，网络恢复后用同一图片和配置重试，CLI 会检查并复用相同内容。遇到 `PATH_COLLISION`，停止上传并报告。
- 上传失败时保留本地原图和写作进度，明确列出尚未完成的配图。不要为绕过错误而改仓库权限、分支策略或删除远端图片。

### 配置与凭据

本机复用 CLI 仓库内的 `azure-picbed.json`，不需要将配置复制到博客仓库。仅检查文件是否存在；不要输出配置内容、PAT 或认证请求头，也不要把它们写进文章或提交到 Git。

若换了机器或目录，先定位 `azure-picbed-cli`，相应调整上述两个绝对路径。命令详情见 CLI 仓库的 `docs/agent-usage.md`，也可运行 `node /Users/sylvain/Work/azure-picbed-cli/src/cli.js help upload`。

若配置缺失，提示用户在自己的终端运行以下交互式向导；agent 不在无人值守进程中启动向导，也不要求用户在聊天中粘贴 PAT：

```sh
node /Users/sylvain/Work/azure-picbed-cli/src/cli.js config \
  --config /Users/sylvain/Work/azure-picbed-cli/azure-picbed.json
```
