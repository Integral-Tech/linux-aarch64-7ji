# Maintainer: 7Ji <pugokughin@gmail.com>

_desc="almost mainline but with 7Ji's patches, for Amlogic and Rockchip platforms"

pkgbase=linux-aarch64-7ji
pkgname=(
  "${pkgbase}"
  "${pkgbase}-headers"
)
pkgver='6.9.1'
pkgrel=1
arch=('aarch64')
url="https://kernel.org"
license=('GPL2')
makedepends=( # Since we don't build the doc, most of the makedeps for other linux packages are not needed here
  'kmod' 'bc' 'dtc' 'uboot-tools'
)
options=(!strip)
_srcname="linux-${pkgver}"
_sha256_patch='0e1ad0c5f2812b819da5deb453b970ab62d4c1778928200989f4e50889387e1d'
_name_patch='0001-rebase-local-changes-to-v6.9.patch.xz'
source=(
  "https://cdn.kernel.org/pub/linux/kernel/v${pkgver%%.*}.x/${_srcname}.tar.xz"
  "${_name_patch}::https://github.com/7Ji-PKGBUILDs/${pkgbase}/releases/download/assets/sha256-${_sha256_patch}-${_name_patch}"
  '0002-block-fix-length-of-strscpy.patch'
  'config'
)
sha256sums=(
  '01b414ba98fd189ecd544435caf3860ae2a790e3ec48f5aa70fdf42dc4c5c04a'
  "${_sha256_patch}"
  '9278761a71d16c48d47e7b4840eeabb31f0ac8645780b8e5d3f9f3e108a3c205'
  '3c4318b506c7ac883555c525469714df58c17e2a2f6217f2666f0e7d8d77afb9'
)

prepare() {
  cd "${_srcname}"

  echo "Patching kernel (stable patchset)..."
  xz -cdk ../"${_name_patch}" | patch -p1

  echo "Patching kernel (temporary patches)..."
  patch -p1 < ../0002-block-fix-length-of-strscpy.patch

  echo "Setting version..."
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

  # Prepare the configuration file
  cat "${srcdir}/config" > '.config'
}

build() {
  cd "${_srcname}"

  # get kernel version, which will be used later for modules
  make olddefconfig prepare
  make -s kernelrelease > version

  # Host LDFLAGS or other LDFLAGS set by makepkg/user is not helpful for building kernel: it should links nothing outside of itself
  unset LDFLAGS
  # Only need normal Image, as most Amlogic devices does not need/support Image.gz
  # Image and modules are built in the same run to make sure they're compatible with each other
  # -@ enables symbols in dtbs, so overlay is possible
  make ${MAKEFLAGS} DTC_FLAGS="-@" Image modules dtbs
}

_package() {
  pkgdesc="The Linux Kernel and module - ${_desc}"
  depends=(
    'coreutils'
    'initramfs'
    'kmod'
  )
  optdepends=(
    'uboot-legacy-initrd-hooks: to generate uboot legacy initrd images'
    'linux-firmware: firmware images needed for some devices'
    'linux-firmware-amlogic-ophub: complete firmware set for devices with Amlogic SoCs'
    'wireless-regdb: to set the correct wireless channels of your country'
  )

  cd "${_srcname}"

  # Install modules
  echo "Installing modules..."
  make INSTALL_MOD_PATH="${pkgdir}/usr" INSTALL_MOD_STRIP=1 modules_install

  # Install DTBs, not to target pkg, but in srcdir, so the later package() routine could use them
  make INSTALL_DTBS_PATH="${srcdir}/dtbs" dtbs_install

  # Install pkgbase
  local _dir_module="${pkgdir}/usr/lib/modules/$(<version)"
  echo "${pkgbase}" | install -D -m 644 /dev/stdin "${_dir_module}/pkgbase"

  # Install kernel image (this is technically not vmlinuz, but I name it this way to utilize mkinitcpio's existing hooks)
  install -Dm644 arch/arm64/boot/Image "${_dir_module}/vmlinuz"

  # Remove build and source links, which points to folders used when building (i.e. dead links)
  rm -f "${_dir_module}/"{build,source}

  # Install DTB
  echo 'Installing DTBs for Amlogic and Rockchip SoCs...'
  install -d -m 755 "${pkgdir}/boot/dtbs/${pkgbase}"
  cp -t "${pkgdir}/boot/dtbs/${pkgbase}" -a "${srcdir}/dtbs/"{amlogic,rockchip}
}

_package-headers() {
  pkgdesc="Header files and scripts for building modules for linux kernel - ${_desc}"
  
  # Mostly copied from alarm's linux-aarch64 and modified
  cd "${_srcname}"
  local _builddir="${pkgdir}/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "${_builddir}" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "${_builddir}/kernel" -m644 kernel/Makefile
  install -Dt "${_builddir}/arch/arm64" -m644 arch/arm64/Makefile
  cp -t "${_builddir}" -a scripts

  echo "Installing headers..."
  cp -t "${_builddir}" -a include
  cp -t "${_builddir}/arch/arm64" -a arch/arm64/include
  install -Dt "${_builddir}/arch/arm64/kernel" -m644 arch/arm64/kernel/asm-offsets.s


  install -Dt "${_builddir}/drivers/md" -m644 drivers/md/*.h
  install -Dt "${_builddir}/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "${_builddir}/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "${_builddir}/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "${_builddir}/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "${_builddir}/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "${_builddir}/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "${_builddir}/{}" \;

  echo "Removing unneeded architectures..."
  local _arch
  for _arch in "${_builddir}"/arch/*/; do
    [[ ${_arch} = */arm64/ ]] && continue
    echo "Removing $(basename "${_arch}")"
    rm -r "${_arch}"
  done

  echo "Removing documentation..."
  rm -r "${_builddir}/Documentation"

  echo "Removing broken symlinks..."
  find -L "${_builddir}" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "${_builddir}" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v ${STRIP_SHARED} "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v ${STRIP_STATIC} "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v ${STRIP_BINARIES} "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v ${STRIP_SHARED} "$file" ;;
    esac
  done < <(find "${_builddir}" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "${_builddir}/vmlinux"

  echo "Adding symlink..."
  mkdir -p "${pkgdir}/usr/src"
  ln -sr "${_builddir}" "$pkgdir/usr/src/$pkgbase"
}

for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done
