# xcache粘合ngx upstream

## 1、背景

按照xcache原本的设计，需要部署一个独立的detect组件做源站探测，xcache在回源前截住dns解析，向detect组件发送tcp解析请求，携带待解析的域名；detect组件根据配置的IP、域名、策略周期解析源站的存活，并根据服务过来的请求，返回一个IP地址。

考虑到对ngx upstream配置的直接支持，xcache决定采取粘合方式。

困难点在于proxy在ngx content阶段介入，向源站发起请求，而xcache需要在ngx content阶段介入，自己向源站发起请求，需要掐掉upstream建连部分。

因为，xcache需要解决的问题是，粘合upstream，只利用其配置、算法选择回源IP地址，而不走upstream的回源流程。

## 2、upstream原理分析

### 2.1、upstream模块

以tengine 2.3.2为例，与ngx http相关的upstream模块如下：

原生的顶层upstream模块：

	src/http/ngx_http_upstream.c
	src/http/ngx_http_upstream_round_robin.c

原生的upstream算法模块：

	src/http/modules/ngx_http_upstream_hash_module.c
	src/http/modules/ngx_http_upstream_ip_hash_module.c
	src/http/modules/ngx_http_upstream_keepalive_module.c
	src/http/modules/ngx_http_upstream_least_conn_module.c
	src/http/modules/ngx_http_upstream_random_module.c
	src/http/modules/ngx_http_upstream_zone_module.c

扩展upstream辅助功能模块：

	modules/ngx_http_upstream_check_module/ngx_http_upstream_check_module.c
	modules/ngx_http_upstream_consistent_hash_module/ngx_http_upstream_consistent_hash_module.c
	modules/ngx_http_upstream_dynamic_module/ngx_http_upstream_dynamic_module.c
	modules/ngx_http_upstream_keepalive_module/ngx_http_upstream_keepalive_module.c
	modules/ngx_http_upstream_session_sticky_module/ngx_http_upstream_session_sticky_module.c
	modules/ngx_http_upstream_vnswrr_module/ngx_http_upstream_vnswrr_module.c
	modules/ngx_multi_upstream_module/ngx_http_multi_upstream.c
	modules/ngx_multi_upstream_module/ngx_http_multi_upstream_module.c
	modules/ngx_multi_upstream_module/ngx_multi_upstream_module.c

### 2.2、源码分析

从配置和源码角度，来分析upstream原理。nginx是通过配置的不同，在运行期间动态结合handler来获得不同能力的。

就从指令proxy_pass入手。

#### 2.2.1、指令proxy_pass配置

upstream配置举例:

	upstream www.test.com {
	    server 192.168.1.146:80 max_fails=3;
	    server 192.168.1.147:80 max_fails=3;
	     
	    keepalive 1000;
	    keepalive_timeout 300s;
	    keepalive_requests 10000;
	}

server的location配置举例：

    location  / {
        set $upstream_name "www.test.com";
        proxy_pass http://$upstream_name;
    }

#### 2.2.2、指令proxy_pass入口源码

