# 1、xfs环境准备

xfs支持裸盘访问，也支持块文件存储访问。

注意：由于存储空间要保留持久化元数据的空间，因此以块文件模拟磁盘时，建议文件大小不低于300G，用truncate命令创建即可。更小的块文件也可以通过调整xfs的编译参数、配置参数支持，不属于本文描述范围。

本文以Debian系统为例，6块NVMe盘/dev/nvme{0..5}n1，我们将1~5号盘作为数据盘，0号盘作为系统盘使用。


S1. 创建目录

	# mkdir -p /data/cache


S2. 建立软链：

	# ln -s /dev/nvme1n1 /data/cache/rnode1
	# ln -s /dev/nvme2n1 /data/cache/rnode2
	# ln -s /dev/nvme3n1 /data/cache/rnode3
	# ln -s /dev/nvme4n1 /data/cache/rnode4
	# ln -s /dev/nvme5n1 /data/cache/rnode5

结果如下：

	# ls -l /data/cache/
	total 0
	lrwxrwxrwx 1 root root 12 Mar  9 17:13 rnode1 -> /dev/nvme1n1
	lrwxrwxrwx 1 root root 12 Mar  9 17:13 rnode2 -> /dev/nvme2n1
	lrwxrwxrwx 1 root root 12 Mar  9 17:13 rnode3 -> /dev/nvme3n1
	lrwxrwxrwx 1 root root 12 Mar  9 17:13 rnode4 -> /dev/nvme4n1
	lrwxrwxrwx 1 root root 12 Mar  9 17:13 rnode5 -> /dev/nvme5n1

# 2、安装

## 2.1、安装命令

	# dpkg -i xfs-5.8.0.9_amd64.deb

安装目录和文件有：

	依赖包目录： /usr/local/xfs/depend
	可执行文件、脚本、配置目录：/usr/local/xfs/bin
	定时任务文件：/etc/cron.d/xfs
	服务脚本文件：/etc/init.d/xfs
	systemctl服务脚本模板：/etc/systemd/system/xfs.deb.service
	日志目录：/data/proclog/log/xfs

## 2.2、初始化

初始化脚本根据已创建的软链，自动在各盘上创建xfs存储系统、生成配置文件、生成systemctl服务脚本。

执行命令：

	# bash /usr/local/xfs/bin/xfs_init.sh | tee aa.log

配置文件：

	# ls -l /usr/local/xfs/bin/config.xml
	-rw-r--r-- 1 root root 49744 Mar 10 21:58 /usr/local/xfs/bin/config.xml

生成的systemctl服务脚本：

	# ls -l /etc/systemd/system/xfs*
	-rw-r--r-- 1 root root 355 Mar  9 17:18 /etc/systemd/system/xfs@rnode1.service
	-rw-r--r-- 1 root root 355 Mar  9 17:18 /etc/systemd/system/xfs@rnode2.service
	-rw-r--r-- 1 root root 355 Mar  9 17:18 /etc/systemd/system/xfs@rnode3.service
	-rw-r--r-- 1 root root 355 Mar  9 17:18 /etc/systemd/system/xfs@rnode4.service
	-rw-r--r-- 1 root root 355 Mar  9 17:18 /etc/systemd/system/xfs@rnode5.service

注：初始化自动生成的配置文件，仅适用于单机场景，不适用节点场景，后者需要采用修改配置或者下发配置的方式进行。

# 3、启动

