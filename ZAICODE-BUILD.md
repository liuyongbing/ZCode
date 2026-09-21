# ZAICode 构建指南

ZAICode 是基于 [ZCode](https://github.com/zai-org/zcode)(Apache-2.0,v3.14.0)的自有品牌构建。
本文记录构建环境、标准流程、以及本机构建时遇到的存储层问题与修复流水线。

## 品牌改造范围

已改为 ZAICode(桌面应用为 `packages/desktop`):

- 产品身份:`packages/desktop/scripts/desktop-product-identity.mjs`
  - `productName: "ZAICode"`(安装包名、exe 名、安装目录都由它驱动)
  - `appId: "com.zaicode.app"`(与官方 `dev.zcode.app` 区分,可并排安装,互不覆盖快捷方式)
- 运行时身份:`packages/desktop/src/main/desktopRuntimeEnv.ts`(`app.setName`、进程名、userData 目录 `%APPDATA%/ZAICode`)
- 窗口标题、托盘、菜单、关于窗口、欢迎页、i18n 全部文案(zh-CN/en-US)
- Web 版标题与登录页、会话分享页
- CA 证书主体名、OpenRouter 归属头、安装目录校验清单(新增 ZAICode 路径)

**保持不变**(有意为之,保证与上游兼容、便于合并更新):

- npm 作用域 `@zcode/*`、根包名 `zcode`(内部标识符,改名收益为零、风险大)
- 环境变量前缀 `ZCODE_*`、数据目录 `~/.zcode`、CLI 命令名 `zcode`
- 应用图标/Logo 仍为 ZCode 原图(如需自有图标,替换 `packages/desktop/build/icon*.ico` 与 `public/logo/` 后重打包)

## 构建环境

- Node **24.14.0** 便携版:`D:\zcode\zcode.win.nadaona.com\.tools\node-v24.14.0-win-x64\`
- 环境脚本:`source /d/zcode/zcode.win.nadaona.com/.tools/env.sh`
  - 自动前置 Node24、pnpm 10.33.2、npmmirror registry、ELECTRON_MIRROR
  - GitHub 直连超时,需走本机代理 `127.0.0.1:7890`(git clone/fetch 时加
    `-c http.proxy=http://127.0.0.1:7890`)

## 标准构建流程(机器健康时)

```bash
source /d/zcode/zcode.win.nadaona.com/.tools/env.sh
cd /d/zcode/zcode.win.nadaona.com/zcode
pnpm install          # 约 10 分钟
pnpm bootstrap        # 准备桌面运行资源 + 构建各子包
ZCODE_ENV=production pnpm bundle:desktop -- --os win --arch x64
# 产物: packages/desktop/dist/ZAICode-<version>-win-x64.exe
```

`ZCODE_ENV=production` 必须带:不带时构建的是 Preview 身份(连测试后端、产物带 `_TEST` 后缀)。
electron-builder 二进制下载走国内镜像需再导出
`ELECTRON_BUILDER_BINARIES_MIRROR=https://registry.npmmirror.com/-/binary/electron-builder-binaries/`。

## ⚠️ 本机存储层会静默损坏大文件(2026-09-21 实测)

本次构建中,electron zip 下载、win-unpacked 解包、app.asar 打包多次出现
**同样大小的位级损坏**(文件大小不变、内容变全零/乱码),普通写后读回(走缓存)校验不出来。
C:、D: 两盘(均 91% 占用)都出现过。特征:构建莫名失败、`SyntaxError: Unexpected token`、
`Cannot find module`、md5 与源不一致。

若怀疑产物损坏,用下面的校验与重建流水线(均在 `.tools/`):

| 脚本 | 用途 |
| --- | --- |
| `scan-syntax.mjs <目录>` | 语法级扫描目录内全部 JS(找损坏文件) |
| `build-asar-final.mjs` | 从 repo 原件重建 app.asar(三读投票防读损坏、写入 C: 盘、补齐 electron-builder 漏打包的 workspace 包/react 等、fsync、抽样回读校验) |
| `deploy-asar.mjs` | 把 C: 盘验证过的 asar 拷回 win-unpacked 并跨卷 md5 复核 |
| `check-pkg-mains.mjs` | 检查 asar 内每个包的 main 是否可解析 |

重建 + 部署一条龙:

```bash
source /d/zcode/zcode.win.nadaona.com/.tools/env.sh
node --experimental-vm-modules /d/zcode/zcode.win.nadaona.com/.tools/build-asar-final.mjs
node /d/zcode/zcode.win.nadaona.com/.tools/deploy-asar.mjs
```

注:`build-asar-final.mjs` 同时修复了本机 electron-builder 依赖收集不完整的问题
(它漏打包 `@zcode/*` workspace 包、`react`、`@lydell/node-pty-linux-*`、
`@modelcontextprotocol/sdk/dist` 等;脚本会从磁盘真实文件补齐)。

## 交付物

- 便携版:`win-unpacked/`(已验证启动,可直接运行 `ZAICode.exe`)及工作区根目录的
  `ZAICode-3.14.0-win-x64-portable.zip`
- 安装包:`pnpm bundle:desktop` 产出的 NSIS exe(注意:在存储问题解决前,安装包内嵌的
  app.asar 未经验证,建议先在健康环境重打包,或用上面的流水线验证后再分发)

## 已知事项

1. **自动更新**:客户端更新清单来自官方服务端(读 `ZCODE_CDN_BASE_URL` 等)。若对外分发
   ZAICode,需考虑固定该地址或屏蔽更新,否则可能被官方包覆盖回 ZCode。
2. ssh2 的原生加密模块因缺 VS C++ 构建工具被跳过,自动回退纯 JS 实现,不影响功能
   (需要时安装 "Desktop development with C++" 工作负载后重装依赖)。
3. Windows 未签名:分发时 SmartScreen 会提示"未知发布者",正式分发需购买代码签名证书。
4. 拉取上游更新:`git -c http.proxy=http://127.0.0.1:7890 fetch origin && git merge origin/main`,
   品牌改动集中在上述文件,冲突面很小。
