# MCP 调用日志补丁（2026-10-21）

用于此前交付的 Tablix release-20261031 Linux amd64 镜像，容器名 `tablix-offline`。补丁仅替换 `/app/Tablix.Server.dll` 和对应 PDB；复用容器内运行时、原 Core 与依赖，宿主机无需 Python 或 .NET。

## 解决什么

1. SyslogLogging 2.3.1 的控制台开关要求存在交互终端；没有 TTY 时即使配置 ConsoleLogging=true，也可能只写文件。本补丁在用户要求控制台日志且日志库关闭该输出时，将已完成的日志条目写入 stdout。仍遵守原日志级别，不重复输出。
2. MCP 工具处理器发生异常时，客户端得到 generic internal error，但原应用日志未记录对应异常。本补丁增加 `[MCP]` 开始、结束、耗时、状态、请求编号，以及异常类型、SQLite 错误码、受限缺表/缺列提示与调用方法。客户端仍返回通用错误。

不记录 MCP 参数、SQL、数据行、Context 或一般异常全文；不启用模型或外部遥测。此补丁提供诊断能力，尚不能证明你现场 internal error 的根因已经修复。

## 先还原文件：可交给 AI

从本目录对应的 Git 固定提交原始地址下载 `restore-from-text.sh` 和 `SHA256SUMS`，完整保留 UTF-8/LF 文本与 Base64。下载端核对 SHA256 后，把文件传入内网；服务器离线还原无需联网。

在独立工作目录执行：

```bash
sha256sum -c SHA256SUMS
bash ./restore-from-text.sh
```

脚本只还原 `tablix-mcp-diagnostics-20261021` 目录并校验文件，不执行 Docker、不修改服务、不填写凭据。目标目录已存在会停止，不覆盖。

给取件 AI 的约束：只获取原文、校验、还原文件，并报告还原路径；不得停止、启动、重建容器，也不得执行下面的部署步骤。部署只由用户本人完成。

## 应用补丁：只由用户本人执行

进入还原的 `tablix-mcp-diagnostics-20261021` 目录。先确认新文件校验通过，再备份现有文件：

```bash
sha256sum -c SHA256SUMS
umask 077
backup_dir="./backup-$(date +%Y%m%d-%H%M%S)"
mkdir "$backup_dir"
docker cp tablix-offline:/app/Tablix.Server.dll "$backup_dir/Tablix.Server.dll"
docker cp tablix-offline:/app/Tablix.Server.pdb "$backup_dir/Tablix.Server.pdb"
printf '%s  %s\n' '1d029aa79f1aedbcfd1407dbc88bb8d0376ecd918947bf21ac2262de1e6014ca' "$backup_dir/Tablix.Server.dll" | sha256sum -c -
```

**仅当最后一条显示 OK，才继续下面步骤。** 此处校验应用 DLL 版本兼容性，不限制镜像 ID。校验失败说明服务器应用版本不同，请保留备份和输出，不直接替换。

以下操作会短暂停服。必须先停容器，再替换加载中的 DLL：

```bash
docker stop tablix-offline
docker cp ./Tablix.Server.dll tablix-offline:/app/Tablix.Server.dll
docker cp ./Tablix.Server.pdb tablix-offline:/app/Tablix.Server.pdb
docker start tablix-offline
docker logs --since 2m --tail 100 tablix-offline
```

仍使用原来 `http://服务器:7713/mcp`，无需重新注册数据库。若原配置将 `Logging.ConsoleLogging` 设为 false，仍应先在原配置中改为 true；调用记录需要 `MinimumSeverity` 为 0 或 1，警告异常需要不高于 2。本补丁遵守已有配置。新生产三件套默认 ConsoleLogging=true、MinimumSeverity=1。

重新调用 `tablix_discover_databases`，使用 JSON 数字 maxResults=100、skip=0，filter 留空字符串。随后执行：

```bash
docker logs --since 5m --tail 200 tablix-offline
```

应能看到类似以下内容（字段顺序可能不同）：

```text
[MCP] requestId=... tool=tablix_discover_databases started
[MCP] requestId=... tool=tablix_discover_databases failed exception=... sqliteCode=... at=...
[MCP] requestId=... tool=tablix_discover_databases completed status=failed elapsedMs=...
```

成功时则为 `status=success`。把此次错误对应的 `[MCP]` 日志用于继续定位；不要发送完整配置文件或数据库口令。如果重试后完全没有 `[MCP]`，还需要核对调用地址、实际容器、日志开关以及 Docker 日志驱动。

## 回退与持久性

备份目录不要删除。需要回退时，填写上一步实际生成的备份目录，由用户执行：

```bash
backup_dir=./backup-实际时间
docker stop tablix-offline
docker cp "$backup_dir/Tablix.Server.dll" tablix-offline:/app/Tablix.Server.dll
docker cp "$backup_dir/Tablix.Server.pdb" tablix-offline:/app/Tablix.Server.pdb
docker start tablix-offline
```

本次替换位于容器可写层：普通 stop/start 会保留，删除或重新创建容器会丢失。原完整生产镜像尚未加入此补丁；若后续确认用于正式交付，需要将其合入新镜像。配置、数据卷、端口、工具名和禁用列表均不由补丁改动。

## 验证边界

新增 6 项诊断测试已通过；已用 Linux 目标的 managed DLL 搭配原镜像 Core、依赖、deps/runtimeconfig，在 Windows 本地重定向 stdout 的进程中完成真实 MCP 握手、正常调用、模拟 SQLite 异常及文件/stdout 双路日志验证。没有增加或升级 NuGet 依赖。

当前完整回归包含一项原有的辅助项目版本号检查失败，该检查在 Theory 与聚合 Fact 中各报告一次：Tablix.Deployment.csproj 未声明 Version=0.3.1。此元数据问题在补丁前已经存在，与日志改动无关。实际计数、来源提交与发布文件哈希见 verification.json。尚未在 Linux Docker、实际 Compose 或你们的内网 MySQL 上执行验证。