指令proxy_pass配置解析入口为ngx\_http\_proxy\_pass，指明了运行态content phase的handler为ngx\_http\_proxy\_handler。

	static char *
	ngx_http_proxy_pass(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
	{
	    ngx_http_proxy_loc_conf_t *plcf = conf;
	    ngx_http_core_loc_conf_t   *clcf;
	    
	    ......
	    clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
	    clcf->handler = ngx_http_proxy_handler;
	    ......
	}

在运行态，ngx\_http\_proxy\_handler做了三件事情：

> (1) 创建一个空的upstream （ngx\_http\_upstream\_t  *u），并挂到当前请求下（r->upstream = u），由ngx\_http\_upstream\_create完成
>  	
> (2) 设定upstream的create\_request handler （u->create_request = ngx\_http\_proxy\_create\_request）
>  	
> (3) 调用upstream的初始化（ngx\_http\_read\_client\_request\_body => ngx\_http\_upstream\_init => ngx\_http\_upstream\_init\_request）


	
	static ngx_int_t
	ngx_http_proxy_handler(ngx_http_request_t *r)
	{
	    ngx_http_upstream_t         *u;
	    ngx_http_proxy_loc_conf_t   *plcf;    
		
	    if (ngx_http_upstream_create(r) != NGX_OK) {
	        return NGX_HTTP_INTERNAL_SERVER_ERROR;
	    }
	    plcf = ngx_http_get_module_loc_conf(r, ngx_http_proxy_module);
	    u = r->upstream;
	    .....
	    
	    u->create_request = ngx_http_proxy_create_request;
	    ....
	    rc = ngx_http_read_client_request_body(r, ngx_http_upstream_init);
	    if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
	        return rc;
	    }
	    return NGX_DONE;
	}
		
	ngx_int_t
	ngx_http_upstream_create(ngx_http_request_t *r)
	{
	    ngx_http_upstream_t  *u;
	    u = r->upstream;
	    if (u && u->cleanup) {
	        r->main->count++;
	        ngx_http_upstream_cleanup(r);
	    }
	    u = ngx_pcalloc(r->pool, sizeof(ngx_http_upstream_t));
	    if (u == NULL) {
	        return NGX_ERROR;
	    }
	    r->upstream = u;
	    ......
	}

upstream的初始化也做了三件事情：

（1）根据proxy_pass指令中解析得到的host名称，从全局的upstream配置解析（umcf）中查找到对应的upstream配置（uscf），并挂入前面新创建的upstream中（u->upstream = uscf）

（2）调用查找到的uscf的peer初始化handler（uscf->peer.init(r, uscf)），这个handler的设定与扩展辅助模块有关，下面再表。

