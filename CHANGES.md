# splashutils / fbcondecor today: what has to change since 2010

splashutils-1.5.4.4 (2008, "lite" tarball 2011) and the fbcondecor kernel
patch (last upstream version for Linux 5.9, 2019) predate most of today's
toolchain and kernel. This is the list of things that no longer work as
shipped and what to do about them, independent of any particular
distribution. Tested with gcc 16, glibc 2.4x, autoconf 2.73/automake 1.18,
zlib 1.3.2, libpng 1.6.58, libjpeg-turbo 3.2.0, freetype 2.14.3, klibc
2.0.14 and Linux 7.0-7.3 (x86_64).

## 1. Kernel: the old patch cannot be refreshed, it has to be reworked

* Since Linux 5.17 the DRM drivers (amdgpu, i915, simpledrm, virtio, bochs,
  ...) implement fbdev through generic helpers with a *shadow buffer* and
  deferred I/O. Only writes made through `fb_fillrect/copyarea/imageblit`
  are reported as damage and reach the display. fbcondecor writes pixels
  straight into the screen memory, so a textually refreshed 5.9 patch
  compiles but shows nothing on such drivers (and i915's panel self refresh
  never wakes up).
  Fix in the 7.x port: an optional `fb_ops.fb_damage()` callback generated
  by the deferred I/O helper macros, called by the renderer after every
  drawing operation. No driver changes needed.
* fbcon itself was rewritten in large parts between 5.9 and 7.0
  (`fbcon_ops` -> `fbcon_par`/`fbcon_bitops`, `fbcon_info_from_console()`
  instead of `registered_fb[con2fb_map[]]`, `con_clear()` clears one line,
  `con_cursor(vc, bool)`, sysctl tables without `child`, compat ioctl
  structures moved, fbcon built into `fb.o` in `drivers/video/fbdev/core`).
  Every hunk of the old patch had to be re-placed by hand.
* The kernel-userspace ABI is unchanged (ioctls `'F' 0x19-0x1D`,
  `struct fbcon_decor_iowrapper`, `struct vc_decor`, `/dev/fbcondecor`,
  helper protocol v2 `<helper> 2 <cmd> <vc> <fb> <theme>`, sysctl
  `kernel.fbcondecor`), so splashutils-1.5.4.4 talks to the new kernel
  without modification.
* Behaviour changes worth knowing: the helper is now called with `init` on
  *every* fbcon takeover (a DRM driver replacing the firmware framebuffer
  before userspace starts would otherwise leave the console undecorated);
  the picture is only reloaded on a console switch when it is missing or
  belongs to another theme/resolution; DIRECTCOLOR palette tricks are gone
  (DRM fbdev never uses that visual); rotated consoles are not decorated.

## 2. splashutils sources: four small fixes for current compilers

gcc 14 turned implicit function declarations and int/pointer conversions
into errors; everything else in the sources still compiles. Needed in every
build variant (`splashutils-1.5.4.4-system-libs.patch`):

* `src/kernel.c`: `#include <sys/mount.h>` and `<sys/stat.h>` (mount,
  umount, mkdir were only implicitly declared; klibc's headers used to
  provide them)
* `src/common.c`: `#include <sys/sysmacros.h>` (makedev)
* `src/render.h`: prototypes for `obj_visibility_set()` and `list_ringadd()`
* `src/util.c`: rename the global `arg_task` (clashes with the one in
  `fbcon_decor_ctl.c` in the unified static `fbsplashctl` build)

The libpng/libjpeg/freetype API used by the code is the current one, no
library porting is needed. `autoreconf -fi` with current autotools works
(warnings about `INCLUDES` and `subdir-objects` only). Optional:
`splashutils-1.5.4.4-exec-child-exit.patch` adds `_exit()` after a failed
`execlp("sh")` in `text ... exec` objects, otherwise the child runs on as a
second helper when `sh` is missing in the initramfs.

## 3. The bundled libraries and klibc are not required any more

The original build compiled bundled 2010 sources of freetype, libjpeg,
libpng and zlib with klcc to get a static `fbcondecor_helper`. Three ways
to build the helper today, all verified to work with the 7.x kernel:

### a) dynamic, system libraries (simplest)

    patch -Np1 -i splashutils-1.5.4.4-system-libs.patch
    autoreconf -fi
    ./configure --without-klibc --disable-static-binaries \
        [--without-gpm --without-ttf --without-ttf-kernel] [--with-themedir=DIR]
    make && make install

`--without-klibc` makes the helper an ordinary glibc program;
`--disable-static-binaries` because `fbsplashctl` needs static libraries
that modern systems do not install. TTF (freetype), gpm and mng (libmng
2.0.3) all compile. The initramfs then needs the helper plus
libfbsplashrender, libfbsplash, libpng16, libjpeg, libz, libm, libc, the
loader (and libfreetype + libbz2 + libbrotli* with TTF).

### b) static glibc helper

