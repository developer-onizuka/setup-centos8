# setup-centos8

# 1. Chrome install
```
$ sudo dnf localinstall google-chrome-stable_current_x86_64.rpm
```

# 2. Install Nvidia-Driver
```
sudo subscription-manager repos --enable=rhel-8-for-x86_64-appstream-rpms
sudo subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms
sudo subscription-manager repos --enable=codeready-builder-for-rhel-8-x86_64-rpms
sudo dnf config-manager --add-repo=https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo
sudo dnf module install nvidia-driver:latest
```

# 3. Enable IOMMU and Load vfio-pci driver instead of Nouveau driver
```
$ sudo su
# vi /etc/default/grub 
...
GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet intel_iommu=on vfio_iommu_type1.allow_unsafe_interrupts=1 iommu=pt"
...

# cat <<EOF > /etc/modprobe.d/vfio.conf
options vfio-pci ids=10de:1cb1,10de:0fb9
EOF

# cat <<EOF > /etc/modules-load.d/vfio-pci.conf
pci_stub
vfio
vfio_iommu_type1
vfio_pci
kvm
kvm_intel
EOF

# grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg
# reboot

$ lspci -nnk -d 10de:1cb1
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP107GL [Quadro P1000] [10de:1cb1] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:11bc]
	Kernel driver in use: vfio-pci
	Kernel modules: nouveau, nvidia_drm, nvidia

$ lspci -nnk -d 10de:0fb9
01:00.1 Audio device [0403]: NVIDIA Corporation GP107GL High Definition Audio Controller [10de:0fb9] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:11bc]
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel
```

# 4. Install KVM and Vagrant
```
$ sudo dnf install -y qemu-kvm qemu-img libvirt virt-install
$ sudo dnf install -y libvirt-devel ruby-devel

$ sudo systemctl start libvirtd
$ virsh list
$ sudo dnf install -y virt-manager
$ sudo virt-manager 

$ sudo yum install -y yum-utils
$ sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
$ sudo yum -y install vagrant
$ vagrant plugin list
$ sudo dnf groupinstall "Development Tools" -y
$ CONFIGURE_ARGS="with-libvirt-include=/usr/include/libvirt with-libvirt-lib=/usr/lib64" vagrant plugin install vagrant-libvirt
```