（3）调用connec（ngx\_http\_upstream\_connect => ngx\_event\_connect\_peer）开始发起到对端peer的连接（即源站或上一层），开始后续流程。

	static void
	ngx_http_upstream_init_request(ngx_http_request_t *r)
	{
	    ngx_str_t                      *host;
	    ngx_http_upstream_t            *u;
	    ngx_http_core_loc_conf_t       *clcf;
	    ngx_http_upstream_srv_conf_t   *uscf, **uscfp;
	    ngx_http_upstream_main_conf_t  *umcf;
	    if (r->aio) {
	        return;
	    }
	    u = r->upstream;
	    if (u->resolved == NULL) {
	        uscf = u->conf->upstream;
	    } else {
	        host = &u->resolved->host;
	        umcf = ngx_http_get_module_main_conf(r, ngx_http_upstream_module);
	        uscfp = umcf->upstreams.elts;
	        for (i = 0; i < umcf->upstreams.nelts; i++) {
	            uscf = uscfp[i];
	            if (uscf->host.len == host->len
	                && ((uscf->port == 0 && u->resolved->no_port)
	                     || uscf->port == u->resolved->port)
	                && ngx_strncasecmp(uscf->host.data, host->data, host->len) == 0)
	            {
	                goto found;
	            }
	        }
	   ......
	found:
	    if (uscf == NULL) {
	        ngx_log_error(NGX_LOG_ALERT, r->connection->log, 0,
	                      "no upstream configuration");
	        ngx_http_upstream_finalize_request(r, u,
	                                           NGX_HTTP_INTERNAL_SERVER_ERROR);
	        return;
	    }
	    u->upstream = uscf;
	    if (uscf->peer.init(r, uscf) != NGX_OK) {
	        ngx_http_upstream_finalize_request(r, u,
	                                           NGX_HTTP_INTERNAL_SERVER_ERROR);
	        return;
	    }
	    ......
	    ngx_http_upstream_connect(r, u);
	}
	
ngx\_http\_upstream\_connect发起socket连接后，向对端发起请求，这些正是xcache粘合upstream需要掐掉。

检视代码，可以看到一个重要的信息：对端信息就藏再u->peer.sockaddr中。所以，我们需要拿到peer即可。考虑到u是新创建的空的upstream （ngx\_http\_upstream\_t），所以， peer只可能、也应该来自于前面查找到的uscf。所以调用点uscf->peer.init(r, uscf)是关键。

	void
	ngx_http_upstream_connect(ngx_http_request_t *r, ngx_http_upstream_t *u)
	{
	    .......
	    rc = ngx_event_connect_peer(&u->peer);
	    ......
	    ngx_http_upstream_send_request(r, u, 1);
	}
	
	
uscf->peer.init这个handler的设定，取决于upstream配置块。

从2.2.1的配置来看，这是走的原生keepalive upstream模块。指令keepalive的解析入口为ngx\_http\_upstream\_keepalive，其设定uscf->peer.init_upstream的handler为ngx\_http\_upstream\_init\_keepalive。

	static char *
	ngx_http_upstream_keepalive(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
	{
	    ngx_http_upstream_srv_conf_t            *uscf;
	    ngx_http_upstream_keepalive_srv_conf_t  *kcf = conf;
	
	    /* init upstream handler */
	    uscf = ngx_http_conf_get_module_srv_conf(cf, ngx_http_upstream_module);
	    kcf->original_init_upstream = uscf->peer.init_upstream
	                                  ? uscf->peer.init_upstream
	                                  : ngx_http_upstream_init_round_robin;
	    uscf->peer.init_upstream = ngx_http_upstream_init_keepalive;
	    return NGX_CONF_OK;
	}



而uscf->peer.init_upstream是在创建nginx的主配置时调用的，即将upstream配置加入到nginx配置中。

	static ngx_http_module_t  ngx_http_upstream_module_ctx = {
	    ngx_http_upstream_add_variables,       /* preconfiguration */
	    NULL,                                  /* postconfiguration */
	    ngx_http_upstream_create_main_conf,    /* create main configuration */
	    ngx_http_upstream_init_main_conf,      /* init main configuration */
	    NULL,                                  /* create server configuration */
	    NULL,                                  /* merge server configuration */
	    NULL,                                  /* create location configuration */
	    NULL                                   /* merge location configuration */
	};
	static char *
	ngx_http_upstream_init_main_conf(ngx_conf_t *cf, void *conf)
	{
	    ngx_http_upstream_main_conf_t  *umcf = conf;
	    ngx_http_upstream_srv_conf_t  **uscfp;
	
	    uscfp = umcf->upstreams.elts;
	    for (i = 0; i < umcf->upstreams.nelts; i++) {
	        init = uscfp[i]->peer.init_upstream ? uscfp[i]->peer.init_upstream:
	                                            ngx_http_upstream_init_round_robin;
	        if (init(cf, uscfp[i]) != NGX_OK) {
	            return NGX_CONF_ERROR;
	        }
	    }
	    ......
	    return NGX_CONF_OK;
	}
	


再回头看具体到一个keepalive upstream配置的初始化入口ngx\_http\_upstream\_init\_keepalive，其设定uscf->peer.init的handler为ngx\_http\_upstream\_init\_keepalive\_peer。

	static ngx_int_t
	ngx_http_upstream_init_keepalive(ngx_conf_t *cf,
	    ngx_http_upstream_srv_conf_t *us)
	{
	    ngx_http_upstream_keepalive_srv_conf_t  *kcf;
	
	    kcf = ngx_http_conf_upstream_srv_conf(us,
	                                          ngx_http_upstream_keepalive_module);
	
	
	    if (kcf->original_init_upstream(cf, us) != NGX_OK) {
	        return NGX_ERROR;
	    }
	    ......
	    kcf->original_init_peer = us->peer.init;
	    us->peer.init = ngx_http_upstream_init_keepalive_peer;
	    ......
	}
	
开看看ngx\_http\_upstream\_init\_keepalive\_peer的实现，设定了一个重要的handler:  r->upstream->peer.get = ngx\_http\_upstream\_get\_keepalive\_peer;

	static ngx_int_t
	ngx_http_upstream_init_keepalive_peer(ngx_http_request_t *r,
	    ngx_http_upstream_srv_conf_t *us)
	{
	    ngx_http_upstream_keepalive_peer_data_t  *kp;
	    ngx_http_upstream_keepalive_srv_conf_t   *kcf;
	
	    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
	                   "init keepalive peer");
	    kcf = ngx_http_conf_upstream_srv_conf(us,
	                                          ngx_http_upstream_keepalive_module);
	    kp = ngx_palloc(r->pool, sizeof(ngx_http_upstream_keepalive_peer_data_t));
	    if (kp == NULL) {
	        return NGX_ERROR;
	    }
	    if (kcf->original_init_peer(r, us) != NGX_OK) {
	        return NGX_ERROR;
	    }
	    kp->conf = kcf;
	    kp->upstream = r->upstream;
	    kp->data = r->upstream->peer.data;
	    kp->original_get_peer = r->upstream->peer.get;
	    kp->original_free_peer = r->upstream->peer.free;
	    r->upstream->peer.data = kp;
	    r->upstream->peer.get = ngx_http_upstream_get_keepalive_peer;
	    r->upstream->peer.free = ngx_http_upstream_free_keepalive_peer;
	#if (NGX_HTTP_SSL)
	    kp->original_set_session = r->upstream->peer.set_session;
	    kp->original_save_session = r->upstream->peer.save_session;
	    r->upstream->peer.set_session = ngx_http_upstream_keepalive_set_session;
	    r->upstream->peer.save_session = ngx_http_upstream_keepalive_save_session;
	#endif
	    return NGX_OK;
	}

继续跟踪ngx\_http\_upstream\_get\_keepalive\_peer，调用kp->original\_get\_peer获得peer信息，找到connection并挂入peer connection下。

	static ngx_int_t
	ngx_http_upstream_get_keepalive_peer(ngx_peer_connection_t *pc, void *data)
	{
	    ngx_http_upstream_keepalive_peer_data_t  *kp = data;
	    ngx_http_upstream_keepalive_cache_t      *item;
	    ngx_int_t          rc;
	    ngx_queue_t       *q, *cache;
	    ngx_connection_t  *c;
	    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, pc->log, 0,
	                   "get keepalive peer");
	    /* ask balancer */
	    rc = kp->original_get_peer(pc, kp->data);
	    if (rc != NGX_OK) {
	        return rc;
	    }
	    /* search cache for suitable connection */
	    cache = &kp->conf->cache;
	    for (q = ngx_queue_head(cache);
	         q != ngx_queue_sentinel(cache);
	         q = ngx_queue_next(q))
	    {
	        item = ngx_queue_data(q, ngx_http_upstream_keepalive_cache_t, queue);
	        c = item->connection;
	        if (ngx_memn2cmp((u_char *) &item->sockaddr, (u_char *) pc->sockaddr,
	                         item->socklen, pc->socklen)
	            == 0)
	        {
	            ngx_queue_remove(q);
	            ngx_queue_insert_head(&kp->conf->free, q);
	            goto found;
	        }
	    }
	    return NGX_OK;
	found:
	    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, pc->log, 0,
	                   "get keepalive peer: using connection %p", c);
	    c->idle = 0;
	    c->sent = 0;
	    c->log = pc->log;
	    c->read->log = pc->log;
	    c->write->log = pc->log;
	    c->pool->log = pc->log;
	    if (c->read->timer_set) {
	        ngx_del_timer(c->read);
	    }
	    pc->connection = c;
	    pc->cached = 1;
	    return NGX_DONE;
	}

