# Initrd 原理与定制
## 背景
## 调用过程
## 查看与制作
### 查看initrd内容
从`/boot`下拷贝`initrd`文件后可以用`file`命令看一下文件类型，一般会是`gz`文件       
- 查看initrd文件内容
    ```commandline
    Listing 1. Inspecting the initrd (prior to FC3)
    # mkdir temp ; cd temp
    # cp /boot/initrd.img.gz .
    # gunzip initrd.img.gz
    # mount -t ext -o loop initrd.img /mnt/initrd
    # ls -la /mnt/initrd
    ```  
    *使用`file`命令查看解压后的文件，如果是`cpio`类型的文件使用下面的`cpio`命令进行解压，非`cpio`类型的可以用`mount`命令挂载*
    ```commandline
    Listing 2. Inspecting the initrd (FC3 and later)
    # mkdir temp ; cd temp
    # cp /boot/initrd-2.6.14.2.img initrd-2.6.14.2.img.gz
    # gunzip initrd-2.6.14.2.img.gz
    # cpio -i --make-directories < initrd-2.6.14.2.img
    #
    ```
### 制作initrd
可选工具：mkinitrd、YAIRD
```commandline
 Numerous tools, such as mkinitrd, can be used to automatically build an initrd with the necessary libraries and 
 modules for bridging to the real root file system. The mkinitrd utility is actually a shell script, so you can see 
 exactly how it achieves its result. There's also the YAIRD (Yet Another Mkinitrd) utility, which permits customization 
 of every aspect of the initrd construction.
```
```commandline
Listing 4. Utility (mkird) to create a custom initrd
#!/bin/bash
 
# Housekeeping...
rm -f /tmp/ramdisk.img
rm -f /tmp/ramdisk.img.gz
 
# Ramdisk Constants
RDSIZE=4000
BLKSIZE=1024
 
# Create an empty ramdisk image
dd if=/dev/zero of=/tmp/ramdisk.img bs=$BLKSIZE count=$RDSIZE
 
# Make it an ext2 mountable file system
/sbin/mke2fs -F -m 0 -b $BLKSIZE /tmp/ramdisk.img $RDSIZE
 
# Mount it so that we can populate
mount /tmp/ramdisk.img /mnt/initrd -t ext2 -o loop=/dev/loop0
 
# Populate the filesystem (subdirectories)
mkdir /mnt/initrd/bin
mkdir /mnt/initrd/sys
mkdir /mnt/initrd/dev
mkdir /mnt/initrd/proc
 
# Grab busybox and create the symbolic links
pushd /mnt/initrd/bin
cp /usr/local/src/busybox-1.1.1/busybox .
ln -s busybox ash
ln -s busybox mount
ln -s busybox echo
ln -s busybox ls
ln -s busybox cat
ln -s busybox ps
ln -s busybox dmesg
ln -s busybox sysctl
popd
 
# Grab the necessary dev files
cp -a /dev/console /mnt/initrd/dev
cp -a /dev/ramdisk /mnt/initrd/dev
cp -a /dev/ram0 /mnt/initrd/dev
cp -a /dev/null /mnt/initrd/dev
cp -a /dev/tty1 /mnt/initrd/dev
cp -a /dev/tty2 /mnt/initrd/dev
 
# Equate sbin with bin
pushd /mnt/initrd
ln -s bin sbin
popd
 
# Create the init file
cat >> /mnt/initrd/linuxrc << EOF
#!/bin/ash
echo
echo "Simple initrd is active"
echo
mount -t proc /proc /proc
mount -t sysfs none /sys
/bin/ash --login
EOF
 
chmod +x /mnt/initrd/linuxrc
 
# Finish up...
umount /mnt/initrd
gzip -9 /tmp/ramdisk.img
cp /tmp/ramdisk.img.gz /boot/ramdisk.img.gz
```
### 命令字解析
- gzip压缩命令:`gzip (选项) (参数)
    - 选项：
        ```commandline
        -a或——ascii：使用ASCII文字模式；
        -d或--decompress或----uncompress：解开压缩文件；
        -f或——force：强行压缩文件。不理会文件名称或硬连接是否存在以及该文件是否为符号连接；
        -h或——help：在线帮助；
        -l或——list：列出压缩文件的相关信息；
        -L或——license：显示版本与版权信息；
        -n或--no-name：压缩文件时，不保存原来的文件名称及时间戳记；
        -N或——name：压缩文件时，保存原来的文件名称及时间戳记；
        -q或——quiet：不显示警告信息；
        -r或——recursive：递归处理，将指定目录下的所有文件及子目录一并处理；
        -S或<压缩字尾字符串>或----suffix<压缩字尾字符串>：更改压缩字尾字符串；
        -t或——test：测试压缩文件是否正确无误；
        -v或——verbose：显示指令执行过程；
        -V或——version：显示版本信息；
        -<压缩效率>：压缩效率是一个介于1~9的数值，预设值为“6”，指定愈大的数值，压缩效率就会愈高；
        --best：此参数的效果和指定“-9”参数相同；
        --fast：此参数的效果和指定“-1”参数相同。    
        ```
    - 参数：
        文件列表：制定要压缩的文件列表
    - 实例：
        ```commandline
        把test6目录下的每个文件压缩成.gz文件            
        gzip *
  
        把上例中每个压缩的文件解压，并列出详细的信息            
        gzip -dv *
  
        详细显示例1中每个压缩的文件的信息，并不解压            
        gzip -l *
  
        压缩一个tar备份文件，此时压缩文件的扩展名为.tar.gz            
        gzip -r log.tar
  
        递归的压缩目录            
        gzip -rv test6
        ```
- mount 挂载命令：mount (选项) (参数)
    - 选项
        ```commandline
        -V：显示程序版本
        -h：显示辅助讯息
        -v：显示较讯息，通常和 -f 用来除错。
        -a：将 /etc/fstab 中定义的所有档案系统挂上。
        -F：这个命令通常和 -a 一起使用，它会为每一个 mount 的动作产生一个行程负责执行。在系统需要挂上大量 NFS 档案系统时可以加快挂上的动作。
        -f：通常用在除错的用途。它会使 mount 并不执行实际挂上的动作，而是模拟整个挂上的过程。通常会和 -v 一起使用。
        -n：一般而言，mount 在挂上后会在 /etc/mtab 中写入一笔资料。但在系统中没有可写入档案系统存在的情况下可以用这个选项取消这个动作。
        -s-r：等于 -o ro
        -w：等于 -o rw
        -L：将含有特定标签的硬盘分割挂上。
        -U：将档案分割序号为 的档案系统挂下。-L 和 -U 必须在/proc/partition 这种档案存在时才有意义。
        -t：指定档案系统的型态，通常不必指定。mount 会自动选择正确的型态。
        -o async：打开非同步模式，所有的档案读写动作都会用非同步模式执行。
        -o sync：在同步模式下执行。
        -o atime、-o noatime：当 atime 打开时，系统会在每次读取档案时更新档案的『上一次调用时间』。当我们使用 flash 档案系统时可能会选项把这个选项关闭以减少写入的次数。
        -o auto、-o noauto：打开/关闭自动挂上模式。
        -o defaults:使用预设的选项 rw, suid, dev, exec, auto, nouser, and async.
        -o dev、-o nodev-o exec、-o noexec允许执行档被执行。
        -o suid、-o nosuid：
        允许执行档在 root 权限下执行。
        -o user、-o nouser：使用者可以执行 mount/umount 的动作。
        -o remount：将一个已经挂下的档案系统重新用不同的方式挂上。例如原先是唯读的系统，现在用可读写的模式重新挂上。
        -o ro：用唯读模式挂上。
        -o rw：用可读写模式挂上。
        -o loop=：使用 loop 模式用来将一个档案当成硬盘分割挂上系统。 
        ```
    参数：
- cpio解压命令：
    - 选项：
    ```commandline
    -0或--null 接受新增列控制字符，通常配合find指令的"-print0"参数使用。    
    -a或--reset-access-time 重新设置文件的存取时间。    
    -A或--append 附加到已存在的备份档中，且这个备份档必须存放在磁盘上，而不能放置于磁带机里。    
    -b或--swap 此参数的效果和同时指定"-sS"参数相同。    
    -B 将输入/输出的区块大小改成5120 Bytes。    
    -c 使用旧ASCII备份格式。    
    -C<区块大小>或--io-size=<区块大小> 设置输入/输出的区块大小，单位是Byte。    
    -d或--make-directories 如有需要cpio会自行建立目录。    
    -E<范本文件>或--pattern-file=<范本文件> 指定范本文件，其内含有一个或多个范本样式，让cpio解开符合范本条件的文件，格式为每列一个范本样式。    
    -f或--nonmatching 让cpio解开所有不符合范本条件的文件。    
    -F<备份档>或--file=<备份档> 指定备份档的名称，用来取代标准输入或输出，也能借此通过网络使用另一台主机的保存设备存取备份档。    
    -H<备份格式> 指定备份时欲使用的文件格式。    
    -i或--extract 执行copy-in模式，还原备份档。    
    -l<备份档> 指定备份档的名称，用来取代标准输入，也能借此通过网络使用另一台主机的保存设备读取备份档。    
    -k 此参数将忽略不予处理，仅负责解决cpio不同版本间的兼容性问题。    
    -l或--link 以硬连接的方式取代复制文件，可在copy-pass模式下运用。    
    -L或--dereference 不建立符号连接，直接复制该连接所指向的原始文件。    
    -m或preserve-modification-time 不去更换文件的更改时间。    
    -M<回传信息>或--message=<回传信息> 设置更换保存媒体的信息。    
    -n或--numeric-uid-gid 使用"-tv"参数列出备份档的内容时，若再加上参数"-n"，则会以用户识别码和群组识别码替代拥有者和群组名称列出文件清单。    
    -o或--create 执行copy-out模式，建立备份档。    
    -O<备份档> 指定备份档的名称，用来取代标准输出，也能借此通过网络 使用另一台主机的保存设备存放备份档。    
    -p或--pass-through 执行copy-pass模式，略过备份步骤，直接将文件复制到目的目录。    
    -r或--rename 当有文件名称需要更动时，采用互动模式。    
    -R<拥有者><:/.><所属群组>或    
    ----owner<拥有者><:/.><所属群组> 在copy-in模式还原备份档，或copy-pass模式复制文件时，可指定这些备份，复制的文件的拥有者与所属群组。    
    -s或--swap-bytes 交换每对字节的内容。    
    -S或--swap-halfwords 交换每半个字节的内容。    
    -t或--list 将输入的内容呈现出来。    
    -u或--unconditional 置换所有文件，不论日期时间的新旧与否，皆不予询问而直接覆盖。    
    -v或--verbose 详细显示指令的执行过程。    
    -V或--dot 执行指令时，在每个文件的执行程序前面加上"."号   
    --block-size=<区块大小倍乘> 设置输入/输出的区块大小，假如设置数值为5，则区块大小为2500KB，若设置成10，则区块大小为5120KB，依次类推。    
    --force-local 强制将备份档存放在本地主机。    
    --help 在线帮助。    
    --no-absolute-filenames 使用相对路径建立文件名称。    
    --no-preserve-owner 不保留文件的拥有者，谁解开了备份档，那些文件就归谁所有。    
    -only-verify-crc 当备份档采用CRC备份格式时，可使用这项参数检查备份档内的每个文件是否正确无误。    
    --quiet 不显示复制了多少区块。    
    --sparse 倘若一个文件内含大量的连续0字节，则将此文件存成稀疏文件。    
    --version 显示版本信息。
    ```        

## 其他
        
> 参考：
> [Linux initial RAM disk (initrd) overview](https://www.ibm.com/developerworks/linux/library/l-initrd/index.html)    