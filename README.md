# pantum-cp2510dn-driver-aur

Arch Linux 的 Pantum（奔图）CP2510DN 驱动打包项目。

把厂商官方的 Debian 包重打成 Arch 的 pacman 包，无需 `dpkg`。
仓库内自带源文件，`git clone` 后可直接 `makepkg -s`。

## 构建

```bash
paru -S --needed base-devel patchelf
git clone git@github.com:whosa/pantum-cp2510dn-driver-aur.git
cd pantum-cp2510dn-driver-aur
makepkg -s
sudo pacman -U pantum-cp2510dn-driver-7.4.170-1-x86_64.pkg.tar.zst
```

`makepkg -s` 会自动装依赖：`cups`、`openssl-1.1`、`libjpeg6-turbo-bin`，
构建期需要 `patchelf`。

## 添加打印机

先确认你的型号有对应 PPD：

```bash
ls /usr/share/cups/model/pantum/ | grep -i <型号关键字>
```

添加队列（把 IP 和 PPD 换成你自己的）：

```bash
sudo lpadmin -p pantum -E \
  -v socket://192.168.0.108:9100 \
  -m /usr/share/cups/model/pantum/Pantum_CP2510DN_Series_PCL.ppd
sudo lpadmin -d pantum
```

端口 `9100` 是 JetDirect 原始协议。多数 Pantum 机型的 IPP 端口（631）
是关闭的，所以**不能**依赖自动发现，请手动填 `socket://` 地址。

图形化配置用 `system-config-printer`（KDE 的 `print-manager` 只管打印队列，
不能添加打印机）。

## 从旧的 pantum 包升级

本包取代早期的 `pantum` 包，已声明 `replaces` 和 `obsoletes`，
直接安装会自动卸载旧包并接管文件：

```bash
sudo pacman -U pantum-cp2510dn-driver-7.4.170-1-x86_64.pkg.tar.zst
```

注意：只改了 package 名，CUPS filter、PPD、`/opt/SecPrinter` 下的文件
路径都保持原样（PPD 内以 filter 名引用，改名会失效），所以已配置的
打印队列不受影响。

## 支持的型号

包内含厂商提供的 45 个 PPD，涵盖 BM / BP / CM / CP / P / M 等系列：

```bash
ls /usr/share/cups/model/pantum/
```

## 关于库路径（重要）

厂商原始包会安装 `/etc/ld.so.conf.d/pantum.conf`，把
`/opt/SecPrinter/pantum/lib` 加进**全局**库搜索路径。本包**不装这个文件**。

原因：那个目录里有厂商自带的 `libpng16.so.16`（2.28），一旦进入全局缓存
并排在 `/usr/lib` 前面，**系统上所有程序**都会加载这个旧版本，而不是系统的
1.6.59。实测会导致无关软件崩溃（例如 Magisk 无法启动）。

本包的做法是给厂商的 GUI / OCR 辅助工具单独加 `RUNPATH=$ORIGIN/../lib`。
`RUNPATH` 的搜索优先级低于 `LD_LIBRARY_PATH` 和 ld.so 缓存，所以：

- Pantum 自己的工具继续用厂商的 libpng16 2.28、leptonica、tesseract、opencv
- 你的系统和其他程序不受任何影响，仍然用系统库

**打印链路**（三个 filter 加 `pantum_bin_gs`）**不设任何 RPATH**，
只依赖系统库。`pantum_bin_gs` 内置了静态链接的 Ghostscript 10.00.0，
本身不依赖 `pantum/lib` 里的任何东西。

如果你从旧版本（`pantum` pkgrel 1，曾带全局 `pantum.conf`）升级过来，
确认该文件已被移除：

```bash
ls /etc/ld.so.conf.d/pantum.conf 2>&1   # 应为「没有那个文件或目录」
sudo ldconfig
ldconfig -p | grep libpng16              # 应指向 /usr/lib/
```

## 已知问题

**打印任务显示完成但打印机不出纸**：多半是 `/opt/SecPrinter/pantum/bin/`
下的 Ghostscript 辅助程序缺失。检查：

```bash
ls /opt/SecPrinter/pantum/bin/pantum_bin_gs
```

**排障**：任务成功时不写日志，需要临时开 debug（**会出纸**）：

```bash
sudo cupsctl --debug-logging
lp -d pantum /path/to/file
sudo tail -40 /var/log/cups/error_log
sudo cupsctl --no-debug-logging
```

正常日志里应出现 `page count = 1`；若是 `total 0` 就是 filter 没渲染出内容。

**厂商的扫描/状态管理服务**（`pantum_smservice`）依赖 Qt5 和 opencv，
本包未声明这些依赖。纯打印不需要它，服务默认不启用。

## 源文件说明

`pantum_7_4_170-1+1nfs1+sv_amd64_deb.zip` 是厂商官方包
（Zhuhai Pantum Electronics Co., Ltd），取自 `drivers.pantum.cn`。
**直接访问该站点的下载链接会拿到一个 3.8K 的 JavaScript 反爬挑战页而非
文件**，需要浏览器 UA 和 Referer 才能下载，所以此处直接收录源文件，
让构建过程无需联网。

厂商还提供 x86_64 和 mips64el 的 RPM 版本，本项目未采用：x86_64 版
依赖 openssl 1.0，且其辅助程序装在 `/opt/pantum/`，与 filter 里硬编码的
`/opt/SecPrinter/pantum/` 路径不符；mips64el 版架构不匹配。

## 许可

二进制部分版权归珠海奔图电子有限公司所有，本项目仅做重新打包、路径修正
和 RUNPATH 设置。详见 `LICENSE`。
