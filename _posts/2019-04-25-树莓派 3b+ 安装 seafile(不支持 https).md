---
published: true
tags:
  - 树莓派
author: persuez
---
### 系统：树莓派自带的 2019-04 版 raspberry。我是用移动硬盘来存放数据的。譬如我将一个移动硬盘分为两个区（ubuntu 下可使用 fdisk 工具) ，如下(假设硬盘设备文件为 /dev/sdb)，那么：

  ```
  1. sudo fdisk /dev/sdb # 这一步我是将其分区为 linux 格式的硬盘的，如果你不是 linux 格式的，那么接下来的硬盘挂载操作将有些稍微的不同，请自寻查找资料
  2. mkfs.ext4 /dev/sdb1 # 这一步是将分区格式化之后才能使用，这可能要 sudo，因为这一步在写这篇文章之前已经做完了，所以记不是很清了，但命令是 mkfs 系列的
  3. mkfs.ext4 /dev/sdb2
  ```
就可以了，接下来是很人性化的 prompt 操作。然后将硬盘挂载到树莓派中，我在树莓派中（没钱买屏幕，只能 ssh 了）建立了两个文件夹，分别命名为 cloud 和 software，其中 cloud 文件夹是我用来存储 seafile 的一些信息的，接着如下挂载：

  ```
  1. sudo chmod 777 cloud # 更改文件夹权限
  2. sudo chmod 777 software
  3. sudo mount /dev/sda1 cloud # 树莓派中我的分区是sda1
  4. sudo mount /dev/sda2
  接着永久挂载硬盘，这要修改 /etc/fstab 文件，并且要用到 blkid 命令。我的 /etc/fstab 文件如下：
       1	proc            /proc           proc    defaults          0       0
       2	PARTUUID=836f20f1-01  /boot           vfat    defaults          0       2
       3	PARTUUID=836f20f1-02  /               ext4    defaults,noatime  0       1
       4	# a swapfile is not a swap partition, no line here
       5	#   use  dphys-swapfile swap[on|off]  for that
       6	UUID="f7c6d434-6a4c-4fc9-a2f5-50984d1c6235"	/home/pi/cloud	ext4	defaults,noexec,nmask=o000	0	0
       7	UUID="541926e4-7180-4478-abe6-44a85e19a110"	/home/pi/software	ext4	defaults,noexec,umask=o000	0	0
  ```

其中，第6、7行是我们要修改的，其余几行是原文件就有的。其中的 UUID 就是从 sudo blkid 命令中得到的。
至此，前期工作准备好了。接下来
步骤如下：

1. 下载对应版本 [seafile](https://github.com/haiwen/seafile-rpi/releases/download/v6.3.4/seafile-server_6.3.4_stable_pi.tar.gz)， 然后解压。

  ```
  1. tar -zxvf seafile-server_6.3.4_stable_pi.tar.gz
  2. mkdir installed # 根据官方文档建立 installed 目录便于管理
  3. mv seafile-server_6.3.4_stable_pi.tar.gz installed
  ```

2. 下载 mysql-server（可能很难搞定，会出很多问题）

  ```
  1. sudo apt-get autoremove --purge mysql-server # 先清除
  2. sudo apt-get remove mysql-common
  3. sudo apt install mysql-server
  为了 mysql -u root -p 可以登录，我们要做以下操作：
  1. sudo /etc/init.d/mysql stop
  2. sudo mysqld_safe --skip-grant-tables &
  3. mysql -uroot
  4. use mysql;
  5. update user set authentication_string=PASSWORD("YOUR PASSWORD") where User='root';
  6. update user set plugin="mysql_native_password" where User='root';
  7. flush privileges;
  8. quit;
  9. sudo killall mysqld
  10. sudo /etc/init.d/mysql start
  11. 尝试 mysql -u root -p，如果输入密码登录不成功，请继续网上查找资料，不然不要往下走，亲身经验告诉你很可怕
  ```

3. 安装 seafile

      1. cd 到 seafile 解压的目录
      2. 运行 ./setup-seafile-mysql.sh (不能加 sudo)，端口号那些请默认，seafile-data 目录我是放在了 cloud 目录，也就是 /dev/sda1 挂载的目录
      3. 启动：
        1. ./seafile.sh start
        2. ./seahub.sh start

4. autossh 隧道打穿内网

  通过 apt install 安装 autossh。参考：https://github.com/ma6174/blog/issues/7。（我们这里要转发 8000 端口，VPS 中用来转发此端口的端口要开放）注意点是树莓派的用户名和 VPS 的要不一样。VPS 要开启 ssh 转发功能，即在VPS 中的/etc/ssh/sshd_config 里加上GatewayPorts yes。
  至此，你已经可以访问 seafile 了。但还有一点工作。

5. 为了可以上传文件，必须要通过 autossh 转发 8082 端口，然后通过 web 端登录 seafile，在系统管理员界面设置中更改 SERVICE_URL 为 http://example.com:[VPS 转发 8000 的端口]，FILE_SERVER_ROOT 为 http://example.com:[VPS 转发 8082 的端口]。

结束。
