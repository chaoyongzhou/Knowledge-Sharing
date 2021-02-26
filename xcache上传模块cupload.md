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
	        content_by_bgn cupload;
	    }
	}

## 4、客户端工具用法

用法：

 	perl cupload.pl src=<local file> des=<remote file> ip=<server server ip[:port]> [host=<hostname>] [timeout=<seconds>] [step=<nbytes>] [loglevel=<1..9>] [verbose=on|off]

举例：

 	perl cupload.pl src=aa.log des=/tmp/ab.log ip=127.1:80 verbose=on

 ## 5、客户端工具源码

	 #! /usr/bin/perl -w
	
	########################################################################################################################
	# description:  upload file to server
	# version    :  v1.1
	# creator    :  chaoyong zhou
	#
	# History:
	#    1. 02/23/2021: v1.0, delivered
	#    2. 02/26/2021: v1.1, support merge file parts
	########################################################################################################################
	
	use strict;
	
	use LWP::UserAgent;
	use Digest::MD5 qw(md5 md5_hex);
	
	my $g_des_host;
	my $g_des_ip;
	my $g_src_fname;
	my $g_des_fname;
	my $g_timeout_nsec;
	my $g_step_nbytes;
	my $g_log_level     = 1; # default log level
	
	my $g_ua_agent = "Mozilla/8.0";
	
	my $g_autoflush_flag;
	my $g_usage =
	    "$0 src=<local file> des=<remote file> ip=<server server ip[:port]> [host=<hostname>] [timeout=<seconds>] [step=<nbytes>] [loglevel=<1..9>] [verbose=on|off]";
	my $verbose;
	
	my $paras_config = {};
	
	&fetch_config($paras_config, @ARGV);
	&check_config($paras_config, $g_usage);
	
	$verbose = $$paras_config{"verbose"}   if ( defined($$paras_config{"verbose"}) );
	if( defined($verbose) && $verbose =~/on/i )
	{
	    &print_config($paras_config);
	}
	
	$g_des_host     = $$paras_config{"host"}        || "www.upload.com";# default server domain
	$g_des_ip       = $$paras_config{"ip"};
	$g_src_fname    = $$paras_config{"src"};
	$g_des_fname    = $$paras_config{"des"};
	$g_timeout_nsec = $$paras_config{"timeout"}     || 30;              # default timeout in seconds
	$g_step_nbytes  = $$paras_config{"step"}        || 2 << 20;         # default segment size in bytes
	
	if(defined($$paras_config{"loglevel"}))
	{
	    $g_log_level = $$paras_config{"loglevel"}; # loglevel range [0..9]
	}
	
	&open_fp_autoflush(\*STDOUT);
	&main();
	&restore_fp_autoflush(\*STDOUT);
	
	################################################################################################################
	# $status = main()
	################################################################################################################
	sub main
	{
	    my $status;
	
	    # check src file name validity
	    if(defined($g_src_fname))
	    {
	        $status = &check_file_name_validity($g_src_fname);
	        if($status =~ /false/i)
	        {
	            &echo(0, sprintf("error:main: invalid src file name '%s'\n",
	                             $g_src_fname));
	            return 1; # fail
	        }
	    }
	
	    # check des file name validity
	    if(defined($g_des_fname))
	    {
	        if($g_des_fname !~ /^\//)
	        {
	            $g_des_fname = "/".$g_des_fname
	        }
	
	        $status = &check_file_name_validity($g_des_fname);
	        if($status =~ /false/i)
	        {
	            &echo(0, sprintf("error:main: invalid des file name '%s'\n",
	                             $g_des_fname));
	            return 1; # fail
	        }
	    }
	
	    $status = &upload_file();
	    if($status =~ /false/i)
	    {
	        &echo(0, sprintf("error:main: upload '%s' -> %s:%s:'%s' failed\n",
	                         $g_src_fname,
	                         $g_des_host, $g_des_ip, $g_des_fname));
	        return 1; # fail
	    }
	
	    &echo(1, sprintf("[DEBUG] main: upload '%s' -> %s:%s:'%s' succ\n",
	                     $g_src_fname,
	                     $g_des_host, $g_des_ip, $g_des_fname));
	
	    if( defined($verbose) && $verbose =~/on/i )
	    {
	        my $local_file_size;
	        my $local_file_md5;
	
	        my $remote_file_size;
	        my $remote_file_md5;
	
	        ($status, $local_file_size)  = &size_local_file_do();
	        &echo(0, sprintf("[DEBUG] main: local  file size %d\n", $local_file_size));
	
	        ($status, $remote_file_size) = &size_remote_file_do();
	        &echo(0, sprintf("[DEBUG] main: remote file size %d\n", $remote_file_size));
	
	        ($status, $local_file_md5)   = &md5_local_file_do(0, $local_file_size, $local_file_size);
	        &echo(0, sprintf("[DEBUG] main: local  file md5 %s\n", $local_file_md5));
	
	        ($status, $remote_file_md5)  = &md5_remote_file_do(0, $remote_file_size, $remote_file_size);
	        &echo(0, sprintf("[DEBUG] main: remote file md5 %s\n", $remote_file_md5));
	    }
	
	    return 0; # succ
	}
	
	################################################################################################################
	# $bool = check_file_name_validity($file_name)
	################################################################################################################
	sub check_file_name_validity
	{
	    my $file_name;
	    my $file_name_segs;
	    my $file_name_seg;
	
	    ($file_name) = @_;
	
	    $file_name_segs = [];
	    @$file_name_segs = split('/', $file_name);
	
	    foreach $file_name_seg (@$file_name_segs)
	    {
	        if($file_name_seg eq "..")
	        {
	            &echo(0, sprintf("error:check_file_name_validity: file name '%s' contains '..'\n",
	                             $file_name));
	            return "false";
	        }
	
	        # posix compatiblity
	        if(255 < length($file_name_seg))
	        {
	            &echo(0, sprintf("error:check_file_name_validity: file name '%s' seg len > 255\n",
	                             $file_name));
	            return "false";
	        }
	    }
	    return "true";
	}
	
	################################################################################################################
	# ($status, $file_size) = size_remote_file_do()
	################################################################################################################
	sub size_remote_file_do
	{
	    my $ua;
	    my $res;
	    my $k;
	
	    $ua = LWP::UserAgent->new;
	
	    $ua->agent($g_ua_agent); # pretend we are very capable browser
	    $ua->timeout($g_timeout_nsec);
	
	    $res = $ua->get(
	                "http://${g_des_ip}/size"."${g_des_fname}",
	                'Host' => $g_des_host);
	
	    &echo(9, sprintf("[DEBUG] size_remote_file_do: status : %d\n", $res->code));
	    &echo(9, sprintf("[DEBUG] size_remote_file_do: headers: %s\n", $res->headers_as_string));
	
	    $k = "X-File-Size";
	    &echo(8, sprintf("[DEBUG] size_remote_file_do: %s:%s\n", $k, $res->header($k) || "0"));
	
	    return ($res->code, $res->header($k) || "0");
	}
	
	################################################################################################################
	# ($status, $md5hex) = md5_remote_file_do($s_offset, $e_offset, $file_size)
	################################################################################################################
	sub md5_remote_file_do
	{
	    my $ua;
	    my $res;
	    my $k;
	
	    my $s_offset;
	    my $e_offset;
	    my $file_size;
	
	    # [s_offset, e_offset) /  file_size
	    ($s_offset, $e_offset, $file_size) = @_;
	
	    $ua = LWP::UserAgent->new;
	
	    $ua->agent($g_ua_agent);
	    $ua->timeout($g_timeout_nsec);
	
	    $e_offset = $e_offset - 1;
	
	    $res = $ua->get(
	                "http://${g_des_ip}/md5"."${g_des_fname}",
	                'Host'          => $g_des_host,
	                'Content-Range' => "bytes ${s_offset}-${e_offset}/${file_size}");
	
	    &echo(8, sprintf("[DEBUG] md5_remote_file_do: status : %d\n", $res->code));
	    &echo(9, sprintf("[DEBUG] md5_remote_file_do: headers: %s\n", $res->headers_as_string));
	
	    $k = "X-MD5";
	    &echo(8, sprintf("[DEBUG] md5_remote_file_do: %s:%s\n",$k, $res->header($k) || "-"));
	
	    return ($res->code, $res->header($k) || "-");
	}
	
	################################################################################################################
	# $status = del_remote_file_do()
	################################################################################################################
	sub del_remote_file_do
	{
	    my $ua;
	    my $res;
	
	    $ua = LWP::UserAgent->new;
	
	    $ua->agent($g_ua_agent); # pretend we are very capable browser
	    $ua->timeout($g_timeout_nsec);
	
	    $res = $ua->get(
	                "http://${g_des_ip}/delete"."${g_des_fname}",
	                'Host' => $g_des_host);
	
	    &echo(8, sprintf("[DEBUG] del_remote_file_do: status : %d\n", $res->code));
	
	    return $res->code
	}
	
	################################################################################################################
	# $status = upload_remote_file_do($s_offset, $e_offset, $file_size, $data)
	################################################################################################################
	sub upload_remote_file_do
	{
	    my $ua;
	    my $res;
	
	    my $s_offset;
	    my $e_offset;
	    my $file_size;
	    my $data;
	
	    ($s_offset, $e_offset, $file_size, $data) = @_;
	
	    $ua = LWP::UserAgent->new;
	
	    $ua->agent($g_ua_agent);
	    $ua->timeout($g_timeout_nsec);
	
	    $e_offset = $e_offset - 1;
	
	    $res = $ua->post(
	                "http://${g_des_ip}/upload"."${g_des_fname}",
	                'Content-Type'  => "text/html; charset=utf-8",
	                Content         =>$data,
	                'Host'          => $g_des_host,
	                'Content-Range' => "bytes ${s_offset}-${e_offset}/${file_size}");
	
	    &echo(8, sprintf("[DEBUG] upload_remote_file_do: status : %d\n", $res->code));
	    &echo(9, sprintf("[DEBUG] upload_remote_file_do: headers: %s\n", $res->headers_as_string));
	
	    &echo(2, sprintf("[DEBUG] upload_remote_file_do: %s, %d-%d/%d => %d\n",
	                     $g_des_fname,
	                     $s_offset, $e_offset, $file_size,
	                     $res->code));
	
	    return $res->code;
	}
	
	################################################################################################################
	# $status = merge_remote_file_do($s_offset, $e_offset, $file_size, $data)
	################################################################################################################
	sub merge_remote_file_do
	{
	    my $ua;
	    my $res;
	
	    my $s_offset;
	    my $e_offset;
	    my $file_size;
	    my $data;
	
	    ($s_offset, $e_offset, $file_size, $data) = @_;
	
	    $ua = LWP::UserAgent->new;
	
	    $ua->agent($g_ua_agent);
	    $ua->timeout($g_timeout_nsec);
	
	    $e_offset = $e_offset - 1;
	
	    $res = $ua->get(
	                "http://${g_des_ip}/merge"."${g_des_fname}",
	                'Content-Type'  => "text/html; charset=utf-8",
	                Content         =>$data,
	                'Host'          => $g_des_host,
	                'Content-Range' => "bytes ${s_offset}-${e_offset}/${file_size}");
	
	    &echo(8, sprintf("[DEBUG] merge_remote_file_do: status : %d\n", $res->code));
	    &echo(9, sprintf("[DEBUG] merge_remote_file_do: headers: %s\n", $res->headers_as_string));
	
	    &echo(2, sprintf("[DEBUG] merge_remote_file_do: %s, %d-%d/%d => %d\n",
	                     $g_des_fname,
	                     $s_offset, $e_offset, $file_size,
	                     $res->code));
	
	    return $res->code;
	}
	
	################################################################################################################
	# $status = override_remote_file_do($s_offset, $e_offset, $file_size, $data)
	################################################################################################################
	sub override_remote_file_do
	{
	    my $ua;
	    my $res;
	
	    my $s_offset;
	    my $e_offset;
	    my $file_size;
	    my $data;
	
	    ($s_offset, $e_offset, $file_size, $data) = @_;
	
	    $ua = LWP::UserAgent->new;
	
	    $ua->agent("Mozilla/8.0");
	    $ua->timeout($g_timeout_nsec);
	
	    $e_offset = $e_offset - 1;
	
	    $res = $ua->post(
	                "http://${g_des_ip}/override"."${g_des_fname}",
	                'Content-Type'  => "text/html; charset=utf-8",
	                Content         =>$data,
	                'Host'          => $g_des_host,
	                'Content-Range' => "bytes ${s_offset}-${e_offset}/${file_size}");
	
	    &echo(8, sprintf("[DEBUG] override_remote_file_do: status : %d\n", $res->code));
	    &echo(9, sprintf("[DEBUG] override_remote_file_do: headers: %s\n", $res->headers_as_string));
	
	    &echo(2, sprintf("[DEBUG] override_remote_file_do: %s, %d-%d/%d => %d\n",
	                     $g_des_fname,
	                     $s_offset, $e_offset, $file_size,
	                     $res->code));
	
	    return $res->code;
	}
	
	################################################################################################################
	# $status = empty_remote_file_do()
	################################################################################################################
	sub empty_remote_file_do
	{
	    my $ua;
	    my $res;
	
	    $ua = LWP::UserAgent->new;
	
	    $ua->agent($g_ua_agent); # pretend we are very capable browser
	    $ua->timeout($g_timeout_nsec);
	
	    $res = $ua->get(
	                "http://${g_des_ip}/empty"."${g_des_fname}",
	                'Host' => $g_des_host);
	
	    &echo(9, sprintf("[DEBUG] empty_remote_file_do: status : %d\n", $res->code));
	    &echo(9, sprintf("[DEBUG] empty_remote_file_do: headers: %s\n", $res->headers_as_string));
	
	    return $res->code;
	}
	
	################################################################################################################
	# ($status, $file_size) = size_local_file_do()
	################################################################################################################
	sub size_local_file_do
	{
	    if(-e $g_src_fname)
	    {
	        my @file_stats = stat ($g_src_fname);
	        return ("true", $file_stats[7]);
	    }
	
	    return ("false", -1);
	}
	
	################################################################################################################
	# ($bool, $md5hex) = md5_local_file_do($s_offset, $e_offset, $file_size)
	################################################################################################################
	sub md5_local_file_do
	{
	    my $ctx;
	    my $data;
	
	    my $s_offset;
	    my $e_offset;
	    my $file_size;
	    my $fp;
	
	    my $len;
	    my $ret;
	
	    # [s_offset, e_offset) /  file_size
	    ($s_offset, $e_offset, $file_size) = @_;
	
	    $ctx = Digest::MD5->new;
	
	    open ($fp, "< $g_src_fname") || die "cannot open file $g_src_fname";
	    binmode ($fp, ":bytes");
	
	    seek($fp, $s_offset, 0) || die "seek file $g_src_fname at offset $s_offset failed";
	
	    while($s_offset < $file_size && $s_offset < $e_offset)
	    {
	        $len = $g_step_nbytes;
	        if($s_offset + $len > $file_size)
	        {
	            $len = $file_size - $s_offset;
	        }
	        if($s_offset + $len > $e_offset)
	        {
	            $len = $e_offset - $s_offset;
	        }
	
	        $ret = read($fp, $data, $len);
	        if(! defined($ret))
	        {
	            die sprintf("error:md5_local_file_do: read %d-%d/%d failed",
	                        $s_offset, $s_offset + $len, $file_size);
	        }
	
	        last if(0 == $ret);
	
	        $ctx->add($data);
	
	        $s_offset = $s_offset + $len;
	    }
	
	    close($fp);
	
	    return ("true", $ctx->hexdigest);
	}
	
	################################################################################################################
	# ($bool, $s_offset) = finger_start_seg_do($local_file_size, $remote_file_size)
	################################################################################################################
	sub finger_start_seg_do
	{
	    my $status;
	
	    my $local_file_size;
	    my $remote_file_size;
	
	    my $local_file_md5;
	    my $remote_file_md5;
	
	    my $s_offset;
	    my $e_offset;
	
	    ($local_file_size, $remote_file_size) = @_;
	
	    $s_offset = 0;
	    while($s_offset < $local_file_size && $s_offset < $remote_file_size)
	    {
	        $e_offset = $s_offset + $g_step_nbytes;
	
	        if($e_offset > $local_file_size)
	        {
	            $e_offset = $local_file_size;
	        }
	
	        if($e_offset > $remote_file_size)
	        {
	            $e_offset = $remote_file_size;
	        }
	
	        ($status, $local_file_md5) = &md5_local_file_do($s_offset, $e_offset, $local_file_size);
	        if($status =~ /false/i)
	        {
	            return ("false", $s_offset);
	        }
	        &echo(2, sprintf("[DEBUG] finger_start_seg_do: md5 local file %s %d-%d/%d is %s\n",
	                        $g_src_fname, $s_offset, $e_offset, $local_file_size, $local_file_md5));
	
	
	        ($status, $remote_file_md5) = &md5_remote_file_do($s_offset, $e_offset, $remote_file_size);
	        if(200 != $status)
	        {
	            return ("false", $s_offset);
	        }
	
	        &echo(2, sprintf("[DEBUG] finger_start_seg_do: md5 remote file %s %d-%d/%d is %s\n",
	                        $g_des_fname, $s_offset, $e_offset, $remote_file_size, $remote_file_md5));
	
	        if($local_file_md5 ne $remote_file_md5)
	        {
	            &echo(2, sprintf("[DEBUG] finger_start_seg_do: %d-%d mismatched\n", $s_offset, $e_offset));
	            return ("true", $s_offset);
	        }
	
	        &echo(2, sprintf("[DEBUG] finger_start_seg_do: %d-%d matched\n", $s_offset, $e_offset));
	
	        $s_offset = $e_offset;
	    }
	
	    return ("true", $s_offset);
	}
	
	################################################################################################################
	# ($bool, $s_offset) = append_file($s_offset, $e_offset, $local_file_size)
	################################################################################################################
	sub append_file
	{
	    my $status;
	
	    my $s_offset;
	    my $e_offset;
	
	    my $s_offset_t;
	    my $e_offset_t;
	
	    my $local_file_size;
	
	    my $data;
	    my $fp;
	
	    ($s_offset, $e_offset, $local_file_size) = @_;
	
	    $s_offset_t = $s_offset;
	
	    open ($fp, "< $g_src_fname") || die "error:append_file: cannot open file $g_src_fname";
	    binmode ($fp, ":bytes");
	
	    seek($fp, $s_offset_t, 0) || die "error:append_file: seek file $g_src_fname at offset $s_offset_t failed";
	
	    while($s_offset_t < $e_offset && $s_offset_t < $local_file_size)
	    {
	        $e_offset_t = $s_offset_t + $g_step_nbytes;
	        if($e_offset_t > $e_offset)
	        {
	            $e_offset_t = $e_offset;
	        }
	        if($e_offset_t > $local_file_size)
	        {
	            $e_offset_t = $local_file_size;
	        }
	
	        if(read($fp, $data, $e_offset_t - $s_offset_t) <= 0)
	        {
	            close($fp);
	
	            &echo(0, sprintf("error:append_file: read local file %s %d-%d failed\n",
	                                     $g_src_fname, $s_offset_t, $e_offset_t));
	            return ("false", $s_offset_t);
	        }
	
	        $status = &upload_remote_file_do($s_offset_t, $e_offset_t, $local_file_size, $data);
	        if(200 != $status)
	        {
	            close($fp);
	
	            &echo(0, sprintf("error:append_file: append %s %d-%d/%d failed, status %d\n",
	                             $g_des_fname, $s_offset_t, $e_offset_t,
	                             $local_file_size, $status));
	
	            return ("false", $s_offset_t);
	        }
	        &echo(9, sprintf("[DEBUG] append_file: append %s %d-%d/%d done\n",
	                      $g_des_fname, $s_offset_t, $e_offset_t, $local_file_size));
	
	        $status = &merge_remote_file_do($s_offset_t, $e_offset_t, $local_file_size, $data);
	        if(200 != $status)
	        {
	            close($fp);
	
	            &echo(0, sprintf("error:append_file: merge %s %d-%d/%d failed, status %d\n",
	                                     $g_des_fname, $s_offset_t, $e_offset_t,
	                                     $local_file_size, $status));
	            return ("false", $s_offset_t);
	        }
	        &echo(9, sprintf("[DEBUG] append_file: merge %s %d-%d/%d done\n",
	                      $g_des_fname, $s_offset_t, $e_offset_t, $local_file_size));
	
	        &echo(1, sprintf("[DEBUG] append_file: append %s, %d-%d/%d => complete %.2f%%\n",
	                         $g_des_fname,
	                         $s_offset_t, $e_offset_t, $local_file_size,
	                         100.0 * ($e_offset_t + 0.0) / ($local_file_size + 0.0)));
	
	        $s_offset_t = $e_offset_t;
	    }
	
	    close($fp);
	
	    return ("true", $s_offset_t);
	}
	
	################################################################################################################
	# ($bool, $s_offset) = override_file($s_offset, $e_offset, $local_file_size, $remote_file_size)
	################################################################################################################
	sub override_file
	{
	    my $status;
	
	    my $local_file_size;
	    my $remote_file_size;
	
	    my $s_offset;
	    my $e_offset;
	
	    my $s_offset_t;
	    my $e_offset_t;
	
	    my $data;
	    my $fp;
	
	    ($s_offset, $e_offset, $local_file_size, $remote_file_size) = @_;
	
	    $s_offset_t = $s_offset;
	
	    open ($fp, "< $g_src_fname") || die "error:override_file: cannot open file $g_src_fname";
	    binmode ($fp, ":bytes");
	
	    seek($fp, $s_offset_t, 0) || die "error:override_file: seek file $g_src_fname at offset $s_offset_t failed";
	
	    while($s_offset_t < $e_offset
	    && $s_offset_t < $local_file_size
	    && $s_offset_t < $remote_file_size)
	    {
	        $e_offset_t = $s_offset_t + $g_step_nbytes;
	        if($e_offset_t > $e_offset)
	        {
	            $e_offset_t = $e_offset;
	        }
	        if($e_offset_t > $local_file_size)
	        {
	            $e_offset_t = $local_file_size;
	        }
	        if($e_offset_t > $remote_file_size)
	        {
	            $e_offset_t = $remote_file_size;
	        }
	
	        if(read($fp, $data, $e_offset_t - $s_offset_t) <= 0)
	        {
	            close($fp);
	            &echo(0, sprintf("error:override_file: read local file %s %d-%d failed\n",
	                             $g_src_fname, $s_offset_t, $e_offset_t));
	
	            return ("false", $s_offset_t);
	        }
	
	        $status = &override_remote_file_do($s_offset_t, $e_offset_t, $local_file_size, $data);
	        if(200 != $status)
	        {
	            close($fp);
	            &echo(0, sprintf("error:override_file: override %s %d-%d/%d failed, status %d\n",
	                             $g_des_fname, $s_offset_t, $e_offset_t,
	                             $local_file_size, $status));
	
	            return ("false", $s_offset_t);
	        }
	        &echo(9, sprintf("[DEBUG] override_file: override %s %d-%d/%d done\n",
	                      $g_des_fname, $s_offset_t, $e_offset_t, $local_file_size));
	
	        &echo(1, sprintf("[DEBUG] override_file: override %s, %d-%d/%d => complete %.2f%%\n",
	                         $g_des_fname,
	                         $s_offset_t, $e_offset_t, $local_file_size,
	                         100.0 * ($e_offset_t + 0.0) / ($local_file_size + 0.0)));
	
	        $s_offset_t = $e_offset_t;
	    }
	
	    close($fp);
	
	    return ("true", $s_offset_t);
	}
	
	################################################################################################################
	# $bool = upload_file($status, $local_file_size)
	################################################################################################################
	sub upload_file
	{
	    my $status;
	    my $code;
	
	    my $local_file_size;
	    my $local_file_md5;
	
	    my $remote_file_size;
	    my $remote_file_md5;
	
	    my $data;
	    my $s_offset;
	    my $e_offset;
	
	    my $fp;
	
	    ($status, $local_file_size) = &size_local_file_do();
	    if($status =~ /false/i)
	    {
	        &echo(0, sprintf("error:upload_file: not found %s\n", $g_src_fname));
	        return "false";
	    }
	
	    ($code, $remote_file_size) = &size_remote_file_do();
	    &echo(9, sprintf("[DEBUG] upload_file: [remote] code: %d, size: %d\n", $code, $remote_file_size));
	
	    # remote file exist
	    if(200 == $code)
	    {
	        &echo(9, sprintf("[DEBUG] upload_file: local file size: %d, remote file size: %d\n",
	                         $local_file_size, $remote_file_size));
	
	        if(0 == $local_file_size)
	        {
	            if(0 < $remote_file_size)
	            {
	                $status = &empty_remote_file_do();
	                if(200 != $status)
	                {
	                    &echo(0, sprintf("error:upload_file: empty remote file %s failed, status %d\n",
	                                     $g_des_fname, $status));
	                    return "false";
	                }
	                &echo(1, sprintf("[DEBUG] upload_file: empty remote file %s done\n",
	                                 $g_des_fname));
	            }
	            &echo(1, sprintf("[DEBUG] upload_file: empty file => succ\n"));
	            return "true";
	        }
	
	        if($local_file_size < $remote_file_size)
	        {
	            $status = &del_remote_file_do();
	            if(200 != $status)
	            {
	                &echo(0, sprintf("error:upload_file: del remote file %s failed, status %d\n",
	                                 $g_des_fname, $status));
	                return "false";
	            }
	
	            $s_offset = 0;
	
	            &echo(1, sprintf("[DEBUG] upload_file: local file size %d < remote file size %ld, del remote file %s done\n",
	                              $g_des_fname, $local_file_size, $remote_file_size));
	        }
	        elsif(0 == $remote_file_size)
	        {
	            $s_offset = 0;
	        }
	        else # $local_file_size > $remote_file_size && $remote_file_size > 0
	        {
	            ($status, $local_file_md5) = &md5_local_file_do(0, $remote_file_size, $local_file_size);
	            if($status =~ /false/i)
	            {
	                &echo(0, sprintf("error:upload_file: md5 local file %s, %d-%d/%d failed\n",
	                                 $g_src_fname, 0, $remote_file_size, $local_file_size));
	                return "false";
	            }
	            &echo(1, sprintf("[DEBUG] upload_file: md5 %d-%d/%d => %s, local  file %s\n",
	                             0, $remote_file_size, $local_file_size,
	                             $local_file_md5,
	                             $g_src_fname));
	
	            ($status, $remote_file_md5) = &md5_remote_file_do(0, $remote_file_size, $remote_file_size);
	            if(200 != $status)
	            {
	                &echo(0, sprintf("error:upload_file: md5 remote file %s failed\n", $g_des_fname));
	                return "false";
	            }
	            &echo(1, sprintf("[DEBUG] upload_file: md5 %d-%d/%d => %s, remote file %s\n",
	                             0, $remote_file_size, $remote_file_size,
	                             $remote_file_md5,
	                             $g_des_fname));
	
	            if($local_file_md5 eq $remote_file_md5)
	            {
	                if($local_file_size == $remote_file_size)
	                {
	                    &echo(2, sprintf("[DEBUG] upload_file: same file => succ\n"));
	                    return "true";
	                }
	
	                $s_offset = 0;
	                $e_offset = $remote_file_size;
	
	                &echo(1, sprintf("[DEBUG] upload_file: skip %s, %d-%d/%d => complete %.2f%%\n",
	                                 $g_des_fname,
	                                 $s_offset, $e_offset, $local_file_size,
	                                 100.0 * ($e_offset + 0.0) / ($local_file_size + 0.0)));
	
	                $s_offset = $remote_file_size;
	            }
	            else
	            {
	                ($status, $s_offset) = &finger_start_seg_do($local_file_size, $remote_file_size);
	                if($status =~ /false/i)
	                {
	                    &echo(0, sprintf("error:upload_file: file %s, finger start seg failed, s_offset = %d\n",
	                                    $g_src_fname, $s_offset));
	                    return "false";
	                }
	                &echo(1, sprintf("[DEBUG] upload_file: file %s, finger start seg done, s_offset = %d\n",
	                                 $g_src_fname, $s_offset));
	
	                ($status, $s_offset) = &override_file($s_offset, $remote_file_size, $local_file_size, $remote_file_size);
	                if($status =~ /false/i)
	                {
	                    &echo(0, sprintf("error:upload_file: file %s, override failed, s_offset = %d\n",
	                                     $g_src_fname, $s_offset));
	                    return ("false", );
	                }
	                &echo(1, sprintf("[DEBUG] upload_file: file %s, override done, s_offset = %d\n",
	                                 $g_src_fname, $s_offset));
	            }
	        }
	    }
	    # remote file not exist
	    else
	    {
	        $s_offset = 0;
	    }
	
	    # optimize: alignment
	    if(0 != ($s_offset % $g_step_nbytes))
	    {
	        $e_offset = $s_offset + $g_step_nbytes - ($s_offset % $g_step_nbytes);
	        if($e_offset > $local_file_size)
	        {
	            $e_offset = $local_file_size;
	        }
	
	        ($status, $s_offset) = &append_file($s_offset, $e_offset, $local_file_size);
	        if($status =~ /false/i)
	        {
	            &echo(0, sprintf("error:upload_file: file %s, append seg failed, s_offset = %d\n",
	                             $g_src_fname, $s_offset));
	            return "false";
	        }
	    }
	
	    ($status, $s_offset) = &append_file($s_offset, $local_file_size, $local_file_size);
	    if($status =~ /false/i)
	    {
	        &echo(0, sprintf("error:upload_file: file %s, append failed, s_offset = %d\n",
	                         $g_src_fname, $s_offset));
	        return "false";
	    }
	
	    &echo(1, sprintf("[DEBUG] upload_file: upload %s done\n", $g_des_fname));
	    return "true";
	}
	
	################################################################################################################
	# echo($loglevel, $msg)
	################################################################################################################
	sub echo
	{
	    my $log_level;
	    my $msg;
	
	    my $date;
	
	    ($log_level, $msg) = @_;
	    if($log_level <= $g_log_level)
	    {
	        chomp($date = `date '+%m/%d/20%y %H:%M:%S'`);
	
	        printf STDOUT ("[%s] %s", $date, $msg) if defined($msg);
	    }
	}
	
	################################################################################################################
	# check_config(%config, $usage)
	################################################################################################################
	sub check_config
	{
	    my $config;
	    my $usage;
	
	    my $str;
	    my @keys;
	    my $key;
	
	    my $invalid_flag;
	
	    ($config, $usage) = @_;
	
	    $str = $usage;
	    $str =~ s/=<.*?>//g;
	
	    @keys = split(/\s+/, $str);
	    shift(@keys);
	
	    $invalid_flag = 0;
	    foreach $key (@keys)
	    {
	        next if(  $key =~ /^\[.*\]$/ );
	
	        if( ! defined( $$config{ $key } ) )
	        {
	            &echo(0, "error: absent parameter of $key\n");
	            $invalid_flag = 1;
	        }
	    }
	
	    &echo(0, "absent parameter(s)\nusage = $usage\n") if ( 0 ne $invalid_flag  );
	}
	
	################################################################################################################
	# print_config(%config)
	################################################################################################################
	sub print_config
	{
	    my $config;
	
	    my $key;
	    my $value;
	
	    ($config) = @_;
	
	    while ( ($key, $value) = each (%$config) )
	    {
	        &echo(0, sprintf("%-16s: %s\n", $key, $value));
	    }
	}
	
	################################################################################################################
	# fetch_config(%config, @argv)
	################################################################################################################
	sub fetch_config
	{
	    my $config;
	    my @argv;
	
	    my $arg_num;
	    my $arg_idx;
	
	    ($config, @argv) = @_;
	
	    $arg_num = scalar(@argv);
	    for( $arg_idx = 0; $arg_idx < $arg_num; $arg_idx ++ )
	    {
	        if( $argv[ $arg_idx ] =~ /(.*?)=(.*)/ )
	        {
	            $$config{ $1 }  = $2;
	            next;
	        }
	    }
	}
	
	################################################################################################################
	# finger_config(%config, $k, $default_v)
	################################################################################################################
	sub finger_config
	{
	    my $config;
	    my $k;
	    my $v;
	
	    my $arg_num;
	    my $arg_idx;
	
	    ($config, $k, $v) = @_;
	
	    if(defined($$config{ $k }))
	    {
	        return $$config{ $k };
	    }
	
	    return $v;
	}
	
	
	################################################################################################################
	# open_fp_autoflush(FILEHANDLE)
	################################################################################################################
	sub open_fp_autoflush
	{
	    my $fp;
	
	    ($fp) = @_;
	
	    $g_autoflush_flag = $|;
	    $|                = 1;
	    select($fp);
	}
	
	################################################################################################################
	# restore_fp_autoflush(FILEHANDLE)
	################################################################################################################
	sub restore_fp_autoflush
	{
	    my $fp;
	
	    ($fp) = @_;
	
	    $| = $g_autoflush_flag;
	    select($fp);
	}