xfs启停建议使用脚本/etc/init.d/xfs，用法如下：

	# /etc/init.d/xfs
	
	usage: /etc/init.d/xfs
	    start     [<1-12>|<tcid list>]      # start xfs servers by scan data cache dir /data/cache or start specific xfs server with rnode id
	    restart   [<1-12>|<tcid list>]      # restart xfs servers by scan data cache dir /data/cache or restart specific xfs server with rnode id
	    retrieve  [<1-12>|<tcid list>]      # retrieve xfs servers by scan data cache dir /data/cache or retrieve specific xfs server with rnode id
	    stop      [<1-12>|<tcid list>]      # stop xfs servers with xfs closing and shutdown or stop specific xfs server with rnode id
	    status    [<1-12>|<tcid list>]      # some or all xfs server(s) status
	    monitor   [<1-12>|<tcid list>]      # monitor some or all xfs server(s) status. notify ngx bgn if anyone down
	    rotate    [<1-12>|<tcid list>]      # rotate log on some or all xfs
	    recycle   [<1-12>|<tcid list>]      # recycle some or all xfs deleted space
	    garbage   [<1-12>|<tcid list>]      # retire expired locked files of some or all xfs
	    flush     [<1-12>|<tcid list>]      # flush some or all xfs to disk
	    retire    [<1-12>|<tcid list>]      # retire aging files of some or all xfs
	    breathe   [<1-12>|<tcid list>]      # memory breathing on some or all xfs
	    actsyscfg [<1-12>|<tcid list>]      # activate system configure on some or all xfs
	    status_np [<1-12>|<tcid list>]      # namenode status on some or all xfs
	    status_dn [<1-12>|<tcid list>]      # datanode status on some or all xfs
	e.g.
	    service xfs start
	    service xfs start 1
	    service xfs start 1,3,5

比如启动全部xfs：

	# /etc/init.d/xfs start

检查xfs状态：

	# /etc/init.d/xfs status
	[2020-03-24 14:38:04] XFS 10.20.67.18 ... running
	[2020-03-24 14:38:04] XFS 10.20.67.19 ... running
	[2020-03-24 14:38:04] XFS 10.20.67.20 ... running
	[2020-03-24 14:38:05] XFS 10.20.67.21 ... running
	[2020-03-24 14:38:05] XFS 10.20.67.22 ... running

或者检查进程：

	# ps -ef | grep xfs
	root      840789  731965  0 15:26 pts/8    00:00:00 grep xfs
	root     2716328       1  1 Mar20 ?        01:43:39 /usr/local/xfs/bin/xfs -tcid 10.20.67.21 -node_type xfs -xfs_sata_path /data/cache/rnode4 -logp /data/proclog/log/xfs -sconfig /usr/local/xfs/bin/config.xml -d
	root     2716365       1  1 Mar20 ?        01:27:51 /usr/local/xfs/bin/xfs -tcid 10.20.67.22 -node_type xfs -xfs_sata_path /data/cache/rnode5 -logp /data/proclog/log/xfs -sconfig /usr/local/xfs/bin/config.xml -d
	root     2744072       1  1 Mar20 ?        01:27:49 /usr/local/xfs/bin/xfs -tcid 10.20.67.18 -node_type xfs -xfs_sata_path /data/cache/rnode1 -logp /data/proclog/log/xfs -sconfig /usr/local/xfs/bin/config.xml -d
	root     2744430       1  1 Mar20 ?        01:29:54 /usr/local/xfs/bin/xfs -tcid 10.20.67.19 -node_type xfs -xfs_sata_path /data/cache/rnode2 -logp /data/proclog/log/xfs -sconfig /usr/local/xfs/bin/config.xml -d
	root     2745175       1  1 Mar20 ?        01:21:49 /usr/local/xfs/bin/xfs -tcid 10.20.67.20 -node_type xfs -xfs_sata_path /data/cache/rnode3 -logp /data/proclog/log/xfs -sconfig /usr/local/xfs/bin/config.xml -d

# 4、访问

xfs支持HTTP REST API访问和CONSOLE口访问

## 4.1、HTTP REST API访问

端口请参见/usr/local/xfs/bin/config.xml中的bgn项。

常用接口列表：


	| 接口 | 方法 | 功能 |
	--------------------
		
	| /xfs/getsmf/<path> [-H "store-offset: <offset>" -H "store-size: <size>"] | GET | 读文件 |
	
	| /xfs/dsmf/<path> | GET | 删文件 |
	| /xfs/ddir/<path> | GET | 删目录 |
	| /xfs/setsmf/<path> | POST | 写文件 |
	| /xfs/update/<path> | POST | 更新文件 |
	| /xfs/renew/<path> | POST | 更新文件（同upate） |
	
