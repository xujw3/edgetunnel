# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 常用命令

本仓库没有 `package.json`、构建脚本、lint 配置或测试框架；`_worker.js` 是直接部署到 Cloudflare Workers/Pages 的单文件 Worker。

```bash
# 语法检查（当前最接近“测试”的本地快速检查）
node --check _worker.js

# 本地运行 Worker（需要本机已登录 Wrangler；也可去掉 --local 使用远端资源）
npx wrangler@latest dev --local

# 部署到 Cloudflare，读取 wrangler.toml 中的 name/main/compatibility_date
npx wrangler@latest deploy

# 查看已部署 Worker 的实时日志
npx wrangler@latest tail

# 创建生产 KV 命名空间；把输出的 id 填回 wrangler.toml 的 [[kv_namespaces]]
npx wrangler@latest kv namespace create KV
```

本地开发时通常需要 `.dev.vars`（已被 `.gitignore` 忽略）提供至少 `ADMIN`，并在 `wrangler.toml` 中启用 `KV` 绑定；否则 `/admin`、`/sub`、配置保存和日志等依赖 KV 的功能无法完整工作。

当前没有单测命令，也没有“运行单个测试”的方式。改动后至少执行 `node --check _worker.js`；涉及实际 Worker 行为时再用 `npx wrangler@latest dev --local` 手工验证对应路由/协议。

## 项目结构与架构

- `_worker.js`：核心代码，单文件 Cloudflare Worker，无 import/export 依赖（除默认导出的 Worker 入口外）。代码大量使用中文标识符和分区注释，修改时保持这种命名与组织方式。
- `wrangler.toml`：Cloudflare 部署入口配置，当前 `main = "_worker.js"`，KV 绑定模板被注释，绑定名默认应为 `KV`。
- `README.md`：面向用户的部署与环境变量说明，是确认公开功能、部署方式和免责声明的主要来源。
- `CHANGELOG`：版本变更记录；`_worker.js` 顶部 `Version` 字符串需要与发布变更保持一致。
- `.github/workflows/sync.yml`：fork 自动同步上游 `cmliu/edgetunnel` 的 workflow，仅在 fork 仓库运行。
- `.github/workflows/Auto-close-empty-PRs.yml`：自动关闭说明为空或过短的 PR。

## `_worker.js` 高层流程

入口是 `export default { async fetch(request, env, ctx) { ... } }`。入口函数先规范化 URL、计算管理员密码/订阅 UUID、读取环境变量和代理默认值，然后按请求类型路由：

1. `/version`：带 UUID 校验的版本信息接口。
2. `Upgrade: websocket`：进入 `处理WS请求`，处理 WS 传输。
3. 非 admin/login 的 `POST`：根据 `Content-Type: application/grpc` 与 Referer 特征分流到 `处理gRPC请求` 或 `处理XHTTP请求`。
4. HTTP 管理/订阅路径：`/login`、`/admin`、`/admin/config.json`、`/admin/ADD.txt`、`/sub`、`/locations`、`/robots.txt` 等。
5. 其他路径：按 `env.URL` 反代伪装站点，或回退到内置 `nginx()`/`html1101()` 页面。

管理面板静态页面不在本仓库内，运行时通过 `Pages静态页面 = 'https://edt-pages.github.io'` 拉取；本仓库主要提供后端 API、订阅生成和传输处理。

## 配置与持久化

关键环境变量来自 README 和代码：`ADMIN`（必需）、`KEY`、`UUID`、`PROXYIP`、`URL`、`GO2SOCKS5`、`DEBUG`、`OFF_LOG`、`BEST_SUB`、`PRELOAD_RACE_DIAL`，代码还支持 `HOST`、`PATH` 覆盖节点域名/路径。

KV 绑定名应为 `KV`。主要 KV key：

- `config.json`：由 `读取config_JSON` 初始化、读取、补齐和规范化，是管理面板和订阅生成的主配置。
- `ADD.txt`：自定义优选 IP/节点列表。
- `tg.json`：Telegram 日志通知配置。
- `cf.json`：Cloudflare 用量查询配置。
- `log.json`：请求日志，受 `OFF_LOG` 控制。

`读取config_JSON(env, hostname, userID, UA, 重置配置)` 是理解默认配置、兼容旧配置和生成最终节点链接的中心函数。

## 传输与代理模块

- 协议解析：`解析魏烈思请求` 处理 VLESS，`解析木马请求` 处理 Trojan，SS AEAD 相关逻辑由 `SS派生主密钥`、`SSAEAD加密`、`SSAEAD解密` 等函数处理。
- 三种入站传输：`处理WS请求`、`处理XHTTP请求`、`处理gRPC请求` 都会解析首包后复用 TCP/UDP 转发逻辑。
- TCP 转发核心：`forwardataTCP` 负责直连、反代、并发拨号、预加载 DoH A/AAAA 竞速、失败兜底和下行 `connectStreams`。
- UDP：`forwardataudp` 主要用于 DNS UDP over TCP 场景；Trojan UDP 由 `转发木马UDP数据` 处理。
- 上下行队列/合包：`创建上行写入队列` 和 `创建下行Grain发送器` 控制背压、合包、顺序写入和 WebSocket frame 数量。
- 代理链路：`反代参数获取` 从 query/path 解析 `proxyip`、`socks5`、`http`、`https`、`turn`、`sstp` 和 `/video/<encoded>` 链式代理；连接函数包括 `socks5Connect`、`httpConnect`、`httpsConnect`、`turnConnect`、`sstpConnect`。
- HTTPS/TURN/SSTP 相关 TLS 由文件内自带的 `TlsClient` 及 TLS/加密辅助函数实现，不依赖外部库。

## 订阅生成

`/sub` 路由负责鉴权、识别客户端 UA/参数并生成订阅：

- 本地 mixed 链接生成会根据 `config_JSON`、`ADD.txt`、随机 IP、优选 API 和链式代理备注构造节点。
- 非 mixed 类型会调用订阅转换后端，并按需要套用 `Clash订阅配置文件热补丁`、`Singbox订阅配置文件热补丁`、`Surge订阅配置文件热补丁`。
- `BEST_SUB=true` 时存在“优选订阅生成器”模式，入口条件在 `/sub` 路由内判断。

## 修改注意点

- 这是单文件 Worker，很多全局变量（如 `反代IP`、`parsedSocks5Address`、`config_JSON`、缓存数组）会在请求流程中被读写；改动代理或传输逻辑时要检查并发请求下的共享状态影响。
- 路径和 query 参数有多种兼容写法，尤其是 `反代参数获取` 和 `读取config_JSON` 中的路径模板逻辑；新增格式时要避免破坏旧格式。
- 管理端 API 返回给远端静态前端使用，字段名和 JSON 结构需要兼容现有 `edt-pages.github.io` 前端。
- 项目无构建步骤，提交前不要引入需要打包的依赖，除非同时补齐构建/部署流程并更新本文档。
