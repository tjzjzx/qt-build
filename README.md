# qt-build

用 GitHub Actions 的 `windows-latest` 从零构建 **Qt 6.12.0 + MinGW-w64 GCC 16.2.0**，产物发布为 Release 资产。

单个 job 最长只有 6 小时，全量 Qt 编不完，所以做成"分块 + 自动接力"：每块跑 260 分钟编译，把 ccache 存下来，然后自动派发下一块，直到装出完整 Qt 后打包发布。

## 怎么用

**方式一：手动触发（推荐）**

Actions → `Build Qt (MinGW GCC, Auto-Chained)` → Run workflow，参数全部留默认即可。

**方式二：推 tag**

```bash
git tag v1.0.0 && git push origin v1.0.0
```

> `push` 只匹配 `v*`。仓库里现存的 `6.12.0` 这类 tag 不会触发。

## 参数

| 输入 | 默认 | 说明 |
|---|---|---|
| `run_id` | 空 | 所有分块共享的逻辑 ID，**首次运行留空**，接力时自动传递 |
| `chunk` | `1` | 分块序号，接力时自动 +1 |
| `release_tag` | `qt-6.12.0-mingw-gcc16.2` | 发布到哪个 tag；推 tag 触发时自动用该 tag |

## 产物

Release 资产：`qt-6.12.0-mingw-gcc16.2.0.7z`，解压后是 `qt-install/`（`-prefix` 指定的安装前缀）。

私有仓库的 Release 资产需要带 token 下载：

```bash
gh release download qt-6.12.0-mingw-gcc16.2 --pattern "*.7z"
```

## 工作方式

```
checkout → 装 Python/Perl/7z 工具 → 下载 MinGW GCC 16.2
        → 恢复 Qt 源码缓存（固定 key，内容不变）
        → 恢复 ccache（key 每块一个，restore-keys 回溯最近一块）
        → configure（-skip qtwebengine）
        → cmake --build（260 分钟，ccache 加速）
        → 构建成功才 install → 校验 qt-install/bin/Qt6Core.dll 存在
        → 打包 7z → 发 Release
        → 保存 ccache
        → relay job 派发下一块
```

几个关键设计：

- **缓存只放源码 + ccache，不放 `build/`**。Qt 构建树几十 GB，必然撑爆仓库 10 GB 缓存上限；ccache 压缩后 1~3 GB。
- **ccache 的 key 每块都不同**。GitHub 的缓存条目不可覆盖，用固定 key 会导致第一块之后的所有进度都存不进去。
- **完成判定靠"build 退出码 0 + install 成功 + `Qt6Core.dll` 存在"**，不是在 build 目录里搜 dll（qtbase 一编完就会命中，会发布半成品）。
- **relay 是独立 job**，等 ccache 保存完才派发，避免下一块恢复到的还是旧快照。

## 换版本

改 `env:` 里的这几项，**必须一起改**：

| 变量 | 说明 |
|---|---|
| `QT_VERSION` | 同时决定源码目录名和下载 URL |
| `QT_SRC_MD5` | `.../qt/<major>.<minor>/<ver>/single/md5sums.txt` 里的值，Qt 只提供 MD5 |
| `GCC_VERSION` / `MINGW_URL` | 版本号和下载地址 |
| `CCACHE_VERSION` / `CCACHE_URL` | 同上 |
| `CACHE_EPOCH` | 改一下可以整体作废旧缓存（源码 + ccache） |

## 已知限制

- `windows-latest` 是 4 核，全量 Qt + GCC 大约 8~15 小时，正常情况下需要 2~4 块。每块开头要把已编过的 TU 从 ccache 重放一遍（`moc`/`rcc`/链接不走 ccache），约占掉 15~40 分钟。
- Qt 6.12 官方没有 GCC 16 的 CI 覆盖，可能出现新的编译错误；`-no-warnings-are-errors` 只能挡 warning。
- `MAX_CHUNKS=12` 是保险丝：跑满 12 块还没编完就直接失败，不会无限烧 Actions 额度。
- 缓存上限 10 GB/仓库；超了保存会失败（`continue-on-error`，只打 warning）。

## 排障

- 每块日志里都有 `ccache stats`。`Cacheable calls` 为 0 或 `Direct` 命中率为 0 都说明 ccache 没接上。
- 下载损坏/被截断会在 `Qt source checksum mismatch` 处立刻失败，不用等解压。
- Release 创建失败时，7z 会作为 Actions Artifact 保留 7 天，不用重跑。