xcache粘合只关心如何获取peer，并不关心connection，因此这里的关注点为original\_get\_peer这个handler是如何被初始化的。

#### 2.2.3、获取peer

梳理代码，original\_get\_peer这个handler在如下接口中被初始化：

	ngx_http_dyups_init_peer
	ngx_http_multi_upstream_init_peer
	ngx_http_upstream_init_chash_peer
	ngx_http_upstream_init_dynamic_peer
	ngx_http_upstream_init_dynamic_peer
	ngx_http_upstream_init_hash_peer
	ngx_http_upstream_init_chash_peer
	ngx_http_upstream_init_ip_hash_peer
	ngx_http_upstream_init_keepalive_peer
	ngx_http_upstream_init_least_conn_peer
	ngx_http_upstream_init_random_peer
	ngx_http_upstream_init_round_robin_peer
	ngx_http_upstream_session_sticky_init_peer
	ngx_http_upstream_init_vnswrr_peer

逐一分析发现，所有初始化接口都直接或间接（通过handler original\_init\_peer）调用了接口ngx\_http\_upstream\_init\_round\_robin\_peer，这是归一化。

那么为什么会归一化到round robin呢？这还需要回头看ngx\_http\_upstream\_init\_keepalive接口中一段未解释的部分，即original\_init\_upstream的调用。

	static ngx_int_t
	ngx_http_upstream_init_keepalive(ngx_conf_t *cf,
	    ngx_http_upstream_srv_conf_t *us)
	{
	    ngx_http_upstream_keepalive_srv_conf_t  *kcf;
	
	    kcf = ngx_http_conf_upstream_srv_conf(us,
	                                          ngx_http_upstream_keepalive_module);
	
	
	    if (kcf->original_init_upstream(cf, us) != NGX_OK) {
	        return NGX_ERROR;
	    }
	    ......
	    kcf->original_init_peer = us->peer.init;
	    us->peer.init = ngx_http_upstream_init_keepalive_peer;
	    ......
	}

