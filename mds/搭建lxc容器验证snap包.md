<center>搭建lxc/lxd容器验证snap包</center>

&emsp;&emsp;lxc是Linux Container的简写，它是一种内核虚拟化技术，可以提供轻量级的虚拟化，以便隔离进程和资源；它不需要提供指令解释机制，没有全虚拟化的复杂性，相当于C++中的NameSpace。lxc容器能有效地把操作系统管理的资源划分到不同的组中，并能在不同的组之间平衡有冲突的资源使用需求，因此它可以在单一的主机节点上同时执行多个相互隔离的容器。lxd是基于lxc构筑的容器管理进程，提供镜像、网络、存储、以及容器等能力。  
&emsp;&emsp;大家可能有个疑问，为什么不用docker容器呢？docker容器原先也是我的首选，但实际操作过程中发现snap包安装所需要的squashfs文件系统在docker中无法mount，会出现如下错误：  

      system does not fully support snapd: cannot mount squashfs imag  
&emsp;&emsp;所以大家不要再尝试用docker容器了。  

&emsp;&emsp;下面我们开始在Ubuntu 16.04上搭建lxc容器来验证nginx snap包。  

1.软件安装  
&emsp;&emsp;lxc/lxd容器需要安装lxd软件。  

      sudo apt install lxd
&emsp;&emsp;lxd安装好之后，再进行lxd初始化。  

      $ sudo lxd init
      Name of the storage backend to use (dir or zfs): zfs
      Create a new ZFS pool (yes/no)? yes
      Name of the new ZFS pool: pool
      Would you like to use an existing block device (yes/no)? no
      Size in GB of the new loop device (1GB minimum): 30
      Would you like LXD to be available over the network (yes/no)? no
      Do you want to configure the LXD bridge (yes/no)? yes 
      ......
&emsp;&emsp;lxd初始化的参数都采用默认值即可，并根据自己的网络情况配置网桥的ipv4和ipv6地址。如果安装时，出现缺失zfs文件系统错误，安装一下即可。  

      sudo apt install zfsutils-linux
&emsp;&emsp;lxd安装完成后，用ifconfig命令可以看到网络中多了一个名称为lxdbr0的网桥设备，后续创建的所有的lxc容器，ip地址都需要配置在网桥的这个网段中，在本文中创建容器时，我会详细说明。    

&emsp;&emsp;我们再确认一下lxd是否已经启动。  

      pp@pp-ThinkPad-S2-2nd-Gen:~$ ps -A | grep "lxd"
       3179 ?        00:00:10 lxd
       3243 ?        00:00:00 lxd

2.配置ubuntu容器  
2.1 下载镜像文件    
&emsp;&emsp;和docker类似，lxc配置容器前也需要下载对应的image镜像文件。  
&emsp;&emsp;lxc的image镜像下载比较慢，我们可以添加清华的remote链接，加速下载。  

      lxc remote add tuna-images https://mirrors.tuna.tsinghua.edu.cn/lxc-images/ --protocol=simplestreams --public
&emsp;&emsp;添加后，我们看一下remote链接。  

      pp@pp-ThinkPad-S2-2nd-Gen:~$ lxc remote list
      +-----------------+--------------------------------------------------+---------------+--------+--------+
      |      NAME       |                       URL                        |   PROTOCOL    | PUBLIC | STATIC |
      +-----------------+--------------------------------------------------+---------------+--------+--------+
      | local (default) | unix://                                          | lxd           | NO     | YES    |
      +-----------------+--------------------------------------------------+---------------+--------+--------+
      | tuna-images     | https://mirrors.tuna.tsinghua.edu.cn/lxc-images/ | simplestreams | YES    | NO     |
      +-----------------+--------------------------------------------------+---------------+--------+--------+
      | ubuntu          | https://cloud-images.ubuntu.com/releases         | simplestreams | YES    | YES    |
      +-----------------+--------------------------------------------------+---------------+--------+--------+
      | ubuntu-daily    | https://cloud-images.ubuntu.com/daily            | simplestreams | YES    | YES    |
      +-----------------+--------------------------------------------------+---------------+--------+--------+