Build static, PIC copies of zlib, libpng and libjpeg-turbo (and freetype)
from their current tarballs into a private prefix `$P`:
zlib `CFLAGS=-fPIC ./configure --static`, libpng `--disable-shared
--enable-static --with-pic`, libjpeg-turbo `cmake -DENABLE_SHARED=OFF
-DENABLE_STATIC=ON -DCMAKE_POSITION_INDEPENDENT_CODE=ON`, freetype
`--disable-shared --enable-static --with-pic --with-bzip2=no --with-png=no
--with-harfbuzz=no --with-brotli=no`. Then `rm -f $P/lib/*.la` (with the
libtool files present, libtool inlines the archives and leaves `-lpng16`
out of `libfbsplashrender.la`, and the static programs fail to link) and:

    ./configure --without-klibc ... \
        PNG_CFLAGS=-I$P/include PNG_LIBS="-L$P/lib -lpng16 -lz -lm" \
        JPEG_CFLAGS=-I$P/include JPEG_LIBS="-L$P/lib -ljpeg" \
        [FREETYPE2_CFLAGS=-I$P/include/freetype2 FREETYPE2_LIBS="-L$P/lib -lfreetype -lz"]
    make fbcondecor_helper_LDFLAGS=-all-static
    make fbcondecor_helper_LDFLAGS=-all-static install

`-lm` is needed by libpng (floor/pow). `--enable-static-binaries` (default)
works here too, giving the static `fbsplashctl` tools.

### c) klibc helper (smallest, the original design)

* klibc 2.0.14 builds with gcc 16 unchanged. It needs the kernel UAPI
  headers: `make KLIBCKERNELSRC=<dir>` where `<dir>/include` holds them;
  the sanitized headers already installed in `/usr/include` do
  (`KLIBCKERNELSRC=/usr`), no kernel source tree required. `make install`
  also drops the shared `klibc-*.so` into `SHLIBDIR` (`/lib` by default),
  which the static helper does not need.
* The current libraries build with `CC=klcc` with these tweaks:
  zlib: `./configure --static`, then `make libz.a` only (the examples do
  not build); libpng: klibc has no libm, so disable floating point with a
  `pngusr.dfa` (`option FLOATING_ARITHMETIC off`, `FLOATING_POINT off`,
  `WRITE off`, `CONSOLE_IO off`, `TIME_RFC1123 off`, `WARNINGS off`,
  passed as `DFA_XTRA=pngusr.dfa`), skip the pow() check with
  `ac_cv_func_pow=yes`, add `--host=x86_64-pc-linux-gnu --disable-tools
  --disable-tests --disable-hardware-optimizations`; libjpeg-turbo:
  cmake cannot detect the ABI with klcc, pass `-DCMAKE_SIZEOF_VOID_P=8`,
  use `-DWITH_SIMD=OFF`, build target `jpeg-static`, copy `libjpeg.a` and
  `jconfig.h`, `src/jpeglib.h`, `src/jmorecfg.h`, `src/jerror.h` by hand;
  freetype: same configure as in (b) plus `CC=klcc --host=...`, no changes.
* splashutils: `--with-klibc --with-klibc-compiler=klcc` and
  `--with-{jpeg,lpng,zlib}-src=<dir with the headers>`
  (`--with-freetype2-src=<dir>` where `<dir>/include` holds `ft2build.h`),
  plus `--disable-static-binaries`. Copy the prebuilt archives to
  `libs/libz.a`, `libs/libpng.a`, `libs/libjpeg.a` (`libs/libfreetype.a`)
  *in this order* before `make`: the Makefile has `libpng.a: libz.a` and
  would try to build libpng from the "source" directory if libz.a were
  newer. Apply `splashutils-1.5.4.4-klibc-gcc14.patch`
  (`-Wno-error=int-conversion` for the helper objects: klibc declares
  `ioctl(int, int, void *)` and the sources pass integers).

Helper sizes (stripped): dynamic 81 kB + ~1.6 MB of shared libraries not
otherwise present in my initramfs; static glibc 1.7 MB (2.4 MB with TTF);
klibc 400 kB (860 kB with TTF).

## 4. Runtime: what the helper expects today

* `/sbin/fbcondecor_helper` (path in `/proc/sys/kernel/fbcondecor`) must be
  in the initramfs together with the theme
  (`<themedir>/<name>/<xres>x<yres>.cfg` + images; the helper picks the
  configuration for the framebuffer resolution or the nearest smaller one)
  and the directory `<libdir>/splash` (it mounts proc and sysfs below it;
  `--with-essential-libdir` sets `<libdir>`). Kernel command line:
  `splash=verbose,theme:<name>`.
* The kernel runs the helper with no stdin/stdout/stderr: a missing shared
  library or a wrong loader path fails silently. Test a dynamic helper
  inside the image tree first (`bwrap --bind <dir> / --dev /dev --proc /proc
  /sbin/fbcondecor_helper`, or chroot). Static helpers avoid the problem.
* The helper creates `/dev/fb0` and `/dev/fbcondecor` itself from sysfs,
  devtmpfs is not required; `/dev/console` exists in every initramfs.
* With TTF compiled in, the theme should set `text_size=<n>` and the theme
  directory must contain the font (`luxisri.ttf` by default), or the helper
  complains about a "0 pt font". `text ... exec` objects need `sh` in the
  image. For a console background TTF only adds static captions baked into
  the picture; the boot message, progress text and message log exist in
  the silent mode of `fbsplashd`, whose init integration is Gentoo-rc
  specific. gpm support only lets `fbsplashd` keep the mouse cursor off the
  silent tty.

