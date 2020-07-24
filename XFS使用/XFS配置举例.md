8个ngx worker和12个xfs的配置举例如下：

	<sysConfig>
	  <taskConfig>
	    <tasks tcid="10.10.67.18-29"  maski="0" maske="0" ipv4="127.0.0.1" bgn="618-629" rest="718-729" cluster="1,3"/>
	    <tasks tcid="10.10.6.18-29"  maski="0" maske="0" ipv4="127.0.0.1" bgn="818-829" rest="918-929" cluster="1,3"/>
	    <tasks tcid="10.10.7.18-29"  maski="0" maske="0" ipv4="127.0.0.1" bgn="838-849" rest="938-949" cluster="1,3"/>
	    <tasks tcid="10.10.8.18-29"  maski="0" maske="0" ipv4="127.0.0.1" bgn="858-869" rest="958-969" cluster="1,3"/>
	    <tasks tcid="10.10.9.18-29"  maski="0" maske="0" ipv4="127.0.0.1" bgn="878-889" rest="978-989" cluster="1,3"/>
	    <tasks tcid="0.0.0.64"  maski="32" maske="32" ipv4="127.0.0.1" bgn="600" cluster="1,2"/>
	    <tasks tcid="0.0.0.65"  maski="32" maske="32" ipv4="127.0.0.1" bgn="700" cluster="2"/>
	  </taskConfig>
	  <clusters>
	    <cluster id="1" name="debug64" model="master_slave">
	      <node role="master"  tcid="0.0.0.64"   rank="0"/>
	      <node role="slave"   tcid="10.10.67.18-29" rank="0"/>
	      <node role="slave"   tcid="10.10.6.18-29" rank="0"/>
	      <node role="slave"   tcid="10.10.7.18-29" rank="0"/>
	      <node role="slave"   tcid="10.10.8.18-29" rank="0"/>
	      <node role="slave"   tcid="10.10.9.18-29" rank="0"/>
	    </cluster>
	    <cluster id="2" name="debug65" model="master_slave">
	      <node role="master"  tcid="0.0.0.65"   rank="0"/>
	      <node role="slave"  tcid="0.0.0.64"   rank="0"/>
	    </cluster>
	    <cluster id="3" name="xfs-ngx" model="master_slave">
	      <node role="master"   tcid="10.10.67.18-29" rank="0"/>
	      <node role="slave"   tcid="10.10.6.18-29" rank="0"/>
	      <node role="slave"   tcid="10.10.7.18-29" rank="0"/>
	      <node role="slave"   tcid="10.10.8.18-29" rank="0"/>
	      <node role="slave"   tcid="10.10.9.18-29" rank="0"/>
	    </cluster> 
	  </clusters>
	  <parasConfig>
	    <paraConfig tcid="10.10.67.18-29" rank="0">
	      <threadConfig maxReqThreadNum="1024"/>
	      <xfsConfig xfsDnAmdSwitch="on" xfsDnAmdMemDiskSize="8589934592"/>
	      <logConfig logLevel="all:0"/>
	    </paraConfig>
	    
	    <paraConfig tcid="10.10.6.18-29" rank="0">
	      <threadConfig maxReqThreadNum="4096"/>
	      <logConfig logLevel="all:0"/>
	    </paraConfig>
	    
	    <paraConfig tcid="10.10.7.18-29" rank="0">
	      <threadConfig maxReqThreadNum="4096"/>
	      <logConfig logLevel="all:0"/>
	    </paraConfig>
	    
	    <paraConfig tcid="10.10.8.18-29" rank="0">
	      <threadConfig maxReqThreadNum="4096"/>
	      <logConfig logLevel="all:0"/>
	    </paraConfig>
	   
	    <paraConfig tcid="10.10.9.18-29" rank="0">
	      <threadConfig maxReqThreadNum="4096"/>
	      <logConfig logLevel="all:0"/>
	    </paraConfig>
	   
	    <paraConfig tcid="0.0.0.64" rank="0">
	      <threadConfig maxReqThreadNum="4" taskSlowDownMsec="3" taskNotSlowDownMaxTimes="1"/>
	      <logConfig logLevel="all:0"/>
	    </paraConfig>
	    <paraConfig tcid="0.0.0.65" rank="0">
	      <threadConfig maxReqThreadNum="4" taskSlowDownMsec="3" taskNotSlowDownMaxTimes="1"/>
	      <logConfig logLevel="all:0"/>
	    </paraConfig>
	  </parasConfig>
	</sysConfig>
