# Introduction:
网上大部分关于GPU直通的文章中，都使用了带有两张显卡的主机。一张给host用另一张直通给guest主机用。  
例如核显供host用，独显直通给guest用。  
但是AMD的CPU往往不带有核显，用的电脑也往往只有一张显卡，  
所以这里的步骤对网上的文章做了些修改整合，来达到单GPU也能直通的目的。  
<b>由于只有一张显卡，直通显卡后host主机画面将不会输出到显示器,所以需要另一台主机使用ssh登陆host来进行后续操作。</b>
如果你使用的硬件和我的不同，这里的步骤大概率需要调整。  
由于这些步骤都是整合自网上的文章且没有做步骤的最小化验证，这里的步骤不一定每一步都是需要的。

# Require:

1. 一台支持 IOMMU 的主机
2. 另一台能够连接 host主机 的主机
3. windows的virtio driver (https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/)
4. windowns 10的镜像文件
5. 命令行操作的linux的能力

# Hardware:
cpu: amd ryzen r5 1600  
mb: msi b350m mortar  
mem: 8*2 ddr4
gpu: rx480 8G

# Software:
host os: lubuntu 18.04 LTS  
qemu-kvm: 2.11  
libvirt: 4.00  
guest os: windows10 2019 LTSC

# Reference:
http://mathiashueber.com/ryzen-based-virtual-machine-passthrough-setup-ubuntu-18-04/  
https://gitlab.com/YuriAlek/vfio  
https://www.reddit.com/r/VFIO/comments/616xih/gpu_passthrough_with_msi_b350_tomahawk/


# Steps:
## 准备工作
在windows下使用 GPU-Z 提取GPU的ROM, Linux的提取方法请自行google.  
安装需要的软件，在linux中执行  `sudo apt-get install libvirt-bin bridge-utils virt-manager qemu-kvm ovmf`


## 开启SVM和IOMMU
在BIOS中打开CPU的SVM和IOMMU选项，设置UEFI启动模式，而不是UEFI+Legacy模式，进入系统修改 `/etc/default/grub` 文件  
在 `GRUB_CMDLINE_LINUX_DEFAULT` 变量中追加 `amd_iommu=on iommu=pt kvm_amd.npt=1`  
例:
```
GRUB_CMDLINE_LINUX_DEFAULT="amd_iommu=on iommu=pt kvm_amd.npt=1"
```
执行sudo update-grub`  
重启电脑  
重启后执行 `dmesg | grep -i AMD-VI` 如果出现下面的输出说明开启成功
```
[ 0.792691] AMD-Vi: IOMMU performance counters supported 
[ 0.794428] AMD-Vi: Found IOMMU at 0000:00:00.2 cap 0x40 
[ 0.794429] AMD-Vi: Extended features (0xf77ef22294ada): 
[ 0.794434] AMD-Vi: Interrupt remapping enabled 
[ 0.794436] AMD-Vi: virtual APIC enabled 
[ 0.794688] AMD-Vi: Lazy IO/TLB flushing enabled 
```

## 定位显卡的IOMMU Group
IOMMU在直通时以IOMMU Group为最小单位，
所以先要找到显卡的IOMMU Group 并记录下来
在终端执行下面的脚本  
(From: https://gitlab.com/YuriAlek/vfio/blob/master/scripts/iommu.sh)
```
for iommu_group in \
  $(find /sys/kernel/iommu_groups/ -maxdepth 1 -mindepth 1 -type d); do \
  echo "IOMMU group $(basename "$iommu_group")"; \
  for device in $(ls -1 "$iommu_group"/devices/); do \
    echo -n $'\t'; lspci -nns "$device"; \
  done; \
done
```

找到显卡和USB控制器的IOMMU Group，我的机器的显示如下
```
IOMMU group 15
        1c:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon RX 470/480/570/570X/580/580X] [1002:67df] (rev c7)
        1c:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon RX 580] [1002:aaf0]
IOMMU group 18
        1d:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 00h-0fh) USB 3.0 Host Controller [1022:145c]   