&emsp;&emsp;添加好之后，我们在清华的remote链接上找找ubuntu的镜像。  

      pp@pp-ThinkPad-S2-2nd-Gen:~$ lxc image list tuna-images: | grep "ubuntu"
      ......
      | ubuntu/14.04 (7 more)                | 556ec728021d | yes    | Ubuntu trusty amd64 (20200203_07:42)         | x86_64  | 75.20MB  | Feb 3, 2020 at 12:00am (UTC)  |
      ......
      | ubuntu/16.04 (7 more)                | 52aeaeb9a861 | yes    | Ubuntu xenial amd64 (20200203_07:42)         | x86_64  | 80.65MB  | Feb 3, 2020 at 12:00am (UTC)  |
      ......
      | ubuntu/18.04 (7 more)                | c1347eac69d7 | yes    | Ubuntu bionic amd64 (20200203_07:42)         | x86_64  | 94.22MB  | Feb 3, 2020 at 12:00am (UTC)  |
      ......
&emsp;&emsp;可以看到目前常用的ubuntu版本镜像都有。  
&emsp;&emsp;我们下载ubuntu 18.04版本的远程镜像到本地，镜像别名为ubuntu-snaptest。  

      sudo lxc image copy ubuntu:18.04 local: --alias ubuntu-snaptest
&emsp;&emsp;下载好之后，我们看看本地的image镜像。  

      pp@pp-ThinkPad-S2-2nd-Gen:~$ lxc image list
      +-----------------+--------------+--------    +---------------------------------------------+--------+----------  +------------------------------+
      |      ALIAS      | FINGERPRINT  | PUBLIC |                 DESCRIPTION                 |  ARCH  |   SIZE   |         UPLOAD DATE          |
      +-----------------+--------------+--------+---------------------------------------------+--------+----------+------------------------------+
      | ubuntu-snaptest | 979ff60086ca | no     | ubuntu 18.04 LTS amd64 (release) (20200107) | x86_64 | 178.70MB | Jan 10, 2020 at 9:22am (UTC) |
      +-----------------+--------------+--------+---------------------------------------------+--------+----------+------------------------------+

2.2 创建ubuntu容器实例  
&emsp;&emsp;我们使用下载好的ubuntu 18.04镜像来创建ubuntu容器实例。  

      pp@pp-ThinkPad-S2-2nd-Gen:~$ lxc launch ubuntu-snaptest snapts
      Creating snapts
      Starting snapts
      pp@pp-ThinkPad-S2-2nd-Gen:~$ lxc list
      +---------+---------+--------------------+------+------------+-----------+
      |  NAME   |  STATE  |        IPV4        | IPV6 |    TYPE    | SNAPSHOTS |
      +---------+---------+--------------------+------+------------+-----------+
      | snapts  | RUNNING |                    |      | PERSISTENT | 0         |
      +---------+---------+--------------------+------+------------+-----------+
&emsp;&emsp;可以看到ubuntu 18.04容器实例已经创建成功，但没有IP地址，还需要进入ubuntu 18.04容器实例配置IP地址，否则容器实例将无法访问访问网络。容器实例的IP地址需要配置在前文所述的lxdbr0网桥IP地址的网段内。  

      pp@pp-ThinkPad-S2-2nd-Gen:~$ lxc exec snapts bash
      root@snapts:~# cd /etc/netplan/
      root@snapts:/etc/netplan# ls
      50-cloud-init.yaml
&emsp;&emsp;ubuntu 18.04容器实例配置网络需要修改这个yaml文件，编辑输入一下内容。  

      # This file is generated from information provided by the datasource.  Changes
      # to it will not persist across an instance reboot.  To disable cloud-init's
      # network configuration capabilities, write a file
      # /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the       following:
      # network: {config: disabled}
      network:
          version: 2
          ethernets:
              eth0:
                   dhcp4: no
                   addresses: [10.10.8.100/8]
                   gateway4: 10.10.8.1
                   nameservers:
                         addresses: [8.8.8.8]