即，初始化当前keepalive upstream模块，需要先通过handler original\_init\_upstream初始化它的父模块，这儿体现的是继承性。再回溯指令keepalive的处理接口ngx\_http\_upstream\_keepalive，original\_init\_upstream要么被设定为uscf->peer.init\_upstream，要么被设定为ngx\_http\_upstream\_init\_round\_robin，取决于当前upstream是否存在父模块。每个upstream模块都有自己的初始化接口，根据模块的继承关系，先初始化符模块，再初始化当前模块。往上追溯继承关系的源头，根模块即使round robin。

	static char *
	ngx_http_upstream_keepalive(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
	{
	    ngx_http_upstream_srv_conf_t            *uscf;
	    ngx_http_upstream_keepalive_srv_conf_t  *kcf = conf;
	    ......
	    /* init upstream handler */
	    uscf = ngx_http_conf_get_module_srv_conf(cf, ngx_http_upstream_module);
	    kcf->original_init_upstream = uscf->peer.init_upstream
	                                  ? uscf->peer.init_upstream
	                                  : ngx_http_upstream_init_round_robin;
	    uscf->peer.init_upstream = ngx_http_upstream_init_keepalive;
	    return NGX_CONF_OK;
	}

检视ngx\_http\_upstream\_init\_round\_robin初始化的实现，其将upstream配置的服务器解析后的地址结果（请读码，一个服务器可能解析出多个地址）形成数组（peer数组，元素数据类型为ngx\_http\_upstream\_rr\_peer\_t），最终放在了upstream配置下面(us->peer.data = peers)。

	ngx_int_t
	ngx_http_upstream_init_round_robin(ngx_conf_t *cf,
	    ngx_http_upstream_srv_conf_t *us)
	{
	    ngx_http_upstream_rr_peer_t   *peer, **peerp;
	       ngx_http_upstream_rr_peers_t  *peers, *backup;
	    ......
	    us->peer.data = peers;
	    /* implicitly defined upstream has no backup servers */
	    return NGX_OK;
	}

那么，获取peer是直接通过ngx\_http\_upstream\_get\_round\_robin\_peer接口获取的吗？非也！

虽然upstream模块都继承于round robin，但是各自算法不同、策略不同，因为通常的办法是设定自己的peer数据结构，并继承ngx\_http\_upstream\_rr\_peer\_t结构，指向round robin模块的peer，这个upstream模块初始化的后半程完成（前半程为round robin模块初始化）。因此，获取peer需要在运行期间，根据算法策略获得相应的模块，并调用其peer get接口。

这里不可理喻之处在于，每个upstream单独维护了peer的down标志。比如，如果一个server同时配置在两个upstream中，运行期间其中一个upstream将其标down，另外一个upstream是不可感知的。

以consistent hash upstream模块为例，代码如下：

	typedef struct {
	    u_char                                  down;
	    uint32_t                                hash;
	    ngx_uint_t                              index;
	    ngx_uint_t                              rnindex;
	    ngx_http_upstream_rr_peer_t            *peer;
	} ngx_http_upstream_chash_server_t;
	
	static ngx_int_t
	ngx_http_upstream_init_chash(ngx_conf_t *cf, ngx_http_upstream_srv_conf_t *us)
	{
	    u_char                               hash_buf[256];
	    ngx_int_t                            j, weight;
	    ngx_uint_t                           sid, id, hash_len;
	    ngx_uint_t                           i, n, *number, rnindex;
	    ngx_http_upstream_rr_peer_t         *peer;
	    ngx_http_upstream_rr_peers_t        *peers;
	    ngx_http_upstream_chash_server_t    *server;
	    ngx_http_upstream_chash_srv_conf_t  *ucscf;
	
	    if (ngx_http_upstream_init_round_robin(cf, us) == NGX_ERROR) {
	        return NGX_ERROR;
	    }
	    ucscf = ngx_http_conf_upstream_srv_conf(us,
	                                     ngx_http_upstream_consistent_hash_module);
	    if (ucscf == NULL) {
	        return NGX_ERROR;
	    }
	    us->peer.init = ngx_http_upstream_init_chash_peer;
	    peers = (ngx_http_upstream_rr_peers_t *) us->peer.data;
	    if (peers == NULL) {
	        return NGX_ERROR;
	    }
	 
	    ......
	
	    ucscf->servers = ngx_pcalloc(cf->pool,
	                                 (ucscf->number + 1) *
	                                  sizeof(ngx_http_upstream_chash_server_t));
	    ......
	    ucscf->number = 0;
	    for (i = 0; i < n; i++) {
	               ......
	        for (j = 0; j < weight; j++) {
	            server = &ucscf->servers[++ucscf->number];
	            server->peer = peer;
	            server->rnindex = i;
	            id = sid * 256 * 16 + j;
	            server->hash = ngx_murmur_hash2((u_char *) (&id), 4);
	        }
	    }
	    ......
	    return NGX_OK;
	}

获取peer的接口为ngx\_http\_upstream\_get\_chash\_peer，最关键的是获得了peer的sockaddr信息。

	static ngx_int_t
	ngx_http_upstream_get_chash_peer(ngx_peer_connection_t *pc, void *data)
	{
	    ngx_http_upstream_rr_peer_t            *peer;
	    ngx_http_upstream_chash_server_t       *server;
	    ngx_http_upstream_chash_srv_conf_t     *ucscf;
	
	    ucscf = uchpd->ucscf;
	    ......
	
	    peer = server->peer;
	    pc->name = &peer->name;
	    pc->sockaddr = peer->sockaddr;
	    pc->socklen = peer->socklen;
	    return NGX_OK;
	}
	
这正是我们想要的。

### 2.3、原理总结

upstream系列模块根植于round robin模块，采用继承方式扩展能力边界。

初始化：首先通过round robin初始化完成前半程初始化，然后根据当前upstream的算法和策略，完成后半程初始化。后半程初始化继承round robin的peer数组，并重新组织。

获取peer：从当前upstream模块的peer get接口中获取。

建连：在content phase阶段，原生proxy模块介入，调用upstream模块的初始化（ngx\_http\_upstream\_init），后者发起建连。建连之前调用peer get获取peer的sockaddr信息。

## 3、xcache粘合upstream

粘合步骤如下：

（1）增加一个新的指令upstream_by_bgn，用来替换proxy_pass指令。

（2）根据配置指定的upstream名称，从全局的upstream列表（r->upstream）中，定位到upstream的配置uscf，对应的就是配置指向的upstream模块

（3）调用uscf模块的peer init接口，完成peer get接口的设定

（4）调用peer get接口，获取peer，从peer sockaddr中反向提取IP地址和端口号信息
