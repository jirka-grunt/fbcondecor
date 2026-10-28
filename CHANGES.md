# fbcondecor today: what has to change since 2010

splashutils-1.5.4.4 (2008, "lite" tarball 2011) and the fbcondecor kernel
patch (last upstream version for Linux 5.9, 2019) predate most of today's
toolchain and kernel. This is the list of things that no longer work as
shipped and what to do about them, independent of any particular
distribution.

## Kernel: the old patch cannot be refreshed, it has to be reworked

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