&emsp;&emsp;我电脑上配置网桥ip地址为10.10.8.1，所以容器实例的ip地址配置为同一个网段的10.10.8.100。配置好ip地址之后，重启容器实例。  

      pp@pp-ThinkPad-S2-2nd-Gen:~$ lxc stop snapts
      pp@pp-ThinkPad-S2-2nd-Gen:~$ lxc start snapts
      pp@pp-ThinkPad-S2-2nd-Gen:~$ lxc list
      +---------+---------+--------------------+------+------------+-----------+
      |  NAME   |  STATE  |        IPV4        | IPV6 |    TYPE    | SNAPSHOTS |
      +---------+---------+--------------------+------+------------+-----------+
      | snapts  | RUNNING | 10.10.8.100 (eth0) |      | PERSISTENT | 0         |
      +---------+---------+--------------------+------+------------+-----------+  
&emsp;&emsp;容器实例配置的IP地址已经生效。  
&emsp;&emsp;ubuntu 18.04配置IP地址和ubuntu 16.04是有差异的，这里也附上ubuntu 16.04配置IP地址的方法，供使用ubuntu 16.04容器参考。  

      vim /etc/network/interface
          # The loopback network interface
          auto lo
          iface lo inet loopback
          # The primary network interface
          auto eth0
          iface eth0 inet static
          address 10.10.8.101
          netmask 255.0.0.0
          gateway 192.168.8.1
      
      vim /etc/resolvconf/resolv.conf.d/base
          nameserver 8.8.8.8
          nameserver 8.8.4.4
      
      resolvconf -u
      
      /etc/init.d/networking restart

3.验证snap包  
&emsp;&emsp;ubuntu 18.04容器实例创建好了，我们把nginx的snap包拷贝到容器中安装验证。  

      lxc file push nginx_1.17.7_amd64.snap snapts/home/ubuntu/
&emsp;&emsp;我们把nginx的snap包拷贝到容器后，再进入容器安装，第一次安装nginx snap包时，需要下载snap core，可能会比较慢。  

      root@snapts1:/home/ubuntu# snap install nginx_1.17.7_amd64.snap --dangerous --devmode
      Download snap ""core" （8268） from  channel "stable"     
      nginx 1.17.7 installed  
&emsp;&emsp;nginx snap包安装成功，再执行一下nginx命令。  

      root@snapts1:/home/ubuntu# nginx
      root@snapts1:/home/ubuntu# ps -A | grep "nginx"
       1508 ?        00:00:00 nginx
       1509 ?        00:00:00 nginx
      root@snapts1:/home/ubuntu# cd /var/snap/nginx/common/logs/
      root@snapts1:/var/snap/nginx/common/logs# ls
      access.log  error.log
      root@snapts1:/var/snap/nginx/common/logs# cd ../run
      root@snapts1:/var/snap/nginx/common/run# ls
      nginx.pid
      
&emsp;&emsp;nginx命令运行成功，并正确生成了log文件和pid文件。  

4.后续工作  
&emsp;&emsp;lxc/lxd是一套完整的虚拟化技术，支持的命令有很多，这里不一一描述，请看参考资料。  
&emsp;&emsp;到这里snap devmode模式的打包验证工作基本完成了，后续我们将验证classic模式的snap包，请大家持续关注。  

5.参考资料:  
&emsp;&emsp;https://blog.51cto.com/johjoh/2153408?source=dra  

----
&emsp;&emsp;安微云是国内领先的基于Arm架构的云技术团队，提供虚拟化、数据分析、数据存储、文本处理、语义分析、自动化脚本等企业级云技术及服务。  
&emsp;&emsp;更多信息，请关注"安微云"公众号。