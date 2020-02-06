<center>snap hooks实现安装过程中config文件的拷贝</center>

&emsp;&emsp;我们在《snap打包Nginx实践》中，熟悉了使用snapcraft工具打包nginx的过程，但nginx启动所需要的配置文件，是我们手动从只读区拷贝到可读写区的，下面我们就来了解一下snap hook技术，并实践通过snap hook技术来实现nginx snap包在安装过程中，自动把配置文件从只读区拷贝到可读写区。  

1.什么是snap hook  
&emsp;&emsp;snap hook通常是一个可执行文件，由某种操作触发，可以在snap的沙箱环境中运行。  
&emsp;&emsp;snap hook的常用场景有：  
&emsp;&emsp;1) 通知发生了什么事情  
&emsp;&emsp;比如snap app升级的时候，可能需要触发hook更新配置文件、升级数据格式等。  
&emsp;&emsp;2) 通知正在发生什么操作  
&emsp;&emsp;比如snap app可能需要知道特定的interface何时连接或断开。  

&emsp;&emsp;snap hook文件需要放在跟snapcraft.yaml文件同级目录的snap/hooks目录下，可执行文件的名称跟hook类型一致，比如我们即将用到的install hook的文件名称就是install，这样使用snapcraft工具打包后，snapd就会根据操作来触发snap hook。  
&emsp;&emsp;snap hook通常只能访问沙箱内部的资源，如果要访问沙箱外部资源，需要通过interface机制的slots and plugs来实现。  
&emsp;&emsp;snap hook按照触发操作分为configure hook、install hook、interface hook、pre-refresh hook、post-refresh hook、remove hook、prepare-device hook。  

2.编写hook实现nginx配置文件拷贝  
&emsp;&emsp;nginx的配置文件，是需要在install过程中进行拷贝的，所以我们需要编写install hook。  
&emsp;&emsp;编写之前，我们要了解一下snap沙箱环境的一些环境变量。以nginx为例，我们可以通过以下命令来查看nginx沙箱环境中的环境变量。  

      pp@pp-ThinkPad-S2-2nd-Gen:~$ snap run --shell nginx
      To run a command as administrator (user "root"), use "sudo <command>".
      See "man sudo_root" for details.

      pp@pp-ThinkPad-S2-2nd-Gen:/home/pp$ env
      ......
      SNAP_USER_COMMON=/home/pp/snap/nginx/common
      SHELL=/bin/bash
      TERM=vt100
      SNAP_CONTEXT=mgFGMhdv9hPbNnQyYNnTRKKmcHmWQSNq3nPtQjdwQKq3
      TMPDIR=/tmp
      SNAP_LIBRARY_PATH=/var/lib/snapd/lib/gl:/var/lib/snapd/lib/gl32:/var/lib/snapd/void
      SNAP_INSTANCE_NAME=nginx
      SNAP_COMMON=/var/snap/nginx/common
      SNAP_USER_DATA=/home/pp/snap/nginx/x1
      SNAP_DATA=/var/snap/nginx/x1
      PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
      TEMPDIR=/tmp
      PWD=/home/pp
      SNAP_REVISION=x1
      HOME=/home/pp/snap/nginx/x1
      SNAP_NAME=nginx
      SNAP_COOKIE=mgFGMhdv9hPbNnQyYNnTRKKmcHmWQSNq3nPtQjdwQKq3
      SNAP_ARCH=amd64
      SNAP_VERSION=1.17.7
      SNAP=/snap/nginx/x1
      XDG_RUNTIME_DIR=/run/user/1000/snap.nginx
      ......
&emsp;&emsp;这些环境变量，在沙箱环境中是可以直接用的，我们需要进行拷贝动作的源目录和目标目录分别是$SNAP和$SNAP-COMMON，$SNAP是只读区，$SNAP-COMMON是可读写区。我们在snapscaft.yaml同级的snap目录下添加并编辑install hook文件。  
  
      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ ls
      snap  snapcraft.yaml
      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ cd snap/hooks/
      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap/snap/hooks$ ls
      install
&emsp;&emsp;install文件的内容如下：  

      #!/bin/sh -e
      
      cp -R $SNAP/var/snap/nginx/common/conf $SNAP_COMMON/
      cp -R $SNAP/var/snap/nginx/common/logs $SNAP_COMMON/
      cp -R $SNAP/var/snap/nginx/common/run $SNAP_COMMON/
      mkdir -m 0777 $SNAP_COMMON/lib
&emsp;&emsp;在install文件中，我们把只读区中的文件拷贝到可读写区中，并创建lib目录。  
&emsp;&emsp;用snapcraft工具重新打包，并安装后，我们看一下目标目录。  

      pp@pp-ThinkPad-S2-2nd-Gen:/var/snap/nginx/common$ ls
      conf  lib  logs  run
&emsp;&emsp;nginx的配置文件和启动所需要的文件都复制成功，执行nginx命令也成功。

      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ sudo nginx
      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ ps -A | grep "nginx"
      18109 ?        00:00:00 nginx
      18110 ?        00:00:00 nginx

3.后续工作  
&emsp;&emsp;我们使用snap install hook实现了安装是自动把配置文件从只读区拷贝到可读写区，基本完成了snapcraft打包nginx的所有工作，但只是在现有的16.04环境中验证了snap包，这是不够的，需要在全新的ubuntu环境中进行验证。由于频繁安装ubuntu环境比较繁琐，后续我们将搭建lxc容器，在lxc的ubuntu实例中验证nginx snap包，请大家持续关注。   

4.参考资料:  
&emsp;&emsp;https://tutorials.ubuntu.com/tutorial/create-your-first-snap  
&emsp;&emsp;https://snapcraft.io/docs/t/pre-built-apps/6739  
&emsp;&emsp;https://snapcraft.io/docs/snapcraft-parts-metadata  
&emsp;&emsp;https://snapcraft.io/docs/snapcraft-app-and-service-metadata  
&emsp;&emsp;https://snapcraft.io/docs/supported-plugins  
&emsp;&emsp;https://snapcraft.io/docs/autotools-plugin  
&emsp;&emsp;https://snapcraft.io/docs/environment-variables  
&emsp;&emsp;https://snapcraft.io/docs/supported-snap-hooks  

----
&emsp;&emsp;安微云是国内领先的基于Arm架构的云技术团队，提供虚拟化、数据分析、数据存储、文本处理、语义分析、自动化脚本等企业级云技术及服务。  
&emsp;&emsp;更多信息，请关注"安微云"公众号。