# r2/r4 → r5 SQL与返回行数日志小镜像更新包

仅更新Server DLL/PDB，可用于已有r2或r4，复用其与r2一致的Core、引导、驱动、配置和挂载。记录SQL、实际库名、可读名称、真实返回行数、耗时、状态及具体Error；凭据继续脱敏；不恢复撤回的r3整数兼容改动。无需联网或宿主机Python/.NET SDK。

所有Docker操作由用户本人在测试环境执行。解压到独立补丁目录，不放入真实.env、data或业务文件。容器名/镜像名如已自定义，需要对应替换。

## 1. 校验

```bash
sha256sum -c SHA256SUMS
docker exec tablix-offline sha256sum /app/Tablix.Core.dll /app/Tablix.Deployment.dll
```

Core应为 `8283cf1861c48b370d7388191cef234ebf9519b684cd4d340acc9406b157344d`；引导应为 `604c10042e8b5086266324c3072079c5e0ec7aa64b76562aa9dac33e44f71984`。这是组件兼容性检查，不是固定旧镜像ID。不同则先核对版本，避免混用依赖。

## 2. 保存实际运行镜像，再离线构建

```bash
docker tag "$(docker inspect --format '{{.Image}}' tablix-offline)" tablix-before-r5:local
DOCKER_BUILDKIT=0 docker build --network=none --pull=false -t tablix-server:release-20261031 .
```

基础镜像就是刚保存的本地镜像。Dockerfile没有RUN/下载/安装，不编译源码。Docker28的legacy builder可能给出弃用提示；制作端未实际运行Docker构建。不要重复覆盖备份标签；它用于回退。构建成功不会自动替换运行容器。

## 3. 回原服务部署目录更新

保留原Compose、生产.env、data和logs，以及此前已验收的user/network_mode等定制配置。先检查Compose仍有 `./logs:/app/logs` 或正确的绝对宿主机日志目录映射。

```bash
docker-compose up -d --force-recreate tablix
docker-compose ps
docker logs --since 2m --tail 100 -f tablix-offline
```

不要在补丁目录执行Compose。重建会短暂中断MCP。文件日志按日期写入实际宿主机logs目录，容器重建不删除；当前不自动清理应用文件日志。实际位置可查：

```bash
docker inspect tablix-offline --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

## 4. 验收

等待服务healthy。调用正常SELECT，再对已知测试表查询一个不存在的字段，核对日志中的databaseName/databaseLabel、sql、rowsReturned；正常返回0行必须记录0。查询失败记录具体Error及rowsReturned=unknown。Docker日志和宿主机logs都应有这些信息，凭据位置显示[redacted]。重启后旧日志应仍在，新调用继续追加。

本地凭据读取失败时，日志隐藏错误正文并注明credential lookup failed，保留异常类型和数值码；正常业务SQL失败仍打印具体原因。单段正文最多16384字符，超出显式标记截断。

先完成测试MySQL/Agent、实际Docker和REST隔离验收，再导出同一镜像发生产：

```bash
docker save -o tablix-server-release-20261031.tar tablix-server:release-20261031
```

生产三件套仍是上面的tar、原Compose、生产.env；不复制测试data/logs。制作端无Docker daemon，验证仅包含真实本机MCP回环、临时SQLite、日志重启保留和镜像归档校验。

## 回退

```bash
docker tag tablix-before-r5:local tablix-server:release-20261031
```

然后在原部署目录执行 `docker-compose up -d --force-recreate tablix`。保留生产.env、data、logs；无状态表结构迁移。

## Context

库表Context可以由运维通过容器内部REST维护，保存在宿主机data/tablix.db，不需要重新打镜像。MCP写工具和对外REST仍禁用。仅搬镜像不会携带测试Context。
