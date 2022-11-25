# Windows Symlinks

Let's figure out what to do about symlinks on Windows.  Sometimes Windows supports them, sometimes not.
What should we do when we are building files on Windows meant for another platform that expects symlinks?

First, let's create a library that can detect when symlinks are supported on windows.
Second, if symlinks aren't supported, can the user do something to enable them and can our software ask the user to do so?
If both of these fail, can we emulate symlinks?
Let's look at what a symlink is:

https://man7.org/linux/man-pages/man7/symlink.7.html

> A symbolic link is a special type of file whose contents are a
  string that is the pathname of another file, the file to which
  the link refers.

Symlinks on linux are identical to normal files except their "file type" is `120000` instead of `100000` (file type is a set of bits in the file's "mode").

Given this, we could make our "virtual windows symlinks" normal files and then find some way of marking these files as "symlinks".  For marking files as symlinks, here are some strategies that come to mind:

1. put something "special" in the filename (i.e. `foo.virtual-windows-symlink`)
2. put something "special" in the file content (i.e. `VirtualWindowsSymlink: I'm a virtual windows symlink, see http://github.com/marler8997/WindowsSymlinks\ntarget`)
    - include some magic characters at the start of the file
    - could include some "explanation" after the magic characters
    - delimit the "explanation" with something (probably a newline) followed by the symlink target
3. use an NTFS "Alternate Data Stream" to mark it as a symlink

Let's look at an example to compare these options.  Say we are on windows building the "foo" library for linux.  It creates the file `libfoo.so.1.0` and a symlink `libfoo.so -> libfoo.so.1.0`.  Now let's say we copy the resulting files to linux with a program that doesn't understand our symlink mechanism.  Here's what would happen with each option.

* Option 1: special filename `libfoo.so.virtual-windows-symlink`.  If you try to link against `foo` you'll get a "library not found" error, but, if you look at where the library should be you could likely figure out what's going on.
* Option 2: special file content `ImAVirtualWindowsSymlink:libfoo.so.1.0`.  If you try to link against `foo` you'll probably get some kind "corrupt library format" error.  Looking at the content of the file will tell the developer exactly what's going on.
* Option 3: NTFS Alternate Data Stream.  In this case the information about this file being a symlink is lost.  A user trying to link against `libfoo.so` will likely get a "corrupt library format" error.  When you look at the file it's possible you could figure out it's suppose to be a symlink to `libfoo.so.1.0`, but, it won't be as clear as it would be with Option 2.

# Make other programs understand our symlinks?

It's easy enough to implement a mechanism for this in our own library and use it, but how do we get other programs to use it?  Programs that are already built?  Is there a way we could override functions in ntdll with wrappers that emulate symlinks using our mechanism?
