amazon-eks-arm64-node-1.19-v20210208 (ami-095bac9f3406565d2)
Linux ip-172-31-31-25.us-west-1.compute.internal 5.4.91-41.139.amzn2.aarch64 #1 SMP Tue Jan 19 20:10:58 UTC 2021 aarch64 aarch64 aarch64 GNU/Linux

BOOT_IMAGE=/boot/vmlinuz-5.4.91-41.139.amzn2.aarch64 root=UUID=53a36bec-2f52-4183-8f7f-3acfb060d4b3 ro console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 nvme_core.io_timeout=4294967295 rd.emergency=poweroff rd.shell=0 LANG=en_US.UTF-8 KEYTABLE=us

###########
# Prepare the tools
###########
sudo yum-builddep -y kernel
sudo yum install -y ncurses-devel <= make menuconfig
sudo yum install -y git <= rt_tests
sudo yum install -y vim <= vimdiff

###########
# Download the kernel source and dependencies
###########
yumdownloader --source kernel-5.4.91
wget https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/5.4/older/patch-5.4.91-rt50.patch.xz

rpm -ivh ./kernel-5.4.91-*.rpm
cd ~/rpmbuild/SPECS
rpmbuild -bp kernel.spec

###########
# Compile the kernel with oldconfig and install
###########
cd ~/rpmbuild/BUILD/kernel*/linux*/
make oldconfig
make menuconfig
=> General setup ->[*] Configure standard kernel features (expert users)

  xzcat ~/patch-5.4.91-rt50.patch.xz | patch -p1
  make oldconfig
  => General setup -> Preemption Model (Fully Preemptible Kernel (Real-Time))

  vi kernel/printk/printk.c
  ---
  "kernel/printk/printk.c" line 2506 of 3313
