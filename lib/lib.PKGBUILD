
# Use this BOILERPLATE to import this in your PKGBUILDs.
# For makepkg '. ../lib/lib.PKGBUILD' would suffice, but
# makechrootpkg wants everything strictly isolated and
# builds in a clean env. But there's an escape hatch:
# it loads PKGBUILD once *outside* the chroot, giving
# a chance to glue everything together.
#
# ```
# # load common settings; be aware if you ever split up this repo.
# [ -f ../lib/lib.PKGBUILD ] && cp -lf ../lib/lib.PKGBUILD .lib.PKGBUILD
# . .lib.PKGBUILD || exit 1
# ```

arch=('any')
url="https://github.com/kousu/arch-conf"
license=("MIT")

_pkgver() {

  # construct a version number from how many commits are in the _folder_ (./);
  # cache it to .pkgver, so that containerized builds which don't have
  # access to git; this works because the containerized builds (pikaur, makechrootpkg) all load PKGBUILD once outside the container first to orient themselves.
  if (command -v git >/dev/null) && \
     REVS=$(git rev-list --count HEAD ./ 2>/dev/null ) && \
     COMMIT=$(git describe --always --dirty 2>/dev/null ) ; then
       COMMIT=$(echo "$COMMIT" | sed s/-/+/g)
       echo "${REVS}$(if git status --porcelain ./ | grep -q .; then echo "+dirty"; fi)" > .pkgver
  fi

  # error handling:
  # if this somehow runs in a container without git and without
  # first creating this file, cat will error to stderr and then
  # makepkg will stop because it refuses empty version strings.
  # That's why there's no explicit error handling here.
  cat .pkgver
}
pkgver="$(_pkgver)"
pkgrel=1

groups=(kousu)

if [ -f "${pkgname}.install" ]; then
  install="${pkgname}.install"
fi

# PKGBUILD's `source` has some basic support for handling individual local files
# but it expects them to all be in the top-level folder. It's not for bundling
# an entire directory tree; for that, it expects to be given a .tar to extract
# -into- src/. That's not maintainable so instead, in most cases,I've committed
# my package contents *directly* to src/ and subvert PKGBUILD slight with this.
_package() {
  # build the package directly from the contents of src/, if any
  if [ -d "${startdir}/src" ]; then
    # note the '/.'. it means "copy the contents of this directory"
    # whereas 'src/' would create nested '${pkgdir}/src/'.
    cp -rp "${startdir}/src/." "${pkgdir}"

    # git doesn't record permissions beyond the executable bit, so what's
    # in src/ depends on the umask at *clone* time. Normalize as if it was
    # umask 022, so that the same perms get embedded by everyone.
    chmod -R u=rwX,go=rX "${pkgdir}"
  fi
}

# Override this per-package if needed.
package() {
  _package
}
