<center>snap打包Nginx实践</center>

&emsp;&emsp;snap是Ubuntu母公司Canonical于2016年4月发布Ubuntu16.04时候引入的一种安全的、易于管理的、沙盒化的软件包格式。    
&emsp;&emsp;snap包基于squashFS文件系统，完全独立与操作系统，它包含了软件运行所需要的库和runtime，并且具有沙箱的属性，不能随意的访问外部资源，需要通过interfaces机制配置接口才能实现。  
&emsp;&emsp;snapcraft是构建和发布snap包的包管理系统。开发者可以编写配置文件snapcraft.yaml来定义snap包，然后通过snapcraft工具生成和发布snap包。snapcraft.yaml的详细定义，请参考https://snapcraft.io/docs/snapcraft-overview，由于涉及的定义很多，这里不一一描述。在打包Nginx的实践中，我们针对用到的字段做详细说明。  

&emsp;&emsp;下面我们开始在Ubuntu 16.04上实践打包Nginx。  

1.软件安装  
&emsp;&emsp;Ubuntu 16.04自带snap和snapcraft，可以用which命令确认一下。如果软件未安装，直接安装一下。  

      sudo apt install snap
      sudo apt install snapcraft  

&emsp;&emsp;snapcraft工具也可以直接安装snap包版本。  

      sudo snap install snapcraft --classic
&emsp;&emsp;Ubuntu 16.04自带的或用apt安装的snapcraft版本较旧，默认使用snap base是core，snap安装的snapcraft版本较新，默认使用的snap base是core18。实际实践中，我使用了Ubutu 16.04自带的snapcraft，用的snap base是core。  
&emsp;&emsp;snap base是snap包执行的基础环境，在安装snap包的时候，如果本地没有snap base，会自动下载。我们可以用snap命令查看本地是否存在。  

      pp@pp-ThinkPad-S2-2nd-Gen:~$ snap list
      Name       Version    Rev   Tracking  Publisher   Notes
      core       16-2.42.5  8268  stable    canonical✓  core
      core18     20200113   1650  stable    canonical✓  base
&emsp;&emsp;文件一般在/snap目录下。  

&emsp;&emsp;我们再确认一下snap的后台服务snapd是否已经启动。  

      pp@pp-ThinkPad-S2-2nd-Gen:~$ ps -A | grep "snapd"
       5072 ?        00:00:36 snapd
      17926 ?        00:00:00 snapd
&emsp;&emsp;如果snapd未启动，需要用命令启动。  

      service snapd start

2.打包nginx软件  
2.1 注册登录  
&emsp;&emsp;snapcraft工具完成打包后，可以直接把snap包发布到snap store上。为此，我们需要到https://snapcraft.io/account上申请Ubuntu One账号，并登录。  

      pp@pp-ThinkPad-S2-2nd-Gen:/snap$ snapcraft login
      Enter your Ubuntu One e-mail address and password.
      If you do not have an Ubuntu One account, you can create one at  https://snapcraft.io/account
      Email:

&emsp;&emsp;登录之后，就可以创建我们发布到snap store上的包的名字。  

      pp@pp-ThinkPad-S2-2nd-Gen:~/snap$ snapcraft register ****
      
      We always want to ensure that users get the software they expect 
      for a particular name.
      
      If needed, we will rename snaps to ensure that a particular name
      reflects the software most widely expected by our community.
      ......

&emsp;&emsp;注册好之后，我们可以登录https://snapcraft.io/snaps查看创建好的snap包名字。  

2.2 初始化snapcraft工程  
&emsp;&emsp;我们为nginx工程创建nginx-snap目录，并初始化，生成了snapcraffile:///C:/Users/zhangjijin.ROGEN/Desktop/教程/snap/snap hooks实现安装过程中config文件拷贝.mdt.yaml文件。  

      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap$ snapcraft init
      Created snap/snapcraft.yaml.
      Go to https://docs.snapcraft.io/the-snapcraft-format/8337 for more 
      information about the snapcraft.yaml format.

&emsp;&emsp;生成的snapcraft.yaml文件内容如下：  

      name: my-snap-name # you probably want to 'snapcraft register <name>'
      version: '0.1' # just for humans, typically '1.2+git' or '1.3.2'
      summary: Single-line elevator pitch for your amazing snap # 79 char long summary
      description: |
        This is my-snap's description. You have a paragraph or two to tell the
        most important story about your snap. Keep it under 100 words though,
        we live in tweetspace and your description wants to look good in the snap
        store.

      grade: devel # must be 'stable' to release into candidate/stable channels
      confinement: devmode # use 'strict' once you have the right plugs and slots

      parts:
        my-part:
          # See 'snapcraft plugins'
          plugin: nil

