# Framebuffer console decoration kernel patch

The framebuffer decorations are a kernel feature which allows displaying
a background picture on selected consoles.

I'm not author of this patch. I actually don't understand this patch. This code has origin in
[last Gentoo version](https://gitweb.gentoo.org/proj/linux-patches.git/tree/4200_fbcondecor.patch?h=4.19)
for 4.19 kernel. I only rebase it to new kernel versions since that time for my personal use.
Therefore patch is also tested only on amdgpu and simpledrm framebuffers (but i915 should also work).

Note: After version 5.17, when `amdgpu` underwent a major refactoring, I stopped maintaining the patch;
however, thanks to Claude Code, it is now back. Change notes can be found in [CHANGES.md](CHANGES.md).

*Use at your own risk.*

