# 维护者笔记

面向改包的人。README 讲用户怎么用，这篇讲当初为什么这么做。

## 为什么选 Debian 包

厂商在 `drivers.pantum.cn` 提供三个构建，文件名都不可信，按 ELF 头判断：

| 包 | ELF 实测 | 依赖缺口 |
|---|---|---|
| `..._amd64_deb.zip` | `ELF 64-bit LSB pie executable, x86-64` | `libssl.so.1.1`、`libjpeg.so.62` |
| `..._nd7_x86_64_rpm.zip` | `ELF 64-bit LSB pie executable, x86-64` | `libssl.so.10`、`libjpeg.so.62` |
| `..._nd7_mips64el_rpm.zip` | `ELF 64-bit LSB, MIPS64 rel2` | — |

x86_64 RPM 版架构其实也对，但有两个问题：

- 它依赖 openssl **1.0**（soname `.so.10`），AUR 的 `openssl-1.0` 维护远不如 `openssl-1.1` 活跃
- 它的辅助程序装在 `/opt/pantum/`，而 filter 里 exec 的是 `/opt/SecPrinter/pantum/bin/...`，路径对不上，得额外加一段重定位

Deb 版路径天然匹配，所以选它。

## 哪些东西不能改

`pkgname` 改了，**以下路径一个都不能动**：

- `/usr/lib/cups/filter/pantum_filter_*`、`pantum_prefilter_r`、`pantum_cmdfilter` —— PPD 里是按**名字**引用的（`*cupsFilter: "application/vnd.cups-pdf 21 pantum_filter_pcl_l"`），CUPS 去 filter 目录按名查找，改名 PPD 立即失效
- `/opt/SecPrinter/pantum/` —— filter 用绝对路径 exec 厂商程序
- `/usr/share/cups/model/pantum/` —— 改成任何其他目录都行，但没必要

## 踩过的坑

### 1. `cp -a` 静默失败（最隐蔽）

```bash
# 错：$pkgdir/opt 尚不存在，带尾斜杠的 cp 什么都不做，也不报错
cp -a "$debdir/opt/SecPrinter" "$pkgdir/opt/"

# 对
install -d "$pkgdir/opt/SecPrinter"
cp -a "$debdir/opt/SecPrinter/." "$pkgdir/opt/SecPrinter/"
```

症状极具迷惑性：CUPS 里任务秒进秒出显示 `completed`，打印机完全静默，队列里看不到任何东西。CUPS 层面一切正常，因为 filter **成功输出了 0 页**。

只能在 debug 日志里看到（`LogLevel warn` 下成功任务不写日志）：

```
sh: 行 1: /opt/SecPrinter/pantum/bin/pantum_bin_gs: 没有那个文件或目录
PAGE: total 0
```

**教训**：验证包内容要验**产物**（`bsdtar -tf pkg.tar.zst | grep ...`），不能验源目录。当时只验了源 deb 里的文件在不在，就以为没问题了。

现在 PKGBUILD 里有构建期断言兜底，7 个必需文件缺任一个就让构建失败。

### 2. 全局 ld.so.conf 污染系统

厂商原始包装 `/etc/ld.so.conf.d/pantum.conf`，内容是一行：

```
/opt/SecPrinter/pantum/lib
```

那个目录里有厂商自带的 `libpng16.so.16`（2.28）。进全局缓存后排在 `/usr/lib` 前面，于是**全系统每个进程**都加载旧库而不是系统的 1.6.59。实测 Magisk 因此无法启动。

改成给厂商自己的 GUI/OCR 工具加 `RUNPATH=$ORIGIN/../lib`。`RUNPATH` 优先级低于 `LD_LIBRARY_PATH` 和 ld.so 缓存，只能在自己无路可走时生效，遮蔽不了别人。

**打印链路刻意保持零 RPATH**：`pantum_bin_gs` 内置静态链接的 Ghostscript 10.00.0，本来就不需要 `pantum/lib` 里任何东西。断言里会检查这一点。

> 不要图省事改成 `libpng16` 软链回系统版本。厂商的 `libtesseract.so.5`、`liblept.so.5`、`libopencv_*` 都依赖特定版本的 libpng，混用会产生难查的符号错误。

### 3. 下载源被 WAF 拦

`drivers.pantum.cn` 在阿里云 WAF 后面。裸请求拿到的是 3.8K 的 gzip 压缩 JS 挑战页（要算 `acw_sc__v2` cookie），不是文件：

```bash
curl -L -o deb.zip "$URL"
file deb.zip          # gzip compressed data
gzip -dc deb.zip      # <html><script>var arg1='C8F4BEC...'
```

带浏览器 UA + Referer 才是真文件：

```bash
curl -L -o deb.zip \
  -A "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" \
  -e "https://www.pantum.cn/" \
  "$URL"
```

pacman 7.1 移除了 makepkg 的 `useragent` / `referenced` 钩子，没法在 PKGBUILD 里注入请求头，所以 `source=` 干脆不写 URL，直接把 zip 收进仓库。

顺带：URL 里的 `+` 必须写成 `%2B`。

### 4. `SRCDEST` 的回落行为

`/etc/makepkg.conf` 里 `SRCDEST` 是注释掉的，`/usr/bin/makepkg` 第 1184 行：

```bash
for var in PKGDEST SRCDEST SRCPKGDEST LOGDEST BUILDDIR; do
	printf -v "$var" '%s' "$(canonicalize_path "${!var:-$startdir}")"
done
```

**未设置时默认是 `$startdir`（构建目录），不是 `~/.cache/makepkg/src`**。所以手动下载的源文件要放在 PKGBUILD 同级目录，放 `src/` 或 `~/.cache/makepkg/src/` 都会校验失败。这个反直觉，排查时浪费了不少时间。

## 空白文档不出纸

厂商 filter 会把 0 页有效内容的 PDF 静默丢弃，CUPS 仍报 `Job completed`。日志里能看到：

```
page is: 0
receive page is: 0
PAGES=0
```

容易误判成驱动没装好。这是驱动固有行为，无法在打包层修掉，只能写进文档。

## 更新驱动版本的流程

改 `_zip`、`_deb`、`pkgver` 和 `sha256sums`（`makepkg -g` 或 `sha256sum` 手动算）：

1. 先确认新版的 filter 依赖是否变了，`ldd` 逐个查
2. 如果依赖的 soname 变了（比如 openssl 1.0→1.1），要改 `depends` 里对应的 AUR 包名
3. 如果新版改用了 `/opt/pantum/` 而非 `/opt/SecPrinter/`，安装段的目录名要跟着改
4. 保留构建期断言，它是唯一能挡住「静默渲染 0 页」的防线

## 未处理

- `/opt/SecPrinter/pantum/suyuan/lib/libPtEmbed.so` 在上游 deb 里本来就不存在，filter 引用了该路径但只在 PDF/OEM 水印分支走到。纯文本和常规 PCL 打印不触发
- `pantum_smservice`（扫描/状态服务）依赖 Qt5 和 opencv，包里没声明这些依赖。纯打印用不到，服务默认不启用
- 厂商把 `libpng16.so`、`libpng16.so.16`、`libpng.so` 全放成实体文件而非符号链接，`ldconfig` 会报「不是符号链接」。无害
