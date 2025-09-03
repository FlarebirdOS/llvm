pkgname=('llvm' 'llvm-libs')
pkgver=21.1.0
pkgrel=1
arch=('x86_64')
url="https://llvm.org/"
license=('Apache-2.0 WITH LLVM-exception')
makedepends=(
    'cmake'
    'curl'
    'libffi'
    'libxml2'
    'ninja'
    'python-myst-parser'
    'python-psutil'
    'python-setuptools'
    'python-sphinx'
    'zlib'
    'zstd'
)
options=('!staticlibs' '!lto') # tools/llvm-shlib/typeids.test fails with LTO
source=(https://github.com/llvm/llvm-project/releases/download/llvmorg-${pkgver}/llvm-${pkgver}.src.tar.xz
    https://github.com/llvm/llvm-project/releases/download/llvmorg-${pkgver}/cmake-${pkgver}.src.tar.xz
    https://github.com/llvm/llvm-project/releases/download/llvmorg-${pkgver}/third-party-${pkgver}.src.tar.xz)
sha256sums=(0582ee18cb6e93f4e370cb4aa1e79465ba1100408053e1ff8294cef7fb230bd8
    528347c84c3571d9d387b825ef8b07c7ad93e9437243c32173838439c3b6028f
    60b3d8c2d1d8d43a705f467299144d232b8061a10541eaa3b0d6eaa2049a462f)

# Utilizing LLVM_DISTRIBUTION_COMPONENTS to avoid
# installing static libraries; inspired by Gentoo
_get_distribution_components() {
    local target
    cmake --build flarebird-build -t targets | grep -Po 'install-\K.*(?=-stripped:)' | while read -r target; do
        case ${target} in
            llvm-libraries|distribution)
                continue
                ;;
            # shared libraries
            LLVM|LLVMgold)
                ;;
            # libraries needed for clang-tblgen
            LLVMDemangle|LLVMSupport|LLVMTableGen)
                ;;
            # testing libraries
            LLVMTestingAnnotations|LLVMTestingSupport)
                ;;
            # exclude static libraries
            LLVM*)
                continue
                ;;
            # exclude llvm-exegesis (doesn't seem useful without libpfm)
            llvm-exegesis)
                continue
                ;;
        esac
        echo ${target}
    done
}

prepare() {
   rename -v -- "-${pkgver}.src" '' {cmake,third-party}-${pkgver}.src

   cd llvm-${pkgver}.src

   # Remove CMake find module for zstd; breaks if out of sync with upstream zstd
   rm cmake/modules/Findzstd.cmake
}

build() {
    cd llvm-${pkgver}.src

    local cmake_args=(
        -B flarebird-build
        -G Ninja
        -D CMAKE_BUILD_TYPE=Release
        -D CMAKE_INSTALL_PREFIX=/usr
        -D LLVM_LIBDIR_SUFFIX=64
        -D CMAKE_INSTALL_DOCDIR=share/doc
        -D CMAKE_SKIP_RPATH=ON
        -D LLVM_BINUTILS_INCDIR=/usr/include
        -D LLVM_BUILD_DOCS=ON
        -D LLVM_BUILD_LLVM_DYLIB=ON
        -D LLVM_BUILD_TESTS=ON
        -D LLVM_ENABLE_BINDINGS=OFF
        -D LLVM_ENABLE_CURL=ON
        -D LLVM_ENABLE_FFI=ON
        -D LLVM_ENABLE_RTTI=ON
        -D LLVM_ENABLE_SPHINX=ON
        -D LLVM_HOST_TRIPLE=${CHOST}
        -D LLVM_INCLUDE_BENCHMARKS=OFF
        -D LLVM_INSTALL_GTEST=ON
        -D LLVM_INSTALL_UTILS=ON
        -D LLVM_LINK_LLVM_DYLIB=ON
        -D LLVM_USE_PERF=ON
        -D SPHINX_WARNINGS_AS_ERRORS=OFF
    )

    # Build only minimal debug info to reduce size
    CFLAGS=${CFLAGS/-g /-g1 }
    CXXFLAGS=${CXXFLAGS/-g /-g1 }

    cmake "${cmake_args[@]}"

    local distribution_components=$(_get_distribution_components | paste -sd\;)

    test -n ${distribution_components}
    cmake_args+=(-DLLVM_DISTRIBUTION_COMPONENTS=${distribution_components})

    cmake "${cmake_args[@]}"

    cmake --build flarebird-build
}

package_llvm() {
    pkgdesc="Compiler infrastructure"
    depends=('llvm-libs' 'curl' 'perl')

    cd llvm-${pkgver}.src

    DESTDIR=${pkgdir} cmake --install flarebird-build

    # Include lit for running lit-based tests in other projects
    pushd utils/lit
        python3 setup.py install --root=${pkgdir} -O1
    popd

    # The runtime libraries go into llvm-libs
    _pick llvm-libs ${pkgdir}/usr/lib64/lib{LLVM,LTO,Remarks}*.so*
    _pick llvm-libs ${pkgdir}/usr/lib64/LLVMgold.so

    # Remove documentation sources
    rm -r ${pkgdir}/usr/share/doc/llvm/html/{_sources,.buildinfo}
}

package_llvm-libs() {
    pkgdesc="LLVM runtime libraries"
    depends=('gcc-libs' 'zlib' 'zstd' 'libffi' 'libxml2')

    mv ${pkgname}/* ${pkgdir}

    # Symlink LLVMgold.so from /usr/lib/bfd-plugins
    # https://bugs.archlinux.org/task/28479
    install -d ${pkgdir}/usr/lib64/bfd-plugins
    ln -s ../LLVMgold.so ${pkgdir}/usr/lib64/bfd-plugins/LLVMgold.so
}
