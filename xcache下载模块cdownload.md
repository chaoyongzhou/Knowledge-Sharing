# xcache下载模块cdownload

## 1、背景

xcache的下载模块cdownload支持断点续传，文件大小不限，支持http协议传输。

客户端下载工具以分片形式下载文件，分片文件名格式为：<目标文件名>.part_<起始偏移量>_<结束偏移量>_<原始文件长度>，分片文件的内容为原始文件[起始偏移量，结束偏移量)的范围，即左闭右开区间的内容。

比如，下载远端文件aa.log到本地的ab.log，分片范围为[100, 200)，原始文件长度为1000字节，那么分片文件名为ab.log.part_100_200_1000。

## 2、服务器端访问

### 2.1、按片下载文件

举例：

	curl -X GET -svo aa.log http://www.download.com/download/tmp/ab.log -x 127.1:80

下载服务器端文件/tmp/ab.log到本地文件aa.log。

### 2.2、按片下载文件

举例：

	curl -X GET -sv aa.log http://www.download.com/override/tmp/ab.log -x 127.1:80 -H "Content-Range: bytes 0-1/11"

下载服务器端文件/tmp/ab.log的[0, 1]区间，内容存储到本地文件aa.log。

### 2.3、备份文件

举例：

	curl -X GET -sv http://www.download.com/backup/tmp/ab.log -x 127.1:80

备份服务器端文件/tmp/ab.log到指定的备份目录下，这里备份目录由配置参数c\_download\_backup\_dir指定。

### 2.4、删除文件

举例：

	curl -X GET -sv http://www.download.com/delete/tmp/ab.log -x 127.1:80

删除服务器端文件/tmp/ab.log。

### 2.5、获取文件大小

举例：

	curl -X GET -sv http://www.download.com/size/tmp/ab.log -x 127.1:80

获取服务器端文件/tmp/ab.log的大小，单位：字节，由响应头X-File-Size带回。

### 2.6、获取目录下的一个文件

举例：

	curl -X GET -sv http://www.download.com/finger/tmp -x 127.1:80

获取服务器端目录/tmp下的一个文件，由响应头X-File带回；如果没有文件，则响应头X-File缺失。

### 2.7、按片计算MD5

举例：

	curl -X GET -sv http://www.download.com/md5/tmp/ab.log -x 127.1:80 -H "Content-Range: bytes 0-1/11"

获取服务器端文件/tmp/ab.log的[0, 1]区间内容的MD5，由响应头X-MD5带回MD5值得十六进制字符串。

### 2.8、按片比对

举例：

	curl -X GET -sv http://www.download.com/check/tmp/ab.log -x 127.1:80 -H "Content-Range: bytes 0-1/11" -H "Content-MD5: 0123456789abcdef"

比对文件大小和区间MD5值一致，返回200，否则返回401。

## 3、服务器端配置

举例：

	server {
	    listen  80;
	    server_name *.download.com;
	
	    set $c_acl_token   0123456789abcdef0123456789abcdef;
    	access_by_bgn cacltime;
	
	    location ~ /(download|check|delete|size|md5|ddir|finger|backup) {
	    	root /tmp/download;
	    	set $c_download_backup_dir /tmp/backup;
	        content_by_bgn cdownload;
	    }
	}

## 4、客户端工具用法

用法：

 	perl cdownload.pl [syncfiles=<num>] des=<local file> src=<remote file> ip=<server server ip[:port]> [host=<hostname>] [sleep=<seconds>] [timeout=<seconds>] [step=<nbytes>] [loglevel=<1..9>] [verbose=on|off]

下载文件举例：

 	perl cdownload.pl src=aa.log des=/tmp/ab.log ip=127.1:80 verbose=on

下载目录举例：

 	perl cdownload.pl sync=on src=/tmp des=/tmp/ ip=127.1:80 verbose=on

 ## 5、客户端工具源码

perl脚本文件：[cdownload.pl](https://github.com/chaoyongzhou/XCACHE/blob/master/bgn_ngx/tool/cdownload.pl "cdownload.pl")


