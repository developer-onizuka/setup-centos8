# setup-centos8

# 1. Chrome install on Host Machine CentOS8
```
$ sudo dnf localinstall google-chrome-stable_current_x86_64.rpm
```

# 2. Install Nvidia-Driver on Host Machine CentOS8
```
$ sudo subscription-manager repos --enable=rhel-8-for-x86_64-appstream-rpms
$ sudo subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms
$ sudo subscription-manager repos --enable=codeready-builder-for-rhel-8-x86_64-rpms
$ sudo dnf config-manager --add-repo=https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo
$ sudo dnf module install -y nvidia-driver:latest
```

# 3. Enable IOMMU and Load vfio-pci driver instead of Nvidia driver on Host Machine CentOS8
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

# 4. Install KVM on Host Machine CentOS8
```
$ sudo dnf install -y qemu-kvm qemu-img libvirt virt-install
$ sudo systemctl start libvirtd
$ virsh list
$ sudo dnf install -y virt-manager
$ sudo virt-manager 
```

# 5. Install Ubuntu as a Virtual Machine with KVM
See also https://github.com/developer-onizuka/nvidia-docker_VirtualMachine2

```
$ virsh edit ubuntu20.04-gpu
・・・
  <features>
  ・・・
    <hyperv>
      <vendor_id state='on' value='whatever'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
  ・・・
  </features>
・・・
```

# 6. Install Vagrant
```
$ dnf update -y
$ dnf install -y @virt virt-install
$ dnf install -y ruby ruby-devel
$ ruby -v
$ dnf install -y make gcc rpm-build ruby-devel zlib-devel
$ dnf install -y rsync
$ gem install nokogiri
$ wget https://releases.hashicorp.com/vagrant/2.2.9/vagrant_2.2.9_x86_64.rpm
$ dnf install -y vagrant_2.2.9_x86_64.rpm
$ vagrant --version
$ dnf install -y libvirt-devel
$ CONFIGURE_ARGS='with-ldflags=-L/opt/vagrant/embedded/lib with-libvirt-include=/usr/include/libvirt with-libvirt-lib=/usr/lib' vagrant plugin install vagrant-libvirt

$ dnf download --source libssh
$ dnf install -y cmake
$ rpm2cpio libssh-0.9.0-5.fc30.src.rpm | cpio -imdV
$ rpm2cpio libssh-0.9.4-2.el8.src.rpm | cpio -imdV
$ tar xf libssh-0.9.4.tar.xz 
$ mkdir build
$ cd build/
$ cmake ../libssh-0.9.4 -DOPENSSL_ROOT_DIR=/opt/vagrant/embedded/
$ make
$ cp lib/libssh* /opt/vagrant/embedded/lib64
```

# 7. Run Vagrant
```
$ mkdir centos
$ cd centos/
$ vagrant box add centos/7 --provider=libvirt
$ vagrant init centos/7
$ vagrant up --provider=libvirt
```