&emsp;&emsp;snapcraft.yaml文件由以下字段组成：  
&emsp;&emsp;name —— snap包的名字  
&emsp;&emsp;version —— snap包的版本号。可以是源代码git库中分支或者标签，也可以是当前日期或者自定义  
&emsp;&emsp;summary —— 不超过80个字符的摘要  
&emsp;&emsp;description —— 不超过100个字的snap包描述  
&emsp;&emsp;grade —— snap包的质量等级，stable或者devel。如果需要在snap store的stable channel中发布snap包，需要设置成stable  
&emsp;&emsp;confinement —— 标记snap运行时，与系统的隔离度。有3个等级：devmode、strict、classic，通常开发时，设置为devmode，snap包验证完成后，改为classic  
&emsp;&emsp;parts —— 描述snap包中代码如何获取、依赖关系和如何编译等  
&emsp;&emsp;另外一个重要字段是apps —— 描述snap包暴露出来给外部调用的命令或服务名称  

2.3 打nginx软件的snap包  
&emsp;&emsp;下面我们结合nginx的实际情况对snapcraft.yaml文件进行修改来完成打包。  
&emsp;&emsp;从nginx源码下载地址http://nginx.org/download/上我们看到最新的nginx源码版本是1.17.7，源码编译使用autotools，因此文件修改为：  

      name: nginx
      version: '1.17.7'
      summary: nginx server
      description: |
          nginx is a free, open-source, high-performance HTTP server and reverse proxy, as well as an IMAP/POP3 proxy server.
      
      grade: stable
      confinement: devmode

      apps:
        nginx:
          command: nginx
          plugs: [home]
      
      parts:
      nginx:
          source: http://nginx.org/download/nginx-1.17.7.tar.gz
          source-type: tar
          plugin: autotools
          configflags:
              - --conf-path=/var/snap/nginx/common/conf/nginx.conf
              - --error-log-path=/var/snap/nginx/common/logs/error.log
              - --http-log-path=/var/snap/nginx/common/logs/access.log
              - --pid-path=/var/snap/nginx/common/run/nginx.pid
              - --lock-path=/var/snap/nginx/common/lock/nginx.lock
              - --http-client-body-temp-path=/var/snap/nginx/common/lib/nginx_client_body
              - --http-proxy-temp-path=/var/snap/nginx/common/lib/nginx_proxy
              - --http-fastcgi-temp-path=/var/snap/nginx/common/lib/nginx_fastcgi
              - --http-uwsgi-temp-path=/var/snap/nginx/common/lib/nginx_uwsgi
              - --http-scgi-temp-path=/var/snap/nginx/common/lib/nginx_scgi
              - --with-http_ssl_module

&emsp;&emsp;configflags字段描述了执行configure命令的参数，具体参数不做具体描述了。这里重点说明一下/var/snap/nginx/common/这个目录。  
&emsp;&emsp;我们以nginx snap包为例，snap包安装完成后，它的文件系统被划分为只读和可读写的两种不同权限的区域，一般情况下只读区域为/snap/nginx目录，可读写区域就是/var/snap/nginx/common/，因为用户需要修改nginx的配置文件，nginx服务也会生成各种日志，因此我们就需要把编译参数中的目录都修改到这个可读写目录来，这个跟apt安装的nginx服务有较大区别。  

&emsp;&emsp;snapcraft.yaml文件修改后之后，我们就开始打snap包，但打包过程中，会遇到以下错误:  

      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ sudo snapcraft
      ......  
      ./configure: error: the HTTP rewrite module requires the PCRE library.
      ......
      ./configure: error: the HTTP gzip module requires the zlib library.
      ......
      ./configure: error: SSL modules require the OpenSSL library.
      ......

&emsp;&emsp;我们在parts中增加编译所依赖的包：  

      build-packages: [ libpcre3, libpcre3-dev, zlib1g, zlib1g-dev,  openssl, libssl-dev ]

&emsp;&emsp;依赖包的包名，我们可以通过apt命令查找到，例如： 

      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ sudo apt-cache search PCRE
      ......
      libpcre3 - Old Perl 5 Compatible Regular Expression Library - runtime files
      libpcre3-dbg - Old Perl 5 Compatible Regular Expression Library - debug symbols
      libpcre3-dev - Old Perl 5 Compatible Regular Expression Library - development files
      libpcre32-3 - Old Perl 5 Compatible Regular Expression Library - 32 bit runtime files
      ......

