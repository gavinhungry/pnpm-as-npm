# Maintainer: Gavin Lloyd <gavinhungry@gmail.com>

pkgname=pnpm-as-npm
pkgver=0.1.0
pkgrel=1
pkgdesc="Compatibility wrapper to package npm projects with pnpm"
url="https://github.com/gavinhungry/${pkgname}"
arch=(any)
license=('MIT')
depends=(bash coreutils)
provides=(npm)
source=(npm npx ${pkgname}.conf)
sha256sums=('c72fee49e816157da8d8c9a5afd4ea31454a80291498c1d010b0a46522a23665'
            '4973d6a01f4d500391ef01d5a8b0393e0e827bda3d742ff41c251b767a9fb675'
            '28651ecc66f9ff6b9b892fe50ee6996914b4e1e94258d04780e7d0e7dc6ab41f')

package() {
  cd "${srcdir}"

  install -Dm755 -t "${pkgdir}/usr/lib/${pkgname}/bin" npm npx
  install -Dm644 -t "${pkgdir}/etc/makepkg.conf.d" "${srcdir}"/${pkgname}.conf
}
