# setup-centos8

# 0. Hardware
```
   (1) Precision Tower 3620
       Intel(R) Core(TM) i5-7600 CPU @ 3.50GHz
       DIMM slot1: DDR4 DIMM 16GB (KLEVV)
       DIMM slot2: DDR4 DIMM 16GB (KLEVV)
       DIMM slot3: Empty
       DIMM slot4: Empty
   (2) NVMe SSD
       KLEVV SSD 256GB CRAS C710 M.2 Type2280 PCIe3x4 NVMe 3D TLC NAND Flash
       P/N: K256GM2SP0-C71
       Performance Spec: Read 1950MB/s, Write 1250MB/s
   (3) NVIDIA Quadro P1000
       For Pass Through at Virtual Machine (do not use as VGA but for GPGPU)
   (4) NVIDIA Quadro K600
       For VGA at Host Machine for Dual Display
   (5) Web Camera
       Logitech, Inc. Webcam C270
```

# 1. Chrome install on Host Machine CentOS8
```
$ sudo sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-Linux-*
$ sudo sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-Linux-*
$ sudo dnf localinstall -y google-chrome-stable_current_x86_64.rpm
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
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=cl/root rhgb quiet rd.driver.blacklist=nouveau intel_iommu=on vfio_iommu_type1.allow_unsafe_interrupts=1 iommu=pt"
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
$ wget https://releases.ubuntu.com/20.04/ubuntu-20.04.3-desktop-amd64.iso
$ virt-manager
  ---> Making virtual machine named as "nvidia-docker" with GPU.
```

```
$ virsh edit nvidia-docker
?????????
  <features>
  ?????????
    <hyperv>
      <vendor_id state='on' value='whatever'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
  ?????????
  </features>
?????????
```

# 6. Install Vagrant
```
$ sudo dnf update -y
$ sudo dnf install -y @virt virt-install
$ sudo dnf install -y ruby ruby-devel
$ ruby -v
$ sudo dnf install -y make gcc rpm-build ruby-devel zlib-devel
$ gem install nokogiri
$ wget https://releases.hashicorp.com/vagrant/2.2.9/vagrant_2.2.9_x86_64.rpm
$ sudo dnf install -y vagrant_2.2.9_x86_64.rpm
$ vagrant --version
$ sudo dnf install -y libvirt-devel
$ CONFIGURE_ARGS='with-ldflags=-L/opt/vagrant/embedded/lib with-libvirt-include=/usr/include/libvirt with-libvirt-lib=/usr/lib' vagrant plugin install vagrant-libvirt
```

You might have the errors like below while vagrant up --provider=libvirt:

lib64/libcrypto.so.1.1: version `OPENSSL_1_1_1b' not found (required by /lib64/libssh.so.4) - /root/.vagrant.d/gems/2.6.6/gems/ruby-libvirt-0.7.1/lib/_libvirt.so
-----
lib64/libk5crypto.so.3: undefined symbol: EVP_KDF_ctrl, version OPENSSL_1_1_1b - /root/.vagrant.d/gems/2.6.6/gems/ruby-libvirt-0.7.1/lib/_libvirt.so
-----


The followings are for the measures to take.
```
$ sudo dnf groupinstall "Development Tools" -y
$ sudo dnf install -y cmake
$ mkdir vagrant_patch
$ cd vagrant_patch
$ sudo dnf download --source libssh
$ rpm2cpio libssh-0.9.4-2.el8.src.rpm | cpio -imdV
$ tar xf libssh-0.9.4.tar.xz 
$ mkdir build
$ cd build/
$ cmake ../libssh-0.9.4 -DOPENSSL_ROOT_DIR=/opt/vagrant/embedded/
$ make
$ sudo cp lib/libssh* /opt/vagrant/embedded/lib64

$ wget http://vault.centos.org/8.2.2004/BaseOS/Source/SPackages/krb5-1.17-18.el8.src.rpm
$ rpm2cpio krb5-1.17-18.el8.src.rpm | cpio -imdV
$ tar xf krb5-1.17.tar.gz 
$ cd krb5-1.17/src/
$ LDFLAGS='-L/opt/vagrant/embedded/' ./configure
$ make
$ sudo cp lib/libk5crypto.so.3 /opt/vagrant/embedded/lib64/
```

# 7. Run Vagrant test
```
$ mkdir centos
$ cd centos/
$ vagrant box add centos/7 --provider=libvirt
$ vagrant init centos/7
$ vagrant up --provider=libvirt
```

# 8. Run Vagrant and GPU workloads
You might use Vagrant file. See also https://github.com/developer-onizuka/nvidia-docker_VirtualMachine2 .
```
$ git clone https://github.com/developer-onizuka/nvidia-docker_VirtualMachine2
$ cd nvidia-docker_VirtualMachine2
$ vagrant up --provider=libvirt
```