&emsp;&emsp;打包过程中，遇到问题修改后，重新打包时，需要clean掉重新编译。  

      sudo snapcraft clean
      sudo snapcraft

&emsp;&emsp;添加好依赖包之后，终于打包成功了。  

      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ sudo snapcraft
      ......  
      usr/lib/x86_64-linux-gnu/libcrypto.so.1.1
      usr/lib/x86_64-linux-gnu/libssl.so.1.1
      Snapping 'nginx' /                                                                                                             
      Snapped nginx_1.17.7_amd64.snap
      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ ls
      nginx_1.17.7_amd64.snap  parts  prime  snap  snapcraft.yaml  stage

3.安装验证  
&emsp;&emsp;nginx的snap打好了，我们来安装验证nginx snap包。  

      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ sudo snap install ./nginx_1.17.7_amd64.snap --dangerous --devmode
      nginx 1.17.7 installed
      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ which nginx
      /snap/bin/nginx

&emsp;&emsp;安装nginx snap包后，我们可以看到snap包的默认安装目录是/snap下。  
&emsp;&emsp;我们来验证一下nginx能否正常工作。  

      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ sudo nginx
      nginx: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.27' not found (required by nginx)
      nginx: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.25' not found (required by /snap/nginx/x1/usr/lib/x86_64-linux-gnu/libcrypto.so.1.1)
&emsp;&emsp;执行nginx命令，无法正常运行，缺少libc.so库，我们再修改snapcraft.yaml文件，通过stage-packages字段在parts中添加运行时的依赖库。  

      stage-packages: [ libc6, libc6-dev ]
&emsp;&emsp;重新执行snapcraft命令打包，并重新安装nginx snap包。  

      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ sudo snap remove nginx
      nginx removed
      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ which nginx
      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ sudo snap install ./nginx_1.17.7_amd64.snap --dangerous --devmode
      nginx 1.17.7 installed
      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ nginx
      nginx: [alert] could not open error log file: open() "/var/snap/nginx/common/logs/error.log" failed (2: No such file or directory)
      2020/01/28 17:39:10 [emerg] 17947#0: open() "/var/snap/nginx/common/conf/nginx.conf" failed (2: No such file or directory)
&emsp;&emsp;出现无法打开log文件和无法找到配置文件的错误，这是因为安装nginx snap包时，在snapcraft.yaml文件中定义的文件，并没有被复制到对应的可读写区目录中，还在只读区的安装目录中，我们来看一下。  

      pp@pp-ThinkPad-S2-2nd-Gen:/snap/nginx/current/var/snap/nginx/common$ ls
      conf  logs  run
      pp@pp-ThinkPad-S2-2nd-Gen:/snap/nginx/current/var/snap/nginx/common$ cd /var/snap/nginx/common
      pp@pp-ThinkPad-S2-2nd-Gen:/var/snap/nginx/common$ ls
      pp@pp-ThinkPad-S2-2nd-Gen:/var/snap/nginx/common$ 
&emsp;&emsp;我们把只读区的common目录中的内容拷贝到可读写区的对应目录中，并手动创建lib目录，再次执行nginx命令，成功。  

      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ sudo nginx
      pp@pp-ThinkPad-S2-2nd-Gen:~/nginx-snap/snap$ ps -A | grep "nginx"
      18109 ?        00:00:00 nginx
      18110 ?        00:00:00 nginx

4.后续工作  
&emsp;&emsp;以上我们熟悉了使用snapcraft工具打包snap包的过程，和snap包的初步验证过程，还有两部分的工作，我们会在后续的教程中详细讲述，请大家持续关注。  
&emsp;&emsp;1) 使用snap hooks在安装时自动把配置文件从只读区拷贝到可读写区  
&emsp;&emsp;2) 搭建lxc容器，在干净的ubuntu环境中验证snap包  

5.参考资料:  
&emsp;&emsp;https://tutorials.ubuntu.com/tutorial/create-your-first-snap  
&emsp;&emsp;https://snapcraft.io/docs/t/pre-built-apps/6739  
&emsp;&emsp;https://snapcraft.io/docs/snapcraft-parts-metadata  
&emsp;&emsp;https://snapcraft.io/docs/snapcraft-app-and-service-metadata  
&emsp;&emsp;https://snapcraft.io/docs/supported-plugins  
&emsp;&emsp;https://snapcraft.io/docs/autotools-plugin  

----
&emsp;&emsp;安微云是国内领先的基于Arm架构的云技术团队，提供虚拟化、数据分析、数据存储、文本处理、语义分析、自动化脚本等企业级云技术及服务。  
&emsp;&emsp;更多信息，请关注"安微云"公众号。