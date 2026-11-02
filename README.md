# qt-build

## 换版本

改 `env:` 里的这几项，**必须一起改**：

| 变量 | 说明 |
|---|---|
| `QT_VERSION` | 同时决定源码目录名和下载 URL |
| `QT_SRC_MD5` | `.../qt/<major>.<minor>/<ver>/single/md5sums.txt` 里的值，Qt 只提供 MD5 |
| `GCC_VERSION` / `MINGW_URL` | 版本号和下载地址 |
| `CCACHE_VERSION` / `CCACHE_URL` | 同上 |
| `CACHE_EPOCH` | 改一下可以整体作废旧缓存（源码 + ccache） |
