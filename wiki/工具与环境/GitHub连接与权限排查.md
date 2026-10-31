# GitHub 连接与权限排查

给 AI 工具接 GitHub，最容易卡在三个地方：用错了凭证、权限给了但不能写、改了权限却没生效。这篇记一次完整的排查路径，换电脑时照着走。

## 三条硬知识

**一、内置连接器给的权限，可能完全没有仓库读写。**

WorkBuddy 的 GitHub 连接器（CodeBuddy-Connector）是 OAuth App。它的权限列表全是用户级的：身份、邮箱、followers、gist、SSH keys、Codespaces secrets、starred 和 watched 仓库的查看——没有任何一条是仓库内容的读写。

结果是：读写公开仓库时看起来一切正常（公开仓库本来就不需要权限），一碰私有仓库就失败。容易误判成配置问题，其实是这个 App 从来没申请过仓库权限。

**二、私有仓库没权限时，GitHub 返回 404，不是 403。**

这一点最误导人。403 会让人想到权限，404 会让人想到"仓库不存在、名字打错了"，于是去改仓库名——方向从一开始就错了。GitHub 对私有仓库故意返回 404 来隐藏它的存在，所以看到 404 要先怀疑权限。

真正的 403 只在"能看见仓库、但这个动作不被允许"时出现，比如 `POST /git/blobs -> 403 Resource not accessible`，那才是权限档位不够。

**三、Fine-grained PAT 有两个地方要一起对。**

- Repository access：选 All repositories，或 Only select repositories 里逐个勾。**如果停在 Public repositories，仓库一旦改成私有就自动掉出去**，跟有没有勾名字无关。
- Repository permissions → Contents：这是个下拉框，有 Read 和 Read and write 两档，很容易只停在前一档。只读的话读没问题，写会 403。

## 排查顺序

三步分开测，比反复重新授权省时间：

1. 凭证是否有效：`GET /user` 返回 200 就说明 token 活着。
2. 能否看见仓库：`GET /repos/{owner}/{repo}`。404 就去查 Repository access，不要查仓库名。
3. 能否写入：`POST /repos/{owner}/{repo}/git/blobs`。这里 403 就是 Contents 档位不够。

## 一个容易忽略的坑

权限改完不会立刻生效，有缓存延迟。改完马上测可能还是 404，会以为白改了。等一两分钟再试。

## 这次的结论

- 内置 GitHub 连接器只能读公开仓库，写不了任何仓库，公开的也不行。
- 要让 AI 读写私有仓库，必须自建 fine-grained PAT，存进 `~/.workbuddy/mcp.json`，从里面取用。
- 不要为了"让 AI 能读"把仓库改成 public。公开只解决读、不解决写，赔上的是内容可见性。

## 换电脑时的步骤

1. 新机器上建 fine-grained PAT：Repository access 选 All repositories，Contents 设为 Read and write。
2. 写 `~/.workbuddy/mcp.json`，加一个 stdio MCP server 项，token 放 `env` 里。
3. 不需要让它在 MCP 面板里显示，AI 直接从配置文件读 token 就能工作。
4. 用 `GET /repos/{owner}/{repo}` 验证是否 200。
5. token 有有效期（这次是一个月），到期重新生成并更新配置。

## 不要写进这里的东西

token 值本身。凭据只存在配置文件里，不进仓库——即使是私有仓库也不要。
