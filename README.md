# pantum-aur

Arch Linux 的 Pantum（奔图）打印机驱动打包项目。

把厂商官方的 Debian 包重打成 Arch 的 pacman 包，无需 `dpkg`，
仓库内自带源文件，`git clone` 后可直接 `makepkg -s`。

## 构建

```bash
paru -S --needed base-devel
git clone <此仓库>
cd pantum-aur
makepkg -s
sudo pacman -U pantum-7.4.170-1-x86_64.pkg.tar.zst
```

`makepkg -s` 会自动装依赖：`cups`、`openssl-1.1`、`libjpeg6-turbo-bin`。

## 添加打印机

先确认你的型号有对应 PPD：

```bash
ls /usr/share/cups/model/pantum/ | grep -i <型号关键字>
```

然后添加队列（把 IP 和 PPD 换成你自己的）：

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

## 支持的型号

包内含厂商提供的 45 个 PPD，涵盖 BM / BP / CM / CP / P / M 等系列。
具体型号列表：

```bash
ls /usr/share/cups/model/pantum/
```

## 已知问题

**安装后报 ldconfig 警告**（无害）：

```
ldconfig: /opt/SecPrinter/pantum/lib/libpng16.so.16 不是符号链接
```

厂商包把 `libpng16.so` 和 `libpng16.so.16` 都放成了实体文件。

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

## 源文件说明

`pantum_7_4_170-1+1nfs1+sv_amd64_deb.zip` 是厂商官方包
（Zhuhai Pantum Electronics Co., Ltd），取自
`drivers.pantum.cn`。**直接访问该站点的下载链接会拿到一个 3.8K 的
JavaScript 反爬挑战页而非文件**，需要浏览器 UA 和 Referer 才能下载，
所以此处直接收录源文件，让构建过程无需联网。

厂商还提供 x86_64 和 mips64el 的 RPM 版本，本项目未采用：x86_64 版
依赖 openssl 1.0 且其辅助程序装在 `/opt/pantum/`，与 filter 里硬编码的
`/opt/SecPrinter/pantum/` 路径不符；mips64el 版架构不匹配。

## 许可

二进制部分版权归珠海奔图电子有限公司所有，本项目仅做重新打包与路径修正。
详见 `LICENSE`。
