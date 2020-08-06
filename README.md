# eosio-haproxy

Configuration guide for HAProxy v2.3 including http and p2p load-balancing, access control and health checks

## Installing HAProxy

### 1. Dependencies (for ubuntu 18.04)
```bash
sudo apt install libssl-dev libsystemd-dev zlib1g-dev libpcre++-dev
```

### 2. Building Lua 5.4
```bash
curl -R -O http://www.lua.org/ftp/lua-5.4.0.tar.gz
tar zxf lua-5.4.0.tar.gz
cd lua-5.4.0
sudo make all install
cd ..
```

### 3. Getting the source code and building HAProxy
```bash
git clone https://github.com/haproxy/haproxy.git
cd haproxy
make -j $(nproc) TARGET=linux-glibc USE_OPENSSL=1 USE_ZLIB=1 USE_LUA=1 USE_PCRE=1 USE_SYSTEMD=1
sudo make install
cd ~
haproxy -v
```

### 4. Preparing the configuration file
```bash
sudo mkdir -p /etc/haproxy
sudo nano /etc/haproxy/haproxy.cfg
```

Example configuration:
```
global
	maxconn		50000
	# sends logs to stdout
	log		stdout format raw local0 info
	nbthread	2

# defaults must be set prior to sections using them
defaults
	log global
	mode http
	timeout connect 5s
        timeout client 10s
        timeout server 10s

# expose route with stats
listen stats
	bind		*:9090
	stats		enable
	stats		uri /monitor
	stats		refresh 5s

# http api listener
frontend http-in
	bind			*:18888
	option			httplog
	default_backend		nodeos-http-api
	# route access control
	acl	blacklist	path_beg -i /v1/producer
	acl	blacklist	path_beg -i /v1/net
	acl	blacklist	path_beg -i /v1/wallet
	acl	whitelist	path_beg -i /v1
	http-request		deny if blacklist OR !whitelist


# http backend with multiple nodeos
backend nodeos-http-api
	balance		leastconn
	option		httpchk GET /v1/chain/get_info HTTP/1.1
	http-check	send hdr Host your.domain.com
	default-server	check
	server		node1 127.0.0.1:28888
	server		node2 127.0.0.1:38888


# redefine defaults for tcp mode
defaults
	log		global
	mode		tcp
	timeout		connect 5s
	timeout		client	10s
	timeout		server	10s

# listener for incoming peers
frontend p2p-in
	bind *:19876
	option tcplog
	default_backend nodeos-p2p-nodes

# peering backend
backend nodeos-p2p-nodes
	balance		leastconn
	option          httpchk GET /v1/chain/get_info HTTP/1.1
	http-check      send hdr Host your.domain.com
	default-server	maxconn 64
	# specify peering servers with http ports used for health check
	server		peer1		127.0.0.1:29876 check port 28888
	server		peer2		127.0.0.1:39876 check port 38888

```

