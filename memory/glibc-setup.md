---
name: glibc-setup
description: 在 Ubuntu 20.04 系统默认 glibc 2.31 上，两种使用高版本 glibc 的方案
metadata: 
  node_type: memory
  type: reference
  originSessionId: 6f7607d4-ebee-48c2-84ec-3f8131418866
---

## 系统环境

- 系统 glibc 2.31 (Ubuntu 20.04)
- nix 2.18.4（不能用 2.34+，因 `/usr/local/lib64/libstdc++.so.6` 来自 GCC 13.2.0 手动编译，只支持 CXXABI_1.3.14）
- nix glibc 路径：`/nix/store/fjkx1l5cnskzrqacf08z7i8z17256w0j-glibc-2.42-61/`

## 方案对比

| | patchelf | ld-linux wrapper |
|------|----------|-----------------|
| 原理 | 修改目标二进制的 ELF interpreter + RUNPATH | 用 nix 的 ld-linux 直接启动目标 |
| /proc/self/exe | 正确指向目标程序 | 指向 ld-linux，process.execPath 异常 |
| npm update 影响 | 不受影响（只改了 node） | 会覆盖 wrapper |
| 作用范围 | 整个 node 版本 | 仅该 wrapper 入口 |
| 恢复方式 | `nvm reinstall` | 删除 wrapper 即可 |

## 方案一：patchelf

```bash
patchelf --set-interpreter /nix/store/fjkx1l5cnskzrqacf08z7i8z17256w0j-glibc-2.42-61/lib/ld-linux-x86-64.so.2 \
  --set-rpath /nix/store/fjkx1l5cnskzrqacf08z7i8z17256w0j-glibc-2.42-61/lib:/usr/local/lib64:/usr/lib/x86_64-linux-gnu \
  /root/.nvm/versions/node/v22.22.2/bin/node
```

- nix glibc 路径没有 libstdc++/libgcc_s，需 RUNPATH 中追加 `/usr/local/lib64` 和 `/usr/lib/x86_64-linux-gnu`
- `nvm reinstall` 会覆盖，需重新执行

## 方案二：ld-linux wrapper

```bash
#!/bin/bash
export NODE_OPTIONS="${NODE_OPTIONS:+$NODE_OPTIONS }--max-old-space-size=8192"
exec /nix/store/fjkx1l5cnskzrqacf08z7i8z17256w0j-glibc-2.42-61/lib/ld-linux-x86-64.so.2 \
  --library-path /nix/store/fjkx1l5cnskzrqacf08z7i8z17256w0j-glibc-2.42-61/lib:/usr/local/lib64:/usr/lib/x86_64-linux-gnu \
  /root/.nvm/versions/node/v22.22.2/bin/node \
  /root/.nvm/versions/node/v22.22.2/bin/gitnexus.orig "$@"
```

- `gitnexus.orig` 是符号链接指向 `../lib/node_modules/gitnexus/dist/cli/index.js`
- `/proc/self/exe` 指向 ld-linux，程序内部调用 `process.execPath` 时会拿到 ld-linux 路径
- `NODE_OPTIONS` 预设是防止 gitnexus 的 `ensureHeap()` 用 `process.execPath` 重新启动时把 `--max-old-space-size` 传给 ld-linux
- `npm update -g` 会覆盖 wrapper

### wrapper 副作用：`process.execPath` 异常 → gitnexus FTS 安装失败

gitnexus 的 `dist/core/lbug/extension-loader.js` 用 `spawn(process.execPath, [install-duckdb-extension.mjs, fts, <maxDbSize>])` 起子进程装 LadybugDB 扩展。wrapper 下 `process.execPath` = ld-linux，于是实际执行 `ld-linux install.mjs` → ld-linux 把 .mjs 当 ELF 加载 → `invalid ELF header`，FTS 装不上（`Full-text search: unavailable`，BM25 全文搜索降级，query 返回空）。

第二层障碍：即使绕过 spawn 手动跑 `ld-linux node install.mjs fts <size>`，LadybugDB 内部 `INSTALL fts` 的 HTTP 下载器**不读系统 `https_proxy`**，直连 `extension.ladybugdb.com`（Cloudflare）超时（国内）。

修复（两层都绕过 —— curl 走代理下载 + 放到 LOAD 能命中的本地目录）：

1. 走代理下载 FTS 扩展（版本必须对齐 gitnexus 当前用的 LadybugDB 版本，看 `~/.lbdb/extension/` 下现有旧版本目录推断当前版本号，此处 v0.17.0）：
   `curl -C - -sSL -x http://127.0.0.1:7890 -o /tmp/libfts.lbug_extension https://extension.ladybugdb.com/v0.17.0/linux_amd64/fts/libfts.lbug_extension`
2. 放到 LadybugDB extension 目录（结构 `~/.lbdb/extension/<ver>/<platform>/fts/libfts.lbug_extension`）：
   `mkdir -p ~/.lbdb/extension/0.17.0/linux_amd64/fts && cp /tmp/libfts.lbug_extension ~/.lbdb/extension/0.17.0/linux_amd64/fts/`
3. gitnexus `LOAD EXTENSION fts` 命中本地文件（不再走坏掉的 spawn INSTALL），跑 `gitnexus analyze --repair-fts` 重建 FTS 索引。
4. 验证：`gitnexus doctor` → `Full-text search: available`。

> 这和 `ensureHeap()` 重启是同一类 `process.execPath` 副作用 —— 只要 gitnexus 用 `spawn(process.execPath, …)` 起任何子进程跑 `.mjs`/`.js`，在 wrapper 下都会失败。根治需 `process.execPath` 指回 node（即 patchelf 方案），但 patchelf 对 124MB node 会 segfault（见方案一），故只能逐个绕过。

## 为什么不选其他方案

| 方案 | 原因 |
|------|------|
| update-alternatives | 不支持 glibc |
| apt | Ubuntu 20.04 只有 2.31 |
| 升级 gcc | 不影响运行时 glibc |
| 手动指定链接 | 仅适用于源码编译 |
| Docker | 太重 |
