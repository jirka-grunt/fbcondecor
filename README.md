# Framebuffer console decoration kernel patch

The framebuffer decorations are a kernel feature which allows displaying
a background picture on selected consoles.

I'm not author of this patch. I actually don't understand this patch. This code has origin in
[last Gentoo version](https://gitweb.gentoo.org/proj/linux-patches.git/tree/4200_fbcondecor.patch?h=4.19)
for 4.19 kernel. I only rebase it to new kernel versions since that time for my personal use.
Therefore patch is also tested only on i915 and amdgpu drm framebuffers.

Note: Versions from 5.17 are currently broken on amdgpu after [kernel commit](https://github.com/torvalds/linux/commit/087451f372bf76d971184caa258807b7c35aac8f)
converting amdgpudrmfb to generic fb helpers.

*Use at your own risk.*

