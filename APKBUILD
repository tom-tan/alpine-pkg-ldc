# Contributor: Mathias LANG <pro.mathias.lang@gmail.com>
# Maintainer: Mathias LANG <pro.mathias.lang@gmail.com>
pkgname=ldc
pkgver=1.37.0
pkgrel=0
_llvmver=15
pkgdesc="The LLVM-based D Compiler"
url="https://github.com/ldc-developers/ldc"
# LDC does not support host compiling on most of the architecture Alpine supports
arch="x86_64 aarch64"
license="BSD-3-Clause AND BSL-1.0 AND ( Artistic-1.0 OR GPL-2.0-or-later ) AND NCSA AND MIT"
depends="
	$pkgname-static=$pkgver-r$pkgrel
	llvm-libunwind-dev
	tzdata
	"
makedepends="
	chrpath
	clang
	cmake
	curl-dev
	diffutils
	gdmd
	libedit-dev
	llvm$_llvmver-dev
	llvm$_llvmver-static
	samurai
	zlib-dev
	"
checkdepends="bash gdb grep llvm$_llvmver-test-utils"
# A user might want to install the '-runtime' subpackage when they have
# a dynamically-linked D program.
subpackages="
	$pkgname-dbg
	$pkgname-runtime
	$pkgname-static
	$pkgname-bash-completion
	"
source="https://github.com/ldc-developers/ldc/releases/download/v$pkgver/ldc-$pkgver-src.tar.gz
	lfs64.patch
	"
builddir="$srcdir/ldc-$pkgver-src/"

build() {
	# use less memory to not oom
	export CC=clang
	export CXX=clang++

	case "$CARCH" in
	aarch64)
		export CFLAGS="${CFLAGS/-fstack-clash-protection}"
		export CXXFLAGS="${CXXFLAGS/-fstack-clash-protection}"
		;;
	esac

	# First, build LDC using GDC
	if [ "$CBUILD" != "$CHOST" ]; then
		CMAKE_CROSSOPTS="-DCMAKE_SYSTEM_NAME=Linux -DCMAKE_HOST_SYSTEM_NAME=Linux"
	fi
	unset DFLAGS
	cmake -G Ninja -B stage1 \
		-DD_COMPILER='gdmd' \
		-DADDITIONAL_DEFAULT_LDC_SWITCHES=' "-linker=bfd", "-link-defaultlib-shared", "-L--export-dynamic", "-L--eh-frame-hdr"' \
		-DLLVM_ROOT_DIR="/usr/lib/llvm$_llvmver" \
		$CMAKE_CROSSOPTS
	ninja -C stage1

	cmake -G Ninja -S . \
		-DCMAKE_INSTALL_PREFIX=/usr \
		-DCMAKE_INSTALL_LIBDIR=lib \
		-DBUILD_SHARED_LIBS=True \
		-DCMAKE_BUILD_TYPE=RelWithDebInfo \
		-DD_COMPILER="$builddir/stage1/bin/ldmd2" \
		-DC_SYSTEM_LIBS="unwind;m;pthread;rt;dl" \
		-DADDITIONAL_DEFAULT_LDC_SWITCHES=' "-linker=bfd", "-link-defaultlib-shared", "-L--export-dynamic", "-L--eh-frame-hdr"' \
		-DLLVM_ROOT_DIR="/usr/lib/llvm$_llvmver" \
		$CMAKE_CROSSOPTS
	ninja

	# Build the test runners
	if want_check; then
		ninja all-test-runners
	fi
}

check() {
	# Find the libraries we just built as final (not stage1!)
	export LD_LIBRARY_PATH="$builddir"/lib

	case "$CARCH" in
	# Math & numeric related tests fail due to https://github.com/ldc-developers/ldc/issues/3270#issuecomment-613132406
	# druntime-test-thread fails due to https://github.com/ldc-developers/ldc/issues/3403
	aarch64)
		_tests_ignore="|core\.thread\.fiber|std.*math.*|std\.numeric.*|std\.format|std\.algorithm\.sorting.*|druntime-test-thread|std\.complex.*"
		;;
	# https://github.com/ldc-developers/ldc/issues/3404
	x86)
		_tests_ignore="|druntime-test-thread"
		;;
	esac

	# Note: The testsuite does not parallelize well, as the 'clean' target get run in parallel.
	# Hence '-j${JOBS}' was left out on purpose
	#
	# - dmd-testsuite takes too long to run and has more to do with language checks
	#	which are less relevant to us than platform integration tests
	# - lit-test disabled because 'TEST 'LDC :: debuginfo/print_gdb.d' FAILED'
	#	See https://gitlab.alpinelinux.org/alpine/aports/-/issues/11154
	# - 'druntime-test-shared' fails, probably because it is using 'Object.factory'
	# - 'druntime-test-stdcpp' fails for an unknown reason and is temporarily disabled
	# - 'druntime-test-exceptions' fails for an unknown reason and is temporarily disabled
	#
	# The following test fails since v1.27.0, but seems to fail for unrelated reason.
	# - 'druntime-test-cycles'
	# Namely, the following assert is triggered:
	# core.exception.AssertError@../../src/rt/lifetime.d(1250): Assertion failure
	# ----------------
	# ??:? _d_assert [0x7fc894efb1a0]
	# ??:? void rt.lifetime.__unittest_L1236_C12() [0x5642121a5040]
	#
	# Link: https://github.com/ldc-developers/druntime/blob/8e135b4e978975b24536e2a938801a29b39dc9f6/src/rt/lifetime.d#L1250
	# However this unittest is AFAICS unrelated to the two tests,
	# and either succeed or isn't run on its own.
	ctest --output-on-failure -E "std.file*|std.datetime.timezone*|dmd-testsuite|lit-tests|druntime-test-exceptions|druntime-test-shared|druntime-test-stdcpp|druntime-test-cycles$_tests_ignore"
}

package() {
	DESTDIR="$pkgdir" cmake --install .

	# CMake added the rpaths to the shared libs (of stage1!) - strip them
	chrpath -d "$pkgdir"/usr/lib/*.so* "$pkgdir"/usr/bin/*

	mkdir -p "$pkgdir"/usr/share/bash-completion
	mv "$pkgdir"/etc/bash_completion.d "$pkgdir"/usr/share/bash-completion/completions
}

runtime() {
	pkgdesc="Dynamic runtime library for D code compiled with $pkgname-$pkgver"
	depends=

	for libn in libdruntime libphobos2; do
		amove usr/lib/$libn-ldc-shared.so*
	done
	# As of LDC v1.28.0, JIT is not supported for LLVM >= 12
	# https://github.com/ldc-developers/ldc/blob/v1.28.0/CMakeLists.txt#L452
	#mv "$pkgdir"/usr/lib/libldc-jit.so* "$subpkgdir/usr/lib"

	amove usr/lib/*.so*
}

sha512sums="
488451dba58262cf533760f471f707f984d66edeb5c7dfff5a512efa0111742cead4ff23ed5ace39ea4d07e9bac290a846d0df3de49fd3fc276241a771aff0ed  ldc-1.37.0-src.tar.gz
068ab2b4f2f4c43ca3b68a7b6adabefd7ee32e9ace272a5e0146d5afd4b37d90dd5d44f838bdb08313445de40b8701fbf19bf265e26c6c4f3a33a8cf3dbbeab1  lfs64.patch
"
