# Maintainer: Levente Polyak <anthraxx[at]archlinux[dot]org>
# Contributor: Daniel Micay <danielmicay@gmail.com>
# Contributor: Tobias Powalowski <tpowa@archlinux.org>
# Contributor: Thomas Baechler <thomas@archlinux.org>

pkgbase=linux-slo2
_pkgver=5.1.14
_commithash=f2711055d227196aae5c05097b853d0848c21d75
_srcname=$pkgbase
pkgname=$pkgbase
pkgver=${_pkgver}
pkgrel=1
pkgdesc="The ${pkgbase/linux/Linux} kernel and modules"
depends=(coreutils linux-firmware kmod mkinitcpio)
optdepends=('crda: to set the correct wireless channels of your country'
            'usbctl: deny_new_usb control')
makedepends=('xmlto' 'kmod' 'inetutils' 'bc' 'libelf' 'git' 'dtc')
provides=('WIREGUARD-MODULE' 'kernel26' "linux=${pkgver}")
replaces=('linux-armv8')
conflicts=('linux')
backup=("etc/mkinitcpio.d/$pkgbase.preset")
install=linux.install
url='https://github.com/angelsl/linux'
arch=('aarch64')
license=('GPL2')
options=('!strip')
source=(config  # the main kernel config files
        60-linux.hook  # pacman hook for depmod
        90-linux.hook  # pacman hook for initramfs regeneration
        linux.preset   # standard config files for mkinitcpio ramdisk
)
sha256sums=('6a0043a85cb3bccb5722ad6de1104b98c9f1dc1ddb12a73adaf6092c39ada2bd'
            'ae2e95db94ef7176207c690224169594d49445e04249d2499e9d2fbc117a0b21'
            '71df1b18a3885b151a3b9d926a91936da2acc90d5e27f1ad326745779cd3759d'
            '6837b3e2152f142f3fff595c6cbd03423f6e7b8d525aac8ae3eb3b58392bd255')

_kernelname=${pkgbase#linux}
: ${_kernelname:=-slo2}

_makecross='ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-'

prepare() {
  mkdir -p $_srcname && cd $_srcname
  git init
  git fetch --depth=1 https://github.com/angelsl/linux.git $_commithash
  git checkout $_commithash
  git clean -xfd

  msg2 "Setting version..."
  # sed -i "s#/sbin/depmod#depmod#" Makefile
  scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "$_kernelname" > localversion.20-pkgname

  msg2 "Setting config..."
  cp ../config .config
  make $_makecross oldconfig

  make $_makecross -s kernelrelease > ../version
  msg2 "Prepared %s version %s" "$pkgbase" "$(<../version)"

  sed -i '2iexit 0' scripts/depmod.sh
}

build() {
  cd $_srcname
  unset LDFLAGS
  make $_makecross ${MAKEFLAGS} Image Image.gz modules
  # Generate device tree blobs with symbols to support applying device tree overlays in U-Boot
  make $_makecross ${MAKEFLAGS} DTC_FLAGS="-@" dtbs
}

package() {
  cd ${_srcname}

  KARCH=arm64

  # get kernel version
  _kernver="$(<../version)"
  _basekernel=${_kernver%%-*}
  _basekernel=${_basekernel%.*}

  mkdir -p "${pkgdir}"/{boot,usr/lib/modules}
  make $_makecross INSTALL_MOD_PATH="${pkgdir}/usr" modules_install
  make $_makecross INSTALL_DTBS_PATH="${pkgdir}/boot/dtbs" dtbs_install
  cp arch/$KARCH/boot/Image{,.gz} "${pkgdir}/boot"

  # make room for external modules
  local _extramodules="extramodules-${_basekernel}${_kernelname}"
  ln -s "../${_extramodules}" "${pkgdir}/usr/lib/modules/${_kernver}/extramodules"

  # add real version for building modules and running depmod from hook
  echo "${_kernver}" |
    install -Dm644 /dev/stdin "${pkgdir}/usr/lib/modules/${_extramodules}/version"

  # remove build and source links
  rm "${pkgdir}"/usr/lib/modules/${_kernver}/{source,build}

  # now we call depmod...
  depmod -b "${pkgdir}/usr" -F System.map "${_kernver}"

  # add vmlinux
  install -Dt "${pkgdir}/usr/lib/modules/${_kernver}/build" -m644 vmlinux

  # sed expression for following substitutions
  local _subst="
    s|%PKGBASE%|${pkgbase}|g
    s|%KERNVER%|${_kernver}|g
    s|%EXTRAMODULES%|${_extramodules}|g
  "

  # install mkinitcpio preset file
  sed "${_subst}" ../linux.preset |
    install -Dm644 /dev/stdin "${pkgdir}/etc/mkinitcpio.d/${pkgbase}.preset"

  # install pacman hooks
  sed "${_subst}" ../60-linux.hook |
    install -Dm644 /dev/stdin "${pkgdir}/usr/share/libalpm/hooks/60-${pkgbase}.hook"
  sed "${_subst}" ../90-linux.hook |
    install -Dm644 /dev/stdin "${pkgdir}/usr/share/libalpm/hooks/90-${pkgbase}.hook"
}

# vim:set ts=8 sts=2 sw=2 et:
