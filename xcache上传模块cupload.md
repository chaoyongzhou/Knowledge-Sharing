# xcache上传模块cupload

## 1、背景

xcache的上传模块cupload支持断点续传，文件大小不限，支持http协议传输。

客户端上传工具以分片形式上传文件，分片文件名格式为：<目标文件名>.part_<起始偏移量>_<结束偏移量>_<原始文件长度>，分片文件的内容为原始文件[起始偏移量，结束偏移量)的范围，即左闭右开区间的内容。

比如，上传本地文件aa.log到服务器端的ab.log，分片范围为[100, 200)，原始文件长度为1000字节，那么分片文件名为ab.log.part_100_200_1000。

## 2、服务器端访问

### 2.1、按片上传文件

举例：

	curl -X POST -sv http://www.upload.com/upload/tmp/ab.log -x 127.1:80 -d "hello world" -H "Content-Range: bytes 0-10/11"

创建服务器端文件/tmp/ab.log，内容为“hello world”。

### 2.2、按片覆盖文件

举例：

	curl -X POST -sv http://www.upload.com/override/tmp/ab.log -x 127.1:80 -d "HE" -H "Content-Range: bytes 0-1/11"

修改服务器端文件/tmp/ab.log的[0, 1]区间的内容为“HE”。

### 2.3、清空文件

举例：

	curl -X GET -sv http://www.upload.com/empty/tmp/ab.log -x 127.1:80

置服务器端文件/tmp/ab.log为空文件。

### 2.4、按片合并

举例：

	curl -X GET -sv http://www.upload.com/merge/tmp/ab.log -x 127.1:80 -H "Content-Range: bytes 0-1/11"

将服务器端分片文件/tmp/ab.log.part_0_1_11的内容，追加合并到服务器端文件/tmp/ab.log，然后删除分片文件。

### 2.5、删除文件

举例：

	curl -X GET -sv http://www.upload.com/delete/tmp/ab.log -x 127.1:80

删除服务器端文件/tmp/ab.log。

### 2.6、获取文件大小

举例：

	curl -X GET -sv http://www.upload.com/size/tmp/ab.log -x 127.1:80

获取服务器端文件/tmp/ab.log的大小，单位：字节，由响应头X-File-Size带回。

### 2.7、按片计算MD5

举例：

	curl -X GET -sv http://www.upload.com/md5/tmp/ab.log -x 127.1:80 -H "Content-Range: bytes 0-1/11"

获取服务器端文件/tmp/ab.log的[0, 1]区间内容的MD5，由响应头X-MD5带回MD5值得十六进制字符串。

### 2.8、按片比对

举例：

	curl -X GET -sv http://www.upload.com/check/tmp/ab.log -x 127.1:80 -H "Content-Range: bytes 0-1/11" -H "Content-MD5: 0123456789abcdef"

比对文件大小和区间MD5值一致，返回200，否则返回401。

## 3、服务器端配置

举例：

	server {
	    listen  80;
	    server_name *.upload.com;
	
	    if ($uri = "/") {
	        rewrite (.*) /index.html;
	    }
	
	    location ~ /(upload|merge|delete|size|md5|empty|override|check) {
	    	root /tmp/upload;
	        content_by_bgn cupload;
	    }
	}

## 4、客户端工具用法

用法：

 	perl cupload.pl [sync=<on|off>] src=<local file> des=<remote file> ip=<server server ip[:port]> [host=<hostname>] [timeout=<seconds>] [step=<nbytes>] [loglevel=<1..9>] [verbose=on|off]

文件上传举例：

 	perl cupload.pl src=aa.log des=/tmp/ab.log ip=127.1:80 verbose=on

文件夹上传举例：

 	perl cupload.pl sync=on src=/tmp des=/tmp ip=127.1:80 verbose=on

 ## 5、客户端工具源码

perl脚本文件：[cupload.pl](https://github.com/chaoyongzhou/XCACHE/blob/master/bgn_ngx/tool/cupload.pl "cupload.pl")

