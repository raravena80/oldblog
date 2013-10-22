---
layout: post
title: "Upgrade Linux Kernel on Chromebook"
date: 2013-10-21 22:15
comments: true
categories: [linux kernel]

---

So after installing ChrUbuntu on my Acer C7 Chromebook, 
I'm very pleased that with the help of this
 <a href="http://velvet-underscore.blogspot.com/2013/01/chrubuntu-virtualbox-with-kvm.html">blog</a>
I was able to upgrade the Linux Kernel to 3.8.11

``` bash uname -a
raravena@chromebook:~/git/blog-src$ uname -a
Linux chromebook 3.8.11 #3 SMP Thu Oct 17 07:41:20 PDT 2013 x86_64 x86_64 x86_64 GNU/Linux
```

These are the modified steps:

``` bash kernel-upgrade
#!/bin/bash
 
set -x
 
#
# Grab verified boot utilities from ChromeOS.
#
mkdir -p /usr/share/vboot
mount -o ro /dev/sda3 /mnt
cp /mnt/usr/bin/vbutil_* /usr/bin
mkdir -p /usr/bin/old_bins
cp /mnt/usr/bin/old_bins/vbutil_* /usr/bin/old_bins/.
cp /mnt/usr/bin/dump_kernel_config /usr/bin
rsync -avz /mnt/usr/share/vboot/ /usr/share/vboot/
umount /mnt
 
#
# On the Acer C7, ChromeOS is 32-bit, so the verified boot binaries need a
# few 32-bit shared libraries to run under ChrUbuntu, which is 64-bit.
#
apt-get install libc6:i386 libssl1.0.0:i386
 
#
# Fetch ChromeOS kernel sources from the Git repo.
#
apt-get install git-core
cd /usr/src
git clone  https://git.chromium.org/git/chromiumos/third_party/kernel-next.git
cd kernel-next
git checkout origin/chromeos-3.8
 
#
# Configure the kernel
#
# First we patch ``base.config`` to set ``CONFIG_SECURITY_CHROMIUMOS``
# to ``n`` ...
cp ./chromeos/config/base.config ./chromeos/config/base.config.orig
sed -e \
  's/CONFIG_SECURITY_CHROMIUMOS=y/CONFIG_SECURITY_CHROMIUMOS=n/' \
  ./chromeos/config/base.config.orig > ./chromeos/config/base.config
./chromeos/scripts/prepareconfig chromeos-intel-pineview
#
# ... and then we proceed as per Olaf's instructions
#
yes "" | make oldconfig
 
#
# Build the Ubuntu kernel packages
#
apt-get install kernel-package
make-kpkg kernel_image kernel_headers
 
#
# Backup current kernel and kernel modules
#
tstamp=$(date +%Y-%m-%d-%H%M)
dd if=/dev/sda6 of=/kernel-backup-$tstamp
cp -Rp /lib/modules/3.4.0 /lib/modules/3.4.0-backup-$tstamp
 
#
# Install kernel image and modules from the Ubuntu kernel packages we
# just created.
#
dpkg -i /usr/src/linux-*.deb
 
#
# Extract old kernel config
#
vbutil_kernel --verify /dev/sda6 --verbose | tail -1 > /config-$tstamp-orig.txt
#
# Add ``disablevmx=off`` to the command line, so that VMX is enabled (for VirtualBox & Co)
#
sed -e 's/$/ disablevmx=off/' \
  /config-$tstamp-orig.txt > /config-$tstamp.txt
 
#
# Wrap the new kernel with the verified block and with the new config.
#
vbutil_kernel --pack /newkernel \
  --keyblock /usr/share/vboot/devkeys/kernel.keyblock \
  --version 1 \
  --signprivate /usr/share/vboot/devkeys/kernel_data_key.vbprivk \
  --config=/config-$tstamp.txt \
  --vmlinuz /boot/vmlinuz-3.8.11 \
  --arch x86_64
 
#
# Make sure the new kernel verifies OK.
#
vbutil_kernel --verify /newkernel
 
#
# Copy the new kernel to the KERN-C partition.
#
dd if=/newkernel of=/dev/sda6

```

I ran into an error while compiling the kernel, but gladly was able to fix it
``` diff diffs
diff --git a/net/mac80211/tx.c b/net/mac80211/tx.c
index 467c1d1..4ba651d 100644
--- a/net/mac80211/tx.c
+++ b/net/mac80211/tx.c
@@ -1749,7 +1749,7 @@ netdev_tx_t ieee80211_subif_start_xmit(struct sk_buff *skb,
        bool multicast;
        u32 info_flags = 0;
        u16 info_id = 0;
-       struct ieee80211_chanctx_conf *chanctx_conf;
+       struct ieee80211_chanctx_conf *chanctx_conf = NULL;
        struct ieee80211_sub_if_data *ap_sdata;
        enum ieee80211_band band;
 
```
