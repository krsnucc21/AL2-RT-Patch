# Overview

This document provides how to apply the real time patch to Amazon Linux 2. The version of AL2 and RT patch is 5.4.91 and RT50. The AL2 with 5.4.91 kernel can be found with the AMI, amazon-eks-arm64-node-1.19-v20210208 (the AMI ID is ami-095bac9f3406565d2 in the N. Virginia region. The ID would be different if your region is not N. Virginia)

The RT patch for kernel 5.4.91 can be found at [kernel.org](https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/5.4/older/patch-5.4.91-rt50.patch.xz).

In this document, AWS Graviton2 instances will be used. Graviton2 is a 64-bit ARM CPU with Neoverse technology. You can find more information about Graviton2 [here](https://aws.amazon.com/ec2/graviton/).

# Step 1: check the kernel version

After bringing up the Graviton2 instance with the AL2 AMI above, the kernel version would look like the following:
```base
% uname -a
Linux ip-172-31-31-25.us-west-1.compute.internal 5.4.91-41.139.amzn2.aarch64 #1 SMP Tue Jan 19 20:10:58 UTC 2021 aarch64 aarch64 aarch64 GNU/Linux
```

The boot command line would be like:
```base
% cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-5.4.91-41.139.amzn2.aarch64 root=UUID=53a36bec-2f52-4183-8f7f-3acfb060d4b3 ro console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 nvme_core.io_timeout=4294967295 rd.emergency=poweroff rd.shell=0 LANG=en_US.UTF-8 KEYTABLE=us
```

# Step 2: Prepare tools

If you haven't done kernel builds before, you may need to install the following development packages.

1) kernel build tools:
```bash
sudo yum-builddep -y kernel
```
2) nurses needs to enable Linux menuconfig:
```bash
sudo yum install -y ncurses-devel
```
3) [rt_tests](https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/rt-tests) is a tool to test the real time task scheduling. We will use [cyclic test](https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest/start) to measure the wake-up latency after applying the RT patch to the kernel.
```base
sudo yum install -y git
```

# Step 3: Download the kernel source and dependencies

The kernel source code and the real time patch can be downloaded by yumdownloader and wget:
```base
% yumdownloader --source kernel-5.4.91
% wget https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/5.4/older/patch-5.4.91-rt50.patch.xz
```

To retrieve the source code from the downloaded RPM, rpm command can be used as like the following:
```base
% rpm -ivh ./kernel-5.4.91-*.rpm
% cd ~/rpmbuild/SPECS
% rpmbuild -bp kernel.spec
```
Note that the rpmbuild command will make proper changes and a configuration for your instance.

# Step 4: Configure the kernel with the real-time patch

In this step, we apply the real-time patch to the kernel. First, we make .config file as follows:
```bash
% cd ~/rpmbuild/BUILD/kernel*/linux*/
% make oldconfig
```

Then, the standard kernel features should be on for full real-time scheduling. They can be turned on by menuconfig:
```bash
% make menuconfig
```
Please go to General setup and on "Configure standard kernel features (expert users)"

Now, the real-time patch code is applied:
```base
% xzcat ~/patch-5.4.91-rt50.patch.xz | patch -p1
```
You should choose 'Fully Preemptible Kernel' by oldconfig:
```bash
% make oldconfig
```

That must be shown first when you run oldconfig. Or, you can choose in by menuconfig by going to General setup, and select Preemption Model (Fully Preemptible Kernel (Real-Time))

# Step 5: apply the required code changes

Please note that the real-time patch have several conflicts with the AL2 source code. To fix these conflicts, the following code changes should be applied:
## Code change #1
```bash
% vi kernel/printk/printk.c
---
"kernel/printk/printk.c" line 2506 of 3313
void register_console(struct console *newcon)
{
        //unsigned long flags;
        //remove this since the variable flags is not used
---
```
## Code change #2
```bash
% vi mm/page_alloc.c
---
"mm/page_alloc.c" line 1426 of 8868
#if 0
remove the below according to RT50
        spin_lock(&zone->lock);
        isolated_pageblocks = has_isolate_pageblock(zone);
...
        spin_unlock(&zone->lock);
#endif
}
---
"mm/page_alloc.c" line 1358 of 8868
                //__free_one_page(page, page_to_pfn(page), zone, 0, mt);
                // add "true" as the last parameter to fix a build error
                __free_one_page(page, page_to_pfn(page), zone, 0, mt, true);
---
```
## Code change #3
```bash
% vi drivers/amazon/net/ena/kcompat.h
---
"drivers/amazon/net/ena/kcompat.h" line 335 of 809
// remove the following function due to duplication definition
/*
static inline void skb_mark_napi_id(struct sk_buff *skb,
                                    struct napi_struct *napi)
{

}

static inline void napi_hash_del(struct napi_struct *napi)
{

}
*/
---
```
## Code change #4
```bash
% vi fs/libfs.c
---
"fs/libfs.c" line 97 of 1294
        //unsigned *seq = &parent->d_inode->i_dir_seq, n;
        //change i_dir_seq
        unsigned *seq = &parent->d_inode->__i_dir_seq, n;
---
"fs/libfs.c" line 133 of 1296
        //unsigned n, *seq = &parent->d_inode->i_dir_seq;
        //change i_dir_seq
        unsigned n, *seq = &parent->d_inode->__i_dir_seq;
---
```

# Step 6: Compile the kernel

Now, the kernel source code is ready to build. You can run regular kernel build commands like the following:
```bash
% make -j 64 # c6g.metal has 64 cores
% sudo make modules_install -j 64
% sudo make install -j 64
```

# Step 7: Tell grub to reboot into the newly compiled kernel

Then, configure the grub to start with the new kernel. The following steps are optional and only needed when you don't have 'GRUB_DISABLE_SUBMENU=y' in your grub file. If you have that, you can skip this and go to the next step.
```base
% sudo cat /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 nvme_core.io_timeout=4294967295 rd.emergency=poweroff rd.shell=0"
GRUB_TIMEOUT=0
GRUB_DISABLE_RECOVERY="true"

# If the default grub config does not have GRUB_DISABLE_SUBMENU, add it
% echo -e "\nGRUB_DISABLE_SUBMENU=y" | sudo tee -a /etc/default/grub
% sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

In this step, we set the new kernel as default to boot:
```bash
% sudo grubby --set-default /boot/vmlinuz-5.4.91-rt50
```

You can check if the new kernel is default as follows:
```bash
% sudo grubby --default-kernel
```

Great! You have completed all the steps. You can reboot the instance and see if the new kernel brings up correctly.

# Step 8: Check the boot kernel image

After the instance comes up, you can check the kernel version. That would look like the following:
```bash
% uname -a
Linux ip-172-31-31-25.us-west-1.compute.internal 5.4.91-rt50 #2 SMP PREEMPT_RT Sun May 9 00:28:54 UTC 2021 aarch64 aarch64 aarch64 GNU/Linux
```
