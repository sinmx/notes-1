# Openstack Windows 镜像制作

### 下载virtio驱动文件
地址：
smb://192.168.51.100/platform/01Software/02Image/windows/virtio-win-0.1.102_amd64.vfd
smb://192.168.51.100/platform/01Software/02Image/windows/virtio-win-0.1.102.iso

> 官方下载： [https://fedoraproject.org/wiki/Windows_Virtio_Drivers](https://fedoraproject.org/wiki/Windows_Virtio_Drivers)

### 创建一个img文件
```shell
$ qemu-img create -f qcow2 win7_x64.qcow2 40G
```

### 安装系统

```
$ kvm -m 2048 -cdrom cn_windows_7_ultimate_with_sp1_x64_dvd_u_677408.iso -drive file=win7_x64.qcow2,if=virtio -fda virtio-win-0.1.102_amd64.vfd -boot d -nographic -vnc :0

------new
kvm -m 2048 -cdrom UQI_W7SP1_X64_16IN1.iso -drive file=win7_x64.qcow2,if=virtio -fda virtio-win-0.1.136_amd64.vfd -boot d -nographic -vnc :0 -usb -usbdevice tablet
```

```
virt-install --connect qemu:///system \
    --name win7vnc --ram 2048 --vcpus=2 --cpuset=auto \
    --disk path=win7.img,bus=virtio,size=100,format=qcow2 \
    --network=network=default,model=virtio,mac=RANDOM \
    --graphics vnc,port=5900
    --disk device=cdrom,path=../../isos/virtio-win-0.1-81.iso \
    --disk device=cdrom,path=../../isos/win7_sp1_ult_64bit/Windows\ 7\ SP1\ Ultimate\ \(64\ Bit\).iso \
    --os-type=windows --os-variant=win7 --boot cdrom,hd
```

**连接VNC进行系统安装**
这里的端口号根据上一步 -vnc :0 推移，如果是-vnc :1则是5901端口
```shell
$ vncview localhost:5900
```

**磁盘选择**
安装选择磁盘时显示是空的，这个时候需要手动去加载驱动
步骤： 加载驱动程序 -> 确定 -> 软盘驱动器A -> i386 -> win7 -> 继续
选择硬盘后就按照正常系统安装方式进行安装，安装完成后关闭系统。

### 安装驱动
需要安装网卡驱动、显卡驱动（qxl）、声卡驱动（ac94）

**安装网卡驱动**
```
$ kvm -m 2048 -cdrom virtio-win-0.1.102.iso -drive file=win7_x64.qcow2,if=virtio -net nic,model=virtio -net user -boot d -nographic -vnc :0

------new
kvm -m 2048 -cdrom virtio-win-0.1.136.iso -drive file=win7_x64.qcow2,if=virtio -boot d -nographic -vnc :0 -usb -usbdevice tablet  -net nic,model=virtio -net user   -soundhw ac97 -vga qxl -balloon virtio
```

**安装显卡、声卡驱动**
安装完网卡驱动后关闭电脑，用命令重新开机, 也可以和网卡驱动一起安装
```
$ kvm -m 2048 -drive file=win7_x64.qcow2,if=virtio -net nic,model=virtio -net user -soundhw ac97 -vga qxl -boot d -nographic -vnc :0
```
驱动地址：[smb://192.168.51.100/platform/01Software/03Windows/kvm drivers](smb://192.168.51.100/platform/01Software/03Windows/kvm drivers)
将驱动拷贝到系统桌面，然后在设备管理页面更新驱动程序，选择对应的驱动目录。

> 如果需要安装一些常用软件，可以再次打开系统进行安装
> 网络参数可以不加，默认即为 -net nic -net user
> 如果安装时鼠标不能移动，需要加上参数 -usb -usbdevice tablet
>  -balloon virtio  add support of ballon


### 镜像压缩
系统安装完之后可能会占用很大空间，如果对大小有要求可以使用命令压缩一下
```shell
$ qemu-img convert -c -O qcow2 win7_x64.qcow2 win7_x64_new.qcow2
$ ll -h
-rw-r--r--  1 cctdesktop cctdesktop  13G 11月 13 14:07 win7_x64.qcow2
-rw-r--r--  1 cctdesktop cctdesktop 6.3G 11月 13 14:21 win7_x64_new.qcow2
```

### 上传镜像
```shell
$ glance image-create --name="Win7 x64" --is-public=true --container-format=ovf --disk-format=qcow2 < win7_x64_new.qcow2
$ glance image-create --name F20-x86_64 --disk-format qcow2 --container-format bare --is-public True < fedora-image.qcow2

$ openstack image create "cirros" \
  --file cirros-0.3.4-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public --progress
```

### 修改windowsMTU
netsh interface ipv4 show subinterfaces
netsh interface ipv4 set subinterface "本地连接" mtu=1450 store=persistent

### 调整源磁盘大小
```shell
#qemu-img resize filename [+|-]size
```

### update administrator password and create new user
```
#ps1_sysnative
net user Administrator /active:yes
net user Administrator password
net user cpuser password /add
net localgroup Administrators cpuser /add
```

```
#ps1_sysnative
net user administrator ycxx123#
curl -o C:/4Suite-XML-1.0.1.zip http://mirrors.163.com/pypi/simple/4Suite-XML/4Suite-XML-1.0.1.zip
expand -r C:/4Suite-XML-1.0.1.zip C:/4Suite-XML-1.0.1/
winrar x -y C:/4Suite-XML-1.0.1.zip C:/4Suite-XML-1.0.1/

-and
```

当使用此命令收缩磁盘镜像之前,必须使用客户机的文件系统和分区工具来收缩文件系统和分区,然后再执行resize操作,不然会可能丢失数据。当使用此命令扩大了磁盘镜像之后,必须使用客户机的文件系统和分区工具来使用新增加的磁盘容量。这很好理解,KVM支持的客户机操作系统多种多样,而且都有成熟的文件系统和分区操作工具,resize操作只是简单的扩大或缩小磁盘镜像大小,而不能也无需来了解客户机怎么应对这个改变,这是客户机的事情。

不幸的是resize尚不支持qcow2格式的磁盘镜像收缩, 但是扩大qcow2磁盘镜像没有问题。

**使用命令查看qcow2文件信息**

```shell
$ qemu-img info win2008_r2_x64.qcow2
image: win2008_r2_x64.qcow2
file format: qcow2
virtual size: 60G (64424509440 bytes)
disk size: 7.1G
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
```


# Openstack Ubuntu 镜像制作

linux制作和windows类似，不过不需要考虑驱动之类的事情, 需要记得安装sshd服务，并开启ssh root登陆,在虚拟机镜像里的分区，其实和物理机器是有很多的区别和不同。例如你做镜像的时候，不需要swap分区，也不建议你使用lvm分区。

```
$ qemu-img create -f qcow2 ubuntu_1404 40G

$ kvm -m 1024 -cdrom ubuntu-14.04-server-amd64.iso -drive file=ubuntu_1404,if=virtio -net nic,model=virtio -net user -boot d -nographic -vnc :0

vncviewer :0

kvm -m 1024 -drive file=ubuntu_1404,if=virtio -net nic,model=virtio -net user -boot d -nographic -vnc :0
```

## Openstack 直接使用原始启动盘作为镜像

创建镜像时直接上传原始启动盘即可，不过理论上是不可以的。


```shell
----------Centos
# qemu-img create -f qcow2 centos.qcow2 10G

# virt-install --virt-type kvm --name centos --ram 1024 \
  --disk centos7.qcow2,format=qcow2 \
  --network network=default \
  --graphics vnc,listen=0.0.0.0 --noautoconsole \
  --os-type=linux --os-variant=rhel7 \
  --location=CentOS-7-x86_64-Minimal-1511.iso

qemu-img convert -c centos7.qcow2 -O qcow2 centos_7_x86_64.qcow2
```

```shell
----------Ubuntu

# qemu-img create -f qcow2 ubuntu.qcow2 10G

# virt-install --virt-type kvm --name ubuntu --ram 1024 \
  --cdrom=ubuntu-14.04.5-server-amd64.iso \
  --disk ubuntu.qcow2,format=qcow2 \
  --network network=default \
  --graphics vnc,listen=0.0.0.0 --noautoconsole \
  --os-type=linux --os-variant=ubuntutrusty

qemu-img convert -c ubuntu.qcow2 -O qcow2 Ubuntu_1404_x86_64.qcow2
```


### 命令行创建虚拟机

```
openstack server create --flavor cf1c7c26-b63d-410f-8082-b0456bb9acf9 --image 9c4a982c-97e8-4ba3-82c6-fc9401915f96 --security-group default --nic net-id=a47e560f-e306-4111-9023-9fb786ce003d,v4-fixed-ip=172.16.100.10  --availability-zone nova:compute6  clidemo
```


### Error:

bash: virt-install: command not found

sudo apt-get install virtinst



### Error

Network not found: no network with matching name 'default'

find / -name "default.xml"
sudo virsh net-define /etc/libvirt/qemu/networks/default.xml
sudo virsh net-start default




###############


## windows hd -->  virtio

虚拟机启动时候默认是没有virtio驱动的（默认为ide），从下面的xml文件也可以看出来（bus=ide）。所以无法直接导入到openstack中使用，需要手动将硬盘、网卡的virtio驱动安装上才可以正常使用.


```
virsh dumpxml winxp

...

  <devices>
    <emulator>/usr/bin/kvm-spice</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/home/yaof/win-xp-yaof.qcow2'/>
      <target dev='hda' bus='ide'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>

...

<interface type='network'>
      <mac address='52:54:00:6a:18:d2'/>
      <source network='default'/>
      <model type='rtl8139'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>

```


#### 更新磁盘virtio驱动


因为硬盘hda是启动盘，不能直接更改类型，所以需要手动添加一块硬盘，指定为virtio的，然后启动虚拟机，进入系统后，到设备管理器中的磁盘驱动器中可以看到一个驱动异常，直接更新驱动，找到virtio驱动安装即可
address字段虚拟机启动时会自动生成，不会配置可以直接删掉

```
# virsh edit winxp

...

    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/home/yaof/win-xp-yaof.qcow2'/>
      <target dev='hda' bus='ide'/>
    </disk>

    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/home/remote_iso/test.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk> 

...

```

安装驱动时需要手动挂载virtio的iso文件

```

virsh attach-disk 42 /image/virtio-win-0.1.136.iso hdb --type cdrom

```

驱动安装完成后，关闭虚拟机，将临时的硬盘再删掉，然后把启动盘的类型改为virtio的即可正常启动

```
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/home/yaof/win-xp-yaof.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>

```


#### 更新网卡virtio驱动

网卡更改为virtio的比较简单，直接把model改为virtio，然后启动虚拟机，进入系统后，到设备管理器中安装网卡驱动即可。

```

<interface type='network'>
      <mac address='52:54:00:6a:18:d2'/>
      <source network='default'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>

```

or

you can set up these parameters using glance. Lookup the id of the image you have imported , then you can set these parameters as follows:

```
glance image-update \
    --property hw_disk_bus=''ide'' \
    --property hw_cdrom_bus="ide" \
    --property hw_vif_model="rtl8139" \
    7db7f821-6be1-4345-ac08-47a938c8f33a
```

Libvirt

https://docs.openstack.org/python-glanceclient/latest/cli/property-keys.html

architecture - name of guest hardware architecture eg i686, x86_64, ppc64
hw_cdrom_bus - name of the CDROM bus to use eg virtio, scsi, ide
hw_disk_bus - name of the hard disk bus to use eg virtio, scsi, ide
hw_floppy_bus - name of the floppy disk bus to use eg fd, scsi, ide
hw_qemu_guest_agent - boolean 'yes' or 'no' to enable QEMU guest agent
hw_rng - name of the RNG device type eg virtio (pending merge)
hw_scsi_model - name of the SCSI bus controller eg 'virtio-scsi', 'lsilogic', etc (pending merge)
hw_video_model - name of the video adapter model to use, eg cirrus, vga, xen, qxl
hw_video_ram - MB of video RAM to provide eg 64 (pending merge)
hw_vif_model - name of a NIC device model eg virtio, e1000, rtl8139
hw_watchdog_action - action to take when watchdog device fires eg reset, poweroff, pause, none (pending merge)
os_command_line - string of boot time command line arguments for the guest kernel


#############

1. 通过KVM安装kali linux时一切正常，然而安装完成后启动的时候卡住 ：   
    
>    intel_rapl: no valid rapl domains found in package 0

是因为虚拟显卡的问题，只需要添加一下镜像的属性
hw_video_model: vga


https://docs.openstack.org/image-guide/convert-images.html#qemu-img-convert-raw-qcow2-qed-vdi-vmdk-vhd

https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux_OpenStack_Platform/5/html/Command-Line_Interface_Reference_Guide/chapter_cli-glance-property.html

Adding images to your OpenStack environment
https://www.ibm.com/support/knowledgecenter/en/SS4KMC_2.5.0.2/com.ibm.ico.doc_2.5/pc/pct_add_images_OpenStack.html


### Libvirt 添加磁盘

1. 创建磁盘：

```
qemu-img create -f qcow2 /data/vm/huge.img 10G
```

2. 编写一个xml文件（disk.xml）：

```
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none'/>
      <source file='/data/vm/huge.img'/>
      <target dev='vdb' bus='virtio'/>
    </disk>
```


3. 添加磁盘：

```
virsh attach-device --persistent vm-name disk.xml
```


## 导入vmdk，vdi

linux系统 vdi网卡名称会变化，手动修改后系统正常运行。

windows