void register_console(struct console *newcon)
{
        //unsigned long flags;
        //remove this since the variable flags is not used
  ---
  vi mm/page_alloc.c
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
  "fs/libfs.c" line 97 of 1294
        //unsigned *seq = &parent->d_inode->i_dir_seq, n;
        //change i_dir_seq
        unsigned *seq = &parent->d_inode->__i_dir_seq, n;
  "fs/libfs.c" line 133 of 1296
        //unsigned n, *seq = &parent->d_inode->i_dir_seq;
        //change i_dir_seq
        unsigned n, *seq = &parent->d_inode->__i_dir_seq;

  ---

make -j 64 # c6g.metal has 64 cores
sudo make modules_install -j 64
sudo make install -j 64

###########
# Tell grub to reboot into newly compiled kernel
###########
sudo cat /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 nvme_core.io_timeout=4294967295 rd.emergency=poweroff rd.shell=0"
GRUB_TIMEOUT=0
GRUB_DISABLE_RECOVERY="true"

# If the default grub config does not have GRUB_DISABLE_SUBMENU, add it
echo -e "\nGRUB_DISABLE_SUBMENU=y" | sudo tee -a /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Check the info for the newly compiled kernel
sudo grubby --info /boot/vmlinuz-5.4.91
index=1
kernel=/boot/vmlinuz-5.4.91
args="ro  console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 nvme_core.io_timeout=4294967295 rd.emergency=poweroff rd.shell=0"
root=UUID=53a36bec-2f52-4183-8f7f-3acfb060d4b3
initrd=/boot/initramfs-5.4.91.img
title=Amazon Linux (5.4.91) 2

  sudo grubby --info /boot/vmlinuz-5.4.91-rt50
  index=1
  kernel=/boot/vmlinuz-5.4.91-rt50
  args="ro  console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 nvme_core.io_timeout=4294967295 rd.emergency=poweroff rd.shell=0"
  root=UUID=53a36bec-2f52-4183-8f7f-3acfb060d4b3
  initrd=/boot/initramfs-5.4.91-rt50.img
  title=Amazon Linux (5.4.91-rt50) 2

# Set the new kernel as default
sudo grubby --set-default /boot/vmlinuz-5.4.91

  sudo grubby --set-default /boot/vmlinuz-5.4.91-rt50

sudo grubby --default-kernel

# Reboot
sudo reboot

###########
# Check the boot kernel image. It should be /boot/vmlinuz-5.4.91
###########
uname -a
Linux ip-172-31-31-25.us-west-1.compute.internal 5.4.91 #1 SMP Sat May 8 22:01:11 UTC 2021 aarch64 aarch64 aarch64 GNU/Linux

Linux ip-172-31-31-25.us-west-1.compute.internal 5.4.91-rt50 #2 SMP PREEMPT_RT Sun May 9 00:28:54 UTC 2021 aarch64 aarch64 aarch64 GNU/Linux

###########
# Add boot cmdline
###########
sudo vi /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT
<=
no_timer_check rcu_nocbs=2-61 rcu_nocb_poll=1 nohz=on nohz_full=2-61 isolcpus=2-61 irqaffinity=0-1,62-63 selinux=0 enforcing=0 noswap default_hugepagesz=1G hugepagesz=1G hugepages=30 mce=off audit=0 crashkernel=auto nmi_watchdog=0 fsck.mode=force fsck.repair=yes skew_tick=1 softlockup_panic=0 idle=poll nosoftlockup pcie_aspm.policy=performance

sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo grubby --set-default /boot/vmlinuz-5.4.91-rt50
sudo grubby --default-kernel
sudo reboot

cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-5.4.91-41.139.amzn2.aarch64 root=UUID=53a36bec-2f52-4183-8f7f-3acfb060d4b3 ro console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 nvme_core.io_timeout=4294967295 rd.emergency=poweroff rd.shell=0 no_timer_check rcu_nocbs=2-61 rcu_nocb_poll=1 nohz=on nohz_full=2-61 isolcpus=2-61 irqaffinity=0-1,62-63 selinux=0 enforcing=0 noswap default_hugepagesz=1G hugepagesz=1G hugepages=30 mce=off audit=0 crashkernel=auto nmi_watchdog=0 fsck.mode=force fsck.repair=yes skew_tick=1 softlockup_panic=0 idle=poll nosoftlockup pcie_aspm.policy=performance

###########
# Test CPU affinity
###########
cat /sys/devices/system/cpu/isolated
2-61
cat /sys/devices/system/cpu/present
0-63
taskset -cp 1
pid 1's current affinity list: 0,1,62,63
lscpu | grep CPU.s
CPU(s):                          64
On-line CPU(s) list:             0-63
NUMA node0 CPU(s):               0-63
cat /proc/1/status|tail -6
Cpus_allowed:	c0000000,00000003
Cpus_allowed_list:	0-1,62-63
Mems_allowed:	1
Mems_allowed_list:	0
voluntary_ctxt_switches:	2523
nonvoluntary_ctxt_switches:	158
cat /proc/irq/32/smp_affinity
80000000,00000000

###########
# Test rt_tests
# https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/rt-tests
###########
sudo yum install -y time

git clone git://git.kernel.org/pub/scm/utils/rt-tests/rt-tests.git
cd rt-tests
git checkout stable/v1.0
make all
sudo make install

sudo time ./cyclictest -S -m -n -p49 -i 100 -d0 -A ffff

###########
# Optimize with tuned
# https://tuned-project.org/docs/tuned_devconf_2019.pdf
###########
sudo yum install -y tuned
tuned-adm --version
tuned-adm 2.8.0

sudo mkdir /etc/tuned/realtime
sudo vi /etc/tuned/realtime/tuned.conf
# https://github.com/redhat-performance/tuned/blob/master/profiles/realtime/tuned.conf
#
# tuned configuration
#
# Red Hat Enterprise Linux for Real Time Documentation:
# https://docs.redhat.com

[main]
summary=Optimize for realtime workloads
include = network-latency

[variables]
# User is responsible for updating variables.conf with variable content such as isolated_cores=X-Y 
include = /etc/tuned/realtime-variables.conf

isolated_cores_assert_check = \\${isolated_cores}
# Make sure isolated_cores is defined before any of the variables that
# use it (such as assert1) are defined, so that child profiles can set
# isolated_cores directly in the profile (tuned.conf)
isolated_cores = ${isolated_cores}
# Fail if isolated_cores are not set
assert1=${f:assertion_non_equal:isolated_cores are set:${isolated_cores}:${isolated_cores_assert_check}}

# Non-isolated cores cpumask including offline cores
not_isolated_cpumask = ${f:cpulist2hex_invert:${isolated_cores}}
isolated_cores_expanded=${f:cpulist_unpack:${isolated_cores}}
isolated_cpumask=${f:cpulist2hex:${isolated_cores_expanded}}
isolated_cores_online_expanded=${f:cpulist_online:${isolated_cores}}

# Fail if isolated_cores contains CPUs which are not online
assert2=${f:assertion:isolated_cores contains online CPU(s):${isolated_cores_expanded}:${isolated_cores_online_expanded}}

# Assembly managed_irq
# Make sure isolate_managed_irq is defined before any of the variables that
# use it (such as managed_irq) are defined, so that child profiles can set
# isolate_managed_irq directly in the profile (tuned.conf)
isolate_managed_irq = ${isolate_managed_irq}
managed_irq=${f:regex_search_ternary:${isolate_managed_irq}:\b[y,Y,1,t,T]\b:managed_irq,domain,:}

[net]
channels=combined ${f:check_net_queue_count:${netdev_queue_count}}

[sysctl]
kernel.hung_task_timeout_secs = 600
kernel.nmi_watchdog = 0
kernel.sched_rt_runtime_us = -1
vm.stat_interval = 10
kernel.timer_migration = 0

[sysfs]
/sys/bus/workqueue/devices/writeback/cpumask = ${not_isolated_cpumask}
/sys/devices/virtual/workqueue/cpumask = ${not_isolated_cpumask}
/sys/devices/system/machinecheck/machinecheck*/ignore_ce = 1

[bootloader]
cmdline_realtime=+isolcpus=${managed_irq}${isolated_cores} intel_pstate=disable nosoftlockup tsc=nowatchdog

[irqbalance]
banned_cpus=${isolated_cores}

[script]
script = ${i:PROFILE_DIR}/script.sh

[scheduler]
isolated_cores=${isolated_cores}

[rtentsk]
---

sudo vi /etc/tuned/realtime-variables.conf
# Examples:
isolated_cores=2-61
# isolated_cores=2-23
#
isolate_managed_irq=Y
---

sudo systemctl enable --now tuned
systemctl status tuned
sudo tuned-adm list

sudo tuned-adm profile realtime
sudo time ./cyclictest -S -m -n -p49 -i 100 -d0 -A ffff