举例： 

	curl -v "http://127.1:718/xfs/getsmf/www.test.com/1K.dat/0"

更多接口请参考cxfshttp.c文件。其中服务脚本/etc/init.d/xfs中的部分功能利用HTTP REST API功能实现。

## 4.2、console访问

常见访问指令：
	
	        hsxfs read                      hsxfs <id> read file <name> on tcid <tcid> at <console|log>
	        hsxfs write                     hsxfs <id> write file <name> with content <string> on tcid <tcid> at <console|log>
	        hsxfs delete                    hsxfs <id> delete {file|dir|path} <name> on tcid <tcid> at <console|log>
	        hsxfs qfile                     hsxfs <id> qfile <file> on tcid <tcid> at <console|log>
	        hsxfs qdir                      hsxfs <id> qdir <dir> on tcid <tcid> at <console|log>
	        hsxfs qlist                     hsxfs <id> qlist <file or dir> {full | short | tree} [of np <np id>] on tcid <tcid> at <console|log>
	        hsxfs show                      hsxfs <id> show npp [<lru | del>] on tcid <tcid> at <console|log>
	        hsxfs show                      hsxfs <id> show dn on tcid <tcid> at <console|log>
	
比如:

	# ./console -tcid 0.0.0.64
	                                ------------------------------------------------------------
	                                |                                                          |
	                                |                WELCOME TO BGN CONSOLE UTILITY            |
	                                |                                                          |
	                               ------------------------------------------------------------
	bgn> hsxfs 0 show npp on tcid 10.20.67.18 at console
	[2020-03-24 15:17:44.448][tid 837727][co 0x555ac1a64d18] [rank_10.20.67.18_0][SUCC]
	[2020-03-24 15:17:44.448][tid 2744072][co 0x556ecc150670] cxfsnp mgr:
	[2020-03-24 15:17:44.448][tid 2744072][co 0x556ecc150670] cxfsnp model            : 9
	[2020-03-24 15:17:44.448][tid 2744072][co 0x556ecc150670] cxfsnp hash algo id     : 1
	[2020-03-24 15:17:44.448][tid 2744072][co 0x556ecc150670] cxfsnp item max num     : 33554430
	[2020-03-24 15:17:44.448][tid 2744072][co 0x556ecc150670] cxfsnp max num          : 1
	[2020-03-24 15:17:44.448][tid 2744072][co 0x556ecc150670] cxfsnp np size          : 4294967296
	[2020-03-24 15:17:44.448][tid 2744072][co 0x556ecc150670] cxfsnp np start offset  : 17039360
	[2020-03-24 15:17:44.448][tid 2744072][co 0x556ecc150670] cxfsnp np end   offset  : 4312006656
	[2020-03-24 15:17:44.448][tid 2744072][co 0x556ecc150670] cxfsnp np total size    : 4294967296
	[2020-03-24 15:17:44.448][tid 2744072][co 0x556ecc150670] cxfsnp 0x556ecacfee30: np 0, fsize 4294967296, del size 411249592, recycle size 53402811
	[2020-03-24 15:17:44.448][tid 2744072][co 0x556ecc150670] cxfsnp 0x556ecacfee30: header:
	[2020-03-24 15:17:44.448][tid 2744072][co 0x556ecc150670] np 0, model 9, hash algo 1, item max num 33554430, item used num 236172
	[2020-03-24 15:17:44.448][tid 2744072][co 0x556ecc150670] pool 7f4bc32000f0, node_max_num 33554430, node_used_num 236172, free_head 18737534, node_sizeof = 64

# 5、日志

日志目录： /data/proclog/log/xfs

日志文件: rank\_\<tcid\>\_<rank>.log 