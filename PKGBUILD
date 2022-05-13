# Maintainer: Ariel Rivas <arielcrivas@gmail.com>

_CHECK=0
_NM_CLOUD_SETUP=0

pkgbase='networkmanager-iwd'
pkgname=('networkmanager-iwd' 'libnm-iwd')
(( $_NM_CLOUD_SETUP )) && pkgname+=('nm-iwd-cloud-setup')
pkgver=1.38.0
pkgrel=1
pkgdesc='NM modified package to use exclusively iwd backend getting rid of wpa_supplicant dependency'
epoch=2
url='https://wiki.gnome.org/Projects/NetworkManager'
arch=('x86_64')
license=('GPL2' 'LGPL2.1')
makedepends=('audit' 'bluez-libs' 'curl' 'dhclient' 'dnsmasq' 'git' 'glib2-docs'
             'gobject-introspection' 'gtk-doc' 'intltool' 'iproute2' 'iptables'
             'iwd' 'jansson' 'libmm-glib' 'libndp' 'libnewt' 'libpsl' 'libteam'
             'meson' 'modemmanager' 'nss' 'openresolv' 'perl-yaml'
             'python-gobject' 'systemd' 'vala' 'wpa_supplicant')

(( $_CHECK )) && checkdepends=('libx11' 'python-dbus')

_commit=5704730a6c4e6851d4ba5471ea439502f7b72949   # tags/1.38.0^{}
source=("git+https://gitlab.freedesktop.org/NetworkManager/NetworkManager.git#commit=$_commit")
sha256sums=('SKIP')

pkgver() {
  cd NetworkManager
  git describe --abbrev=10 | sed 's/-dev/dev/;s/-rc/rc/;s/-/+/g'
}

prepare() {
  cd NetworkManager
}

build() {
  local meson_args=(
    # system paths
    -D dbus_conf_dir=/usr/share/dbus-1/system.d

    # platform
    -D dist_version="$pkgver"
    -D session_tracking_consolekit=false
    -D suspend_resume=systemd
    -D modify_system=true
    -D selinux=false

    # features
    -D iwd=true
    -D ppp=false
    -D teamdctl=true
    -D bluez5_dun=true
    -D ebpf=true

    # configuration plugins
    -D config_plugins_default=keyfile

    # handlers for resolv.conf
    -D netconfig=no
    -D config_dns_rc_manager_default=symlink

    # dhcp clients
    -D dhcpcd=no

    # miscellaneous
    -D vapi=true
    -D docs=true
    -D more_asserts=no
    -D more_logging=false
    -D qt=false
  )

  if (( $_CHECK )); then
    meson_args+=( -D tests=yes )
  else
    meson_args+=( -D tests=no )
  fi

  if (( $_NM_CLOUD_SETUP )); then
    meson_args+=( -D nm_cloud_setup=true )
  else
    meson_args+=( -D nm_cloud_setup=false )
  fi

  arch-meson NetworkManager build "${meson_args[@]}"
  meson compile -C build
}


check() {
  if (( $_CHECK )); then
    # iproute2 bug
    # https://gitlab.freedesktop.org/NetworkManager/NetworkManager/commit/be76d8b624fab99cbd76092ff511e6adc305279c
    meson test -C build --print-errorlogs || :
  fi
}

_pick() {
  local p="$1" f d; shift
  for f; do
    d="$srcdir/$p/${f#$pkgdir/}"
    mkdir -p "$(dirname "$d")"
    mv "$f" "$d"
    rmdir -p --ignore-fail-on-non-empty "$(dirname "$f")"
  done
}

package_networkmanager-iwd() {
  depends=('audit' 'bluez-libs' 'curl' 'iproute2' 'iwd' 'libmm-glib' 'libndp'
           'libnewt' 'libnm-iwd' 'libpsl' 'libteam'
           'mobile-broadband-provider-info')
  provides=('networkmanager')
  conflicts=('networkmanager')
  optdepends=('dnsmasq: connection sharing'
              'bluez: Bluetooth support'
              'polkit: let non-root users control networking'
              'modemmanager: cellular network support'
              'dhclient: alternative DHCP client'
              'openresolv: alternative resolv.conf manager')
  backup=('etc/NetworkManager/NetworkManager.conf')
  groups=('gnome')
  install="$pkgbase.install"

  DESTDIR="$pkgdir" meson install -C build

  # /etc/NetworkManager
  install -d "$pkgdir"/etc/NetworkManager/{conf,dnsmasq}.d
  install -dm700 "$pkgdir/etc/NetworkManager/system-connections"
  install -m644 /dev/stdin "$pkgdir/etc/NetworkManager/NetworkManager.conf" <<END
# Configuration file for NetworkManager.
# See "man 5 NetworkManager.conf" for details.
END

  # packaged configuration
  install -Dm644 /dev/stdin "$pkgdir/usr/lib/NetworkManager/conf.d/20-connectivity.conf" <<END
[connectivity]
uri=http://ping.archlinux.org/nm-check.txt
END

  # iwd wifi backend
  install -Dm644 /dev/stdin "$pkgdir/usr/lib/NetworkManager/conf.d/30-wifi-backend.conf" <<END
[device]
wifi.backend=iwd
END

  # iwd.service overriding configuration
  install -Dm644 /dev/stdin "$pkgdir/etc/systemd/system/iwd.service.d/90-networkmanager.conf" <<END
[Unit]
After=systemd-udevd.service
Before=NetworkManager.service
END

  shopt -s globstar

  ### Split libnm
  _pick libnm "$pkgdir"/usr/include/libnm
  _pick libnm "$pkgdir"/usr/lib/girepository-1.0/NM-*
  _pick libnm "$pkgdir"/usr/lib/libnm.*
  _pick libnm "$pkgdir"/usr/lib/pkgconfig/libnm.pc
  _pick libnm "$pkgdir"/usr/share/gir-1.0/NM-*
  _pick libnm "$pkgdir"/usr/share/gtk-doc/html/libnm
  _pick libnm "$pkgdir"/usr/share/vala/vapi/libnm.*

  if (( $_NM_CLOUD_SETUP )); then
    ### Split nm-cloud-setup
    _pick nm-cloud-setup "$pkgdir"/usr/lib/**/*nm-cloud-setup*
    _pick nm-cloud-setup "$pkgdir"/usr/share/man/*/nm-cloud-setup*
  fi

  # Restore empty dir
  install -d -m 755 "$pkgdir/usr/lib/NetworkManager/dispatcher.d/no-wait.d"
}

package_libnm-iwd() {
  pkgdesc="NetworkManager client library with iwd backend"
  depends=('glib2' 'jansson' 'libutil-linux' 'nss' 'systemd-libs')
  provides=("libnm")
  conflicts=("libnm")

  mv libnm/* "$pkgdir"
}

package_nm-iwd-cloud-setup() {
  pkgdesc="Automatically configure NetworkManager in cloud"
  depends=('networkmanager-iwd')
  provides=("nm-cloud-setup")
  conflicts=("nm-cloud-setup")

  mv nm-cloud-setup/* "$pkgdir"
}

# vim: ft=sh syn=sh ts=2 sts=2 sw=2 et:
