.. SPDX-License-Identifier: GPL-2.0

==========================
The Linux Microcode Loader
==========================

:Authors: - Fenghua Yu <fenghua.yu@intel.com>
          - Borislav Petkov <bp@suse.de>

The kernel has a x86 microcode loading facility which is supposed to
provide microcode loading methods in the OS. Potential use cases are
updating the microcode on platforms beyond the OEM End-Of-Life support,
and updating the microcode on long-running systems without rebooting.

The loader supports three loading methods:

Early load microcode
====================

The kernel can update microcode very early during boot. Loading
microcode early can fix CPU issues before they are observed during
kernel boot time.

The microcode is stored in an initrd file. During boot, it is read from
it and loaded into the CPU cores.

The format of the combined initrd image is microcode in (uncompressed)
cpio format followed by the (possibly compressed) initrd image. The
loader parses the combined initrd image during boot.

The microcode files in cpio name space are:

on Intel:
  kernel/x86/microcode/GenuineIntel.bin
on AMD  :
  kernel/x86/microcode/AuthenticAMD.bin

During BSP (BootStrapping Processor) boot (pre-SMP), the kernel
scans the microcode file in the initrd. If microcode matching the
CPU is found, it will be applied in the BSP and later on in all APs
(Application Processors).

The loader also saves the matching microcode for the CPU in memory.
Thus, the cached microcode patch is applied when CPUs resume from a
sleep state.

Here's a crude example how to prepare an initrd with microcode (this is
normally done automatically by the distribution, when recreating the
initrd, so you don't really have to do it yourself. It is documented
here for future reference only).
::

  #!/bin/bash

  if [ -z "$1" ]; then
      echo "You need to supply an initrd file"
      exit 1
  fi

  INITRD="$1"

  DSTDIR=kernel/x86/microcode
  TMPDIR=/tmp/initrd

  rm -rf $TMPDIR

  mkdir $TMPDIR
  cd $TMPDIR
  mkdir -p $DSTDIR

  if [ -d /lib/firmware/amd-ucode ]; then
          cat /lib/firmware/amd-ucode/microcode_amd*.bin > $DSTDIR/AuthenticAMD.bin
  fi

  if [ -d /lib/firmware/intel-ucode ]; then
          cat /lib/firmware/intel-ucode/* > $DSTDIR/GenuineIntel.bin
  fi

  find . | cpio -o -H newc >../ucode.cpio
  cd ..
  mv $INITRD $INITRD.orig
  cat ucode.cpio $INITRD.orig > $INITRD

  rm -rf $TMPDIR


The system needs to have the microcode packages installed into
/lib/firmware or you need to fixup the paths above if yours are
somewhere else and/or you've downloaded them directly from the processor
vendor's site.

Late loading
============

There are two legacy user space interfaces to load microcode, either through
/dev/cpu/microcode or through /sys/devices/system/cpu/microcode/reload file
in sysfs.

The /dev/cpu/microcode method is deprecated because it needs a special
userspace tool for that.

The easier method is simply installing the microcode packages your distro
supplies and running::

  # echo 1 > /sys/devices/system/cpu/microcode/reload

as root.

The loading mechanism looks for microcode blobs in
/lib/firmware/{intel-ucode,amd-ucode}. The default distro installation
packages already put them there.

Late loading metadata file
==========================

New microcode blobs may remove or modify CPU feature bits. Prior to this
metadata file, the microcode was blindly loaded and might have created an
unrecoverable error (e.g. remove an instruction used currently in the kernel).

In order to improve visibility on what features a new microcode that is being
loaded at runtime (late loading) brings in, a new metadata file is created
together with the microcode blob. The metadata file has the same name as the
microcode blob with a suffix of ".metadata". The metadata file respects the
following regular expression: "{m|c} {+|-} u32 [u32]*", where "m" means MSR
feature and "c" means a CPUID exposed feature.

Here is an example of content for the metadata file::
   m + 0x00000122
   m - 0x00000120
   c + 0x00000007 0x00 0x00000000 0x021cbfbb 0x00000000 0x00000000
   c - 0x00000007 0x00 0x00000000 0x021cbfbb 0x00000000 0x00000000

The definition of the file format is as follows::
   - each line contains an action on a CPU feature that the microcode will do
   - the first letter specify the type of the feature
   - the second letter specify the operation:
   -- + - adds the feature
   -- - - removes the feature
   - the third letter specifies the index of the CPUID or the MSR
   - for the CPUID case all the others parameters specifies the
     leaf, eax, ebx, ecx and edx values

Using this metadata file, the kernel, based on its internal policies, may
deny a microcode update in order to ensure system stability (e.g. if an
instruction is removed by the microcode and that instruction is still being
used by the current code, we would drop the update as it would brake the
system).

Builtin microcode
=================

The loader supports also loading of a builtin microcode supplied through
the regular builtin firmware method CONFIG_EXTRA_FIRMWARE. Only 64-bit is
currently supported.

Here's an example::

  CONFIG_EXTRA_FIRMWARE="intel-ucode/06-3a-09 amd-ucode/microcode_amd_fam15h.bin"
  CONFIG_EXTRA_FIRMWARE_DIR="/lib/firmware"

This basically means, you have the following tree structure locally::

  /lib/firmware/
  |-- amd-ucode
  ...
  |   |-- microcode_amd_fam15h.bin
  ...
  |-- intel-ucode
  ...
  |   |-- 06-3a-09
  ...

so that the build system can find those files and integrate them into
the final kernel image. The early loader finds them and applies them.

Needless to say, this method is not the most flexible one because it
requires rebuilding the kernel each time updated microcode from the CPU
vendor is available.
