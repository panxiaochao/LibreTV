# AGENTS.md — LibreTV

免费在线视频聚合搜索与观看平台：Next.js 15（App Router）+ TypeScript + React 19 + Tailwind 3，单包仓库。播放内核 ArtPlayer + hls.js（点播）/ mpegts.js（HTTP-FLV 直播）/ mux.js（TS→MP4 转封装）。

## 服务端与数据边界

- **服务端无数据库**：观看历史/播放进度存浏览器 IndexedDB（`src/lib/db.ts`，Dexie）；全局设置存 localStorage（`src/lib/store.ts`，zustand persist）。API 层是无状态聚合/代理，所有“搜索聚合、什么放 401”的判断都在服务端完成。
- **鉴权无 middleware.ts**：每个 API 路由在处理函数顶部调用 `src/lib/api-guard.ts` 的 `guardRequest(req)`——未配置 `PASSWORD` 返回 503，未登录返回 401。新增 API 路由必须接入；个别例外（如 proxy 允许未登录访问部分图片类目标做豆瓣封面防盗链）在该路由内自行放行。
- **SSRF 防护必须走 `src/lib/ssrf.ts`**：服务端向远程 URL 发请求（代理、直播流、订阅拉取等）一律经过它——含 DNS 解析后校验与 IPv4-mapped IPv6 归一化；内网/保留地址默认拒绝，仅 `LIVE_ALLOW_PRIVATE=1` 时为直播流放行。新增任何“服务端拉远程地址”的逻辑时接入该模块。
- 改动源订阅行为前先读 README「数据源订阅」一节：去重规则、数量上限（点播 100/直播 50）、TS 中 TVBOX 配置与 source-catalog 裸数组的兼容解析、“订阅源以远端列表为准”的管理边界等都在那里有详尽约定，代码在 `src/lib/subscription-sync.ts`、`tvbox-parser.ts`、`source-list.ts`。

## 目录速览

- `src/app`：App Router 页面（`/` `/live` `/watch` `/about`）与 `src/app/api/*` 路由（search、detail、proxy、live/*、douban、bangumi、hot-list、source-list、auth、status、publish 等）。
- `src/lib`：纯逻辑库（CMS/M3U/TVBOX/解析与下载、测速探活、鉴权、SSRF 等），绝大多数有同名 `.test.ts`；复杂规则的用例先在测试文件里找现成行为基线。
- `src/components`：`'use client'` 组件与 hooks；`src/app/api` 同层的业务页面均在此消费。

## 常用命令

- 开发：`PASSWORD=dev-password npm run dev` → http://localhost:8080。端口是 **8080** 不是 3000；不设 `PASSWORD` 时页面/API 直接提示未配置。
- 单测：`npm test`（vitest run）。单个文件：`npx vitest run src/lib/ssrf.test.ts`。
- `npm run typecheck`（tsc --noEmit）、`npm run lint`（eslint src）。
- 路径别名 `@` → `src`。

## 测试

- 位置：`.test.ts` 与被测源文件同目录（`src/lib/*` 与 `src/app/api/**` 都有）。
- 环境：**node**，非 jsdom——只测纯逻辑与 API 路由（路由测试通过 `route.test.ts` 直接调用 GET/POST）。浏览器/组件行为没有自动化测试，勿在 vitest 中引入 jsdom 相关断言期望。
- 仓库未配置测试数据库或外部服务，一律本地运行，无需额外前置。

## 构建 / 发布

- `next.config.ts` 的 `output` 按 `DOCKER_BUILD=1` 切为 standalone，**仅用于 Docker 镜像构建**（Dockerfile）。本地构建产物不支持 `next start`；本地验证直接 `npm run build && npm start`（监听 8080）。
- 版本号以 `package.json` 为单一来源，构建时注入 `APP_VERSION`（`/api/status` 暴露）。发布流程：`npm version patch && git push && git push --tags` → `docker-publish.yml` 自动构建 GHCR 与 Docker Hub 双 registry 双架构镜像（CI 校验通过后才发布）。
- 不使用 next/image 优化（`images.unoptimized: true`，封面走 `<img>` 或代理原图）——勿引入依赖图片优化的改法。

## 约定

- 全过程使用中文：注释、文档、commit message 均为中文。
- 提交信息遵循 Conventional Commits：`feat(scope):`、`fix(scope):`、`docs:`、`ci:` 等（见 `git log`）。
- `PASSWORD` 必填；`.env` 已在仓库本地存在但不得提交敏感值。
- 仓库未配置 monorepo，无需 workspace 处理。
