# Maintainer: aimileus <me at aimileus dot nl>
# Maintainer: Maciej Borzecki <maciek.borzecki@gmail.com>
# Contributor: Timothy Redaelli <timothy.redaelli@gmail.com>
# Contributor: Zygmunt Krynicki <me at zygoon dot pl>

pkgname=snapd
pkgdesc="Service and tools for management of snap packages."
depends=('squashfs-tools' 'libseccomp' 'libsystemd')
optdepends=('bash-completion: bash completion support')
pkgver=2.31
pkgrel=3
arch=('x86_64')
url="https://github.com/snapcore/snapd"
license=('GPL3')
makedepends=('git' 'go-pie' 'go-tools' 'libseccomp' 'libcap' 'systemd' 'xfsprogs')
conflicts=('snap-confine')
options=('!strip' 'emptydirs')
install=snapd.install
source=("$pkgname-$pkgver::https://github.com/snapcore/${pkgname}/archive/$pkgver.tar.gz"
        "0001-cmd-snap-seccomp-drop-link-flags-that-will-be-reject.patch")
sha256sums=('973e7e8098f5780d71a0633a0fa7c3371ef7fb7ae120d464b2e25af9588c1f89'
           'ba4591f70b032b5e6f63d251cf6463ef93f3b963b8f19aac098b4c7dbed0309d')

_gourl=github.com/snapcore/snapd

prepare() {
  cd "$pkgname-$pkgver"

  export GOPATH="$srcdir/go"
  mkdir -p "$GOPATH"

  # Have snapd checkout appear in a place suitable for subsequent GOPATH. This
  # way we don't have to go get it again and it is exactly what the tag/hash
  # above describes.
  mkdir -p "$(dirname "$GOPATH/src/${_gourl}")"
  ln --no-target-directory -fs "$srcdir/$pkgname-$pkgver" "$GOPATH/src/${_gourl}"

  patch -Np1 -i "${srcdir}/0001-cmd-snap-seccomp-drop-link-flags-that-will-be-reject.patch"
}

build() {
  cd "$pkgname-$pkgver"
  export GOPATH="$srcdir/go"

  export CGO_ENABLED="1"
  export CGO_CFLAGS="${CFLAGS}"
  export CGO_CPPFLAGS="${CPPFLAGS}"
  export CGO_CXXFLAGS="${CXXFLAGS}"
  export CGO_LDFLAGS="${LDFLAGS}"

  ./mkversion.sh $pkgver-$pkgrel

  # Use get-deps.sh provided by upstream to fetch go dependencies using the
  # godeps tool and dependencies.tsv (maintained upstream).
  cd "$GOPATH/src/${_gourl}"
  XDG_CONFIG_HOME="$srcdir" ./get-deps.sh

  # Build go binaries
  go build -o $GOPATH/bin/snap "${_gourl}/cmd/snap"
  go build -o $GOPATH/bin/snapctl "${_gourl}/cmd/snapctl"
  go build -o $GOPATH/bin/snapd "${_gourl}/cmd/snapd"
  go build -o $GOPATH/bin/snap-seccomp "${_gourl}/cmd/snap-seccomp"
  # build snap-exec and snap-update-ns completely static for base snaps
  go build -o $GOPATH/bin/snap-update-ns -ldflags '-extldflags "-static"' "${_gourl}/cmd/snap-update-ns"
  CGO_ENABLED=0 go build -o $GOPATH/bin/snap-exec "${_gourl}/cmd/snap-exec"

  # Generate data files such as real systemd units, dbus service, environment
  # setup helpers out of the available templates
  make -C data \
       BINDIR=/bin \
       LIBEXECDIR=/usr/lib \
       SYSTEMDSYSTEMUNITDIR=/usr/lib/systemd/system \
       SNAP_MOUNT_DIR=/var/lib/snapd/snap \
       SNAPD_ENVIRONMENT_FILE=/etc/default/snapd

  cd cmd
  autoreconf -i -f
  ./configure \
    --prefix=/usr \
    --libexecdir=/usr/lib/snapd \
    --with-snap-mount-dir=/var/lib/snapd/snap \
    --disable-apparmor \
    --enable-nvidia-biarch \
    --enable-merged-usr
  make $MAKEFLAGS
}