```
可以看到 IOMMU Group 15 中有两个设备，  
1c:00.0 为显卡的GPU，1c:00.1 为 显卡的HDMI声卡  
这两个设备需要一起直通到主机里。

另外因为GPU直通后的guest需要外接键盘鼠标操作，所以需要额外直通一个usb控制器到guest。

记录下上面三个设备的开头的 `xx:xx.x` 部分和最后的 `[xxxx:xxxx]` 部分。  
例如我的机器就需要记录 
```
1c:00.0 1002:67df
1c:00.1 1002:aaf0
1d:00.3 1022:145c
```
后面的操作中我都会以我的PCI设备为准。你需要将这些设备号都替换成自己机器的设备号
  
## 从host上剥离显卡,并启用VFIO模块
<b>注意：这个步骤之后，主机的host上的GPU将会被剥离，导致host主机不会有任何画面输出</b>

只需要剥离显卡相关的设备即可，USB控制器在启动guest系统时由系统自动剥离

修改 `/etc/initramfs-tools/modules` 文件，添加
```
vfio
vfio_iommu_type1
vfio_virqfd
vfio_pci ids=1002:67df,1002:aaf0
```

修改 `/etc/modules` 文件，添加
```
vfio
vfio_iommu_type1
vfio_pci ids=1002:67df,1002:aaf0
```

新建 `/etc/modprobe.d/amd.conf` 文件，添加
```
softdep amdgpu pre: vfio-pci
softdep snd_hda_intel pre: vfio-pci
softdep nouveau pre: vfio-pci
softdep drm pre: vfio-pci
options kvm_amd avic=1
```

新建 `/etc/modprobe.d/vfio.conf` 文件， 添加
```
options vfio-pci ids=1002:67df,1002:aaf0
```

新建 `/etc/modprobe.d/kvm.conf` 文件， 添加
```
options kvm ignore_msrs=1
```

修改 `/etc/default/grub` 文件， 在 `GRUB_CMDLINE_LINUX_DEFAULT ` 后添加 `vfio-pci.ids=1002:67df video=efifb:off`
例如：
```
GRUB_CMDLINE_LINUX_DEFAULT="amd_iommu=on iommu=pt kvm_amd.npt=1 vfio-pci.ids=1002:67df video=efifb:off"
```
执行  
```
sudo update-initramfs -u -k all
sudo update-grub 
```
重启计算机  
通过另一台主机ssh连接到host主机上
执行 `lspci -nnv` 并找到显卡的条目  
如果显卡的条目中显示 `Kernel driver in use: vfio-pci ` 则说明显卡剥离成功

## 禁用apparmor对libvirt的访问限制
由于我们创建的的虚拟机文件并没有使用libvirt的pool和volume,
所以apparmor会禁止我们访问这些文件，需要将libvirt加入到apparmor的禁用列表中
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/usr.sbin.libvirtd
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/usr.lib.libvirt.virt-aa-helper
```

## 创建虚拟机

创建虚拟机磁盘 `qemu-img create -f raw win10.img 100G`  

修改虚拟机的xml文件中需要替换的部分(https://github.com/staroffish/kvm_gpu_passthrough/blob/master/r5_1600_rx480_ubuntu_1804/win10.xml)  
关于替换部分的详细如下

```
MEM_SIZE:               为虚拟机分配的内存大小，单位KB
CPU_CORE_CNT:           为虚拟机分配的CPU核心数
IMG_FILE_PATH:          虚拟机磁盘文件的绝对路径
WIN10_ISO_PATH:         Windows10镜像文件的绝对路径
VIRTIO_DRIVER_ISO_PATH：virtio驱动的镜像文件
GPU_BUS:                GPU的BUS号 1c:00.0 中的 1c 的部分
GPU_SLOT:               GPU的Slot号 1c:00.0 中的 00 的部分
GPU_FUNC:               GPU的function号 1c:00.0 中的 0 的部分
GPU_HDMI_BUS:           GPU HDMI的BUS号
GPU_HDMI_SLOT:          GPU HDMI的Slot号
GPU_HDMI_FUNC:          GPU HDMI的function号
USB_BUS:                USB控制器的BUS号
USB_SLOT:               USB控制器的Slot号
USB_FUNC:               USB控制器的function号
GPU_ROM_PATH:           显卡ROM文件的绝对路径
```

执行下面的命令将xml导入libvirt 并启动虚拟机
```
virsh define win10.xml
virsh start win10
``` 
此时连接host主机显卡的显示器如果有显示了，就证明我们的GPU直通成功了  

注1: 因为该xml文件中选择直接引导硬盘所以还没有安装系统的时候会卡在EFI的界面，此时输入exit退出EFI界面，可进入BIOS界面在BIOS中选择光盘启动就可以安装系统了

注2: 因为使用了virtio半虚拟化io，所以在安装windows时会找不到硬盘，需要手动加载virtio驱动 具体路径 cdrom:/viostor/w10/amd64

# Issues
1. 安装windows以后，windows更新中安装的显卡驱动可以正常运行，但是安装官网最新驱动后显示器黑屏，且重启也无法修复。但是默认驱动玩游戏问题不大(只测试了OW和hitman2)
2. CPU性能下降的比较厉害