pkgname=fix-thinkbook-media-keys
pkgver=1.0
pkgrel=1
pkgdesc="Fixes missing release events in media keys on Lenovo ThinkBook G7"
arch=('any')
url="https://codeberg.org/nathawat/thinkbook-g7-media-key-fix"
license=('CC0-1.0')
depends=('systemd')

package() {
    # Create the target directory
    install -d "${pkgdir}/usr/lib/udev/hwdb.d"

    # Write the hardware database rule directly into the package
    cat << 'EOF' > "${pkgdir}/usr/lib/udev/hwdb.d/90-media-keys.hwdb"
# Fix missing release events for Lenovo ThinkBook G7
evdev:atkbd:dmi:*
 KEYBOARD_KEY_ae=!volumedown
 KEYBOARD_KEY_b0=!volumeup
EOF
}