package() {
  cd "$pkgname-$pkgver"
  export GOPATH="$srcdir/go"

  # Install bash completion
  install -Dm644 data/completion/snap \
    "$pkgdir/usr/share/bash-completion/completion/snap"
  install -Dm644 data/completion/complete.sh \
    "$pkgdir/usr/lib/snapd/complete.sh"
  install -Dm644 data/completion/etelpmoc.sh \
    "$pkgdir/usr/lib/snapd/etelpmoc.sh"

  # Install systemd units, dbus services and a script for environment variables
  make -C data/ install \
     DBUSSERVICESDIR=/usr/share/dbus-1/services \
     BINDIR=/usr/bin \
     SYSTEMDSYSTEMUNITDIR=/usr/lib/systemd/system \
     SNAP_MOUNT_DIR=/var/lib/snapd/snap \
     DESTDIR="$pkgdir"

  # Install polkit policy
  install -Dm644 data/polkit/io.snapcraft.snapd.policy \
    "$pkgdir/usr/share/polkit-1/actions/io.snapcraft.snapd.policy"

  # Install executables
  install -Dm755 "$GOPATH/bin/snap" "$pkgdir/usr/bin/snap"
  install -Dm755 "$GOPATH/bin/snapctl" "$pkgdir/usr/bin/snapctl"
  install -Dm755 "$GOPATH/bin/snapd" "$pkgdir/usr/lib/snapd/snapd"
  install -Dm755 "$GOPATH/bin/snap-seccomp" "$pkgdir/usr/lib/snapd/snap-seccomp"
  install -Dm755 "$GOPATH/bin/snap-update-ns" "$pkgdir/usr/lib/snapd/snap-update-ns"
  install -Dm755 "$GOPATH/bin/snap-exec" "$pkgdir/usr/lib/snapd/snap-exec"

  # Symlink /var/lib/snapd/snap to /snap so that --classic snaps work
  ln -s var/lib/snapd/snap "$pkgdir/snap"

  # pre-create directories
  install -dm755 "$pkgdir/var/lib/snapd/snap"
  install -dm755 "$pkgdir/var/cache/snapd"
  install -dm755 "$pkgdir/var/lib/snapd/assertions"
  install -dm755 "$pkgdir/var/lib/snapd/desktop/applications"
  install -dm755 "$pkgdir/var/lib/snapd/device"
  install -dm755 "$pkgdir/var/lib/snapd/hostfs"
  install -dm755 "$pkgdir/var/lib/snapd/mount"
  install -dm755 "$pkgdir/var/lib/snapd/seccomp/bpf"
  install -dm755 "$pkgdir/var/lib/snapd/snap/bin"
  install -dm755 "$pkgdir/var/lib/snapd/snaps"
  install -dm755 "$pkgdir/var/lib/snapd/lib/gl"
  install -dm755 "$pkgdir/var/lib/snapd/lib/gl32"
  install -dm755 "$pkgdir/var/lib/snapd/lib/vulkan"
  # these dirs have special permissions
  install -dm000 "$pkgdir/var/lib/snapd/void"
  install -dm700 "$pkgdir/var/lib/snapd/cookie"
  install -dm700 "$pkgdir/var/lib/snapd/cache"

  make -C cmd install DESTDIR="$pkgdir/"

  # Install man file
  "$GOPATH/bin/snap" help --man > "$pkgdir/usr/share/man/man1/snap.1"

  # Install the "info" data file with snapd version
  install -m 644 -D "$GOPATH/src/${_gourl}/data/info" \
          "$pkgdir/usr/lib/snapd/info"

  # Remove snappy core specific units
  rm -fv "$pkgdir/usr/lib/systemd/system/snapd.system-shutdown.service"
  rm -fv "$pkgdir/usr/lib/systemd/system/snapd.autoimport.service"
  rm -fv "$pkgdir"/usr/lib/systemd/system/snapd.snap-repair.*
  rm -fv "$pkgdir"/usr/lib/systemd/system/snapd.core-fixup.*
  # and scripts
  rm -fv "$pkgdir/usr/lib/snapd/snapd.core-fixup.sh"
  rm -fv "$pkgdir/usr/bin/ubuntu-core-launcher"
  rm -fv "$pkgdir/usr/lib/snapd/system-shutdown"
}
