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
	# global connection limit
	maxconn		50000
	# sends logs to stdout
	log		stdout format raw local0 info
	# number of threads
	nbthread	2

# ------- HTTP Section ---------

# defaults must be set prior to sections using them (can be redefined later)
defaults
	log global
	mode http
	timeout connect 2s
        timeout client 5s
        timeout server 5s

# expose route with stats (optional, but recommended)
listen stats
	bind		127.0.0.1:9090
	stats		enable
	stats		uri /monitor
	stats		refresh 5s

# http api listener (endpoint where external users will connect)
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


# http backend with multiple nodeos (chain_api_plugin)
backend nodeos-http-api
	# load balacing mode (roundrobin,leastconn,first,random,etc...)
	balance		leastconn
	# define the path and method for healthcheck
	option		httpchk GET /v1/chain/get_info HTTP/1.1
	# http 1.1 requires defining a Host
	http-check	send hdr Host eosrio.io
	# add default options for all servers
	default-server	check
	# define member servers
	server		node1 127.0.0.1:28888
	server		node2 127.0.0.1:38888

# ------ TCP Section -----------

# redefine defaults for tcp mode
defaults
	log		global
	mode		tcp
	# different timeouts can be defined here
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
	http-check      send hdr Host eosrio.io
	default-server	maxconn 64
	# specify peering servers with http ports used for health checking (they must have the chain_api_plugin enabled)
	server		peer1		127.0.0.1:29876 check port 28888
	server		peer2		127.0.0.1:39876 check port 38888

```

#### 5. Validating the configuration
```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```
A `Configuration file is valid` is expected.

#### 6. Starting
```bash
sudo haproxy -f /etc/haproxy/haproxy.cfg
```

#### Extra: Basic nodeos config.ini for this example
```
# Node 1
http-server-address = 127.0.0.1:28888
plugin = eosio::chain_api_plugin
http-validate-host = false

p2p-listen-endpoint = 127.0.0.1:29876
p2p-server-address = 127.0.0.1:29876
# connect to the other local node
p2p-peer-address = 127.0.0.1:39876
```

```
# Node 2
http-server-address = 127.0.0.1:38888
plugin = eosio::chain_api_plugin
http-validate-host = false

p2p-listen-endpoint = 127.0.0.1:39876
p2p-server-address = 127.0.0.1:39876
# connect to the other local node
p2p-peer-address = 127.0.0.1:29876
```
