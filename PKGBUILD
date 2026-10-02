# Based on the official Pantum CUPS/SANE driver
# (pantum_7.4.170-1+1nfs1+sv_amd64.deb, Zhuhai Pantum Electronics Co., Ltd)

pkgname=pantum-cp2510dn-driver
pkgver=7.4.170
pkgrel=1
pkgdesc='CUPS driver for Pantum CP2510DN and related Pantum printers'
arch=('x86_64')
url='https://www.pantum.com'
license=('custom')
depends=('cups' 'openssl-1.1' 'libjpeg6-turbo-bin')
optdepends=('sane: scanning support'
            'avahi: printer discovery'
            'ghostscript: PDF rendering for print jobs')
makedepends=('patchelf')
replaces=('pantum')
obsoletes=('pantum')
_zip=pantum_7_4_170-1+1nfs1+sv_amd64_deb.zip
_deb=pantum_7.4.170-1+1nfs1+sv_amd64.deb
source=("$_zip" "LICENSE")
sha256sums=('c8c33cb4765d3f7b554d5f219371c518b2665f1374cd97427d88d8d2d2d47555' 'SKIP')
noextract=("$_zip")

package() {
  local d="$srcdir/deb"
  mkdir -p "$d"

  bsdtar -C "$srcdir" -xf "$srcdir/$_zip"
  bsdtar -C "$d" -xf "$srcdir/$_deb" data.tar.xz
  bsdtar -C "$d" -xf "$d/data.tar.xz"

  install -Dm755 "$d/usr/lib/cups/filter/"* -t "$pkgdir/usr/lib/cups/filter/"
  install -Dm644 "$d/usr/share/cups/model/pantum/"*.ppd \
    -t "$pkgdir/usr/share/cups/model/pantum/"

  # The filters exec /opt/SecPrinter/pantum/... by absolute path, so this
  # tree must land there verbatim.
  install -d "$pkgdir/opt/SecPrinter"
  cp -a "$d/opt/SecPrinter/." "$pkgdir/opt/SecPrinter/"

  install -Dm644 "$d/etc/udev/rules.d/60-pantum_mfp.rules" \
    -t "$pkgdir/etc/udev/rules.d/"
  install -Dm644 "$d/lib/systemd/system/pantum_smservice.service" \
    -t "$pkgdir/usr/lib/systemd/system/"

  # Do NOT ship /etc/ld.so.conf.d/pantum.conf from upstream. That adds
  # /opt/SecPrinter/pantum/lib to the global search path, where the
  # vendor's libpng16.so.16 (2.28) outranks the system one (1.6.59).
  # Every process on the box then loads the old library, which breaks
  # unrelated software (observed: Magisk failing to start). Give the
  # vendor binaries their own RUNPATH instead; RUNPATH is searched
  # after LD_LIBRARY_PATH and the ld.so cache, so it cannot shadow
  # system libraries for anyone else.
  # pantum_bin_gs and pantum_bin_status are part of the printing path and
  # resolve everything from the system, so they must stay RPATH-free.
  # The rest are the vendor's GUI/OCR helpers, which need their bundled
  # libpng16 2.28, leptonica, tesseract and opencv.
  local f
  for f in "$pkgdir"/opt/SecPrinter/pantum/bin/*; do
    strip --strip-unneeded "$f" 2>/dev/null || true
    case $(basename "$f") in
      pantum_bin_gs|pantum_bin_status|pantum_bin_status_r) ;;
      *) patchelf --set-rpath '$ORIGIN/../lib' "$f" 2>/dev/null || true ;;
    esac
  done
  for f in "$pkgdir"/opt/SecPrinter/pantum/lib/*.so*; do
    patchelf --set-rpath '$ORIGIN' "$f" 2>/dev/null || true
  done

  # Upstream installs scan backends under /usr/local, which pacman ignores.
  if [ -d "$d/usr/local/lib/sane" ]; then
    install -Dm755 "$d/usr/local/lib/sane/"*.so.* -t "$pkgdir/usr/lib/sane/"
    for f in "$d/usr/local/lib/sane/"*.so; do
      ln -sf "$(basename "$f")" "$pkgdir/usr/lib/sane/$(basename "$f")"
    done
  fi
  if [ -d "$d/usr/local/etc/sane.d" ]; then
    cp -a "$d/usr/local/etc/sane.d/." "$pkgdir/etc/sane.d/"
  fi

  install -Dm644 LICENSE "$pkgdir/usr/share/licenses/$pkgname/LICENSE"

  # A missing /opt/SecPrinter yields a package that silently renders empty
  # pages, so fail the build instead of shipping it. The printing path
  # must also stay free of RUNPATH so it only ever uses system libraries.
  local f missing=()
  for f in \
    usr/lib/cups/filter/pantum_filter_pcl_l \
    usr/lib/cups/filter/pantum_prefilter_r \
    usr/lib/cups/filter/pantum_cmdfilter \
    usr/share/cups/model/pantum/Pantum_CP2510DN_Series_PCL.ppd \
    opt/SecPrinter/pantum/bin/pantum_bin_gs \
    opt/SecPrinter/pantum/bin/pantum_bin_status \
    opt/SecPrinter/pantum/etc/pdfpageinfo.ps; do
    [ -e "$pkgdir/$f" ] || missing+=("$f")
  done
  if (( ${#missing[@]} )); then
    printf 'ERROR: missing required files:\n' >&2
    printf '  %s\n' "${missing[@]}" >&2
    return 1
  fi

  for f in "$pkgdir"/usr/lib/cups/filter/pantum_* \
           "$pkgdir"/opt/SecPrinter/pantum/bin/pantum_bin_gs \
           "$pkgdir"/opt/SecPrinter/pantum/bin/pantum_bin_status; do
    if objdump -p "$f" 2>/dev/null | grep -qE 'RUNPATH|RPATH'; then
      printf 'ERROR: %s carries an RPATH and may shadow system libs\n' "$f" >&2
      return 1
    fi
  done
}
