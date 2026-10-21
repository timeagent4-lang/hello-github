# Tablix release-20261031 无宿主机 Python 部署补丁

适用前提：Linux amd64 服务器已经收到并解压此前的完整离线包，目录中有 `start.sh`、`tablix.json.template` 和 `tablix-server-linux-amd64.tar`（或本地已经导入 `tablix-server:release-20261031` 镜像）。本补丁复用现有镜像内的 .NET 运行时，宿主机无需 Python 或 .NET SDK。

`install-from-text.sh` 为 UTF-8 文本，约35KB。它内嵌必需脚本、部署工具 DLL 和更新后的部署说明，先核对归档 SHA256，再解压并逐文件校验。执行仅还原文件，随后由部署人员进行初始化、配置和启动。

## 获取与还原

通过 Git 的原始文件地址下载 `install-from-text.sh` 和同目录的 `SHA256SUMS`，使用固定提交版本。仓库为私有时，由有访问权限的取件端通过既有 Git 认证获取，认证信息不要写入文件或日志。

将这两个文件保存到内网**原完整包中包含 start.sh 的目录**。可以由有联网能力的 AI 获取文本后写入文件；必须取回完整原文，保留内嵌 Base64，每个文件都使用 UTF-8/LF，不能仅获取网页预览或摘要。服务器实际取件需使用其允许的网络路径，补丁执行过程无需联网。

在该目录运行：

```bash
sha256sum -c SHA256SUMS
bash ./install-from-text.sh
```

安装后会生成独立的 `COPY_RUNTIME_SHA256SUMS`，记录本次6个运行和说明文件。原完整包的旧 `SHA256SUMS` 不用于校验本次覆盖后的文件。

## 新服务器初始化与启动

确认当前目录仍是原完整包目录。将 IP 换成服务器实际内网 IP：

```bash
export TABLIX_BIND_IP=10.0.0.10
export TABLIX_MCP_PORT=7713
export TABLIX_IMAGE=tablix-server:release-20261031
bash ./start.sh --init-only
```

在内网本地填写连接：

```bash
umask 077
cp mysql-connection.example.json mysql-connection.local.json
chmod 600 mysql-connection.local.json
vi mysql-connection.local.json
```

如文件已经存在，直接编辑现有文件。保留稳定的数据库 Id，填写 Hostname、Port、User、Password、DatabaseName、Schema，使用只读账号，`AllowedQueries` 保留 `["SELECT"]`。MySQL 的 Schema 通常与 DatabaseName 一致。

编辑 `deployment.local.json`，将 `ConnectionFiles` 设置为：

```json
"ConnectionFiles": ["mysql-connection.local.json"]
```

随后启动：

```bash
bash ./start.sh
docker ps --filter name=tablix-offline
docker port tablix-offline
```

Agent 接入 `http://服务器内网IP:7713/mcp`，刷新工具列表；默认前缀 `tablix_`，默认10个工具，三个 Context 写工具关闭。启动时对已存在 Id 读取并跳过，不覆盖已有 Context；新增连接会注册、等待结构抓取并执行 `SELECT 1 AS ok`。停止使用 `bash ./stop.sh`，数据与本地配置保留。

服务器运行需 Bash4+、Docker 和常见 Linux 工具；文本还原额外用到 `base64`、`tar`、`sha256sum`、`mktemp`。支持普通 rootful Docker 和 rootless Docker 的 UID 映射；rootful `userns-remap` 会提前报错，需要管理员配置映射目录权限。具体配置维护见安装后 `README-offline.md`。

## 给取件 AI 的执行要求

1. 从用户给定的**固定提交**原始地址取回完整的安装文本与 SHA256SUMS，不要根据网页预览重新生成代码。
2. 确认工作目录为原完整包目录，核对 `tablix.json.template` 与原镜像/镜像归档存在。
3. 核对下载文件 SHA256 后运行安装文本；如校验失败则停止，不使用缺失或截断内容。
4. 按以上步骤生成本地配置、填写用户提供的 MySQL 连接、注册并启动；缺少数据库信息时先完成不依赖它的初始化，再指出缺失字段。凭据只保留在内网本地。
5. 验收7713的 MCP 工具发现、三个禁用工具不可调用、MySQL只读查询与停止后重新启动的持久性。

## 验证边界

所含文件与已验证交付版本逐字节一致：32项部署工具测试、11项启动脚本测试、最终 DLL 复用原服务依赖和 SQLite 处理的宿主机验证、归档还原及哈希校验通过。制作端没有 Docker daemon，实际 Linux 容器权限、内网 MySQL 与 Agent 联调仍需在目标服务器验收。补丁不更改镜像版本、SQL行数/截断/超时或 MCP 鉴权行为。

上游项目：[jchristn/Tablix](https://github.com/jchristn/Tablix)。本补丁对应定制镜像 `tablix-server:release-20261031`；规划上线日期2026-10-31。
