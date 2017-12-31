#!/bin/sh

##################s###################################################
#    Make sure to run the setup described in the README.md first!    #
######################################################################

check_hash() {
    file_hash=`shasum $1 | cut -d " " -f1`
    if [ "$file_hash" != "$2" ] ; then
        echo "HASH for $1 does not match expected hash"
        exit 1
    fi
}

# Downloads the latest IPFS binaries from the Shift IPFS cluster â€“ inception
install_binaries() {
    mkdir -p ~/bin
    echo "Downloading latest IPFS binariesâ€¦"
    if [ ! -f ~/bin/ipfs ] ; then
        wget -O ~/bin/ipfs https://gateway-test.shiftnrg.org/ipfs/QmY2qzXDDczVZ8UWR6G1LHiqTBPhhobRBgJsiyDkBfMNiP
        chmod +x ~/bin/ipfs
        check_hash ~/bin/ipfs "9fc871c2522e0bb1fa0ce59e9dc968cb6658ac1d"
    fi

    if [ ! -f ~/bin/ipfs-cluster-ctl ] ; then
        wget -O ~/bin/ipfs-cluster-ctl https://gateway-test.shiftnrg.org/ipfs/QmahAfgUBiYMAGTPhpAeBvELbdCxUkybPCPPmRhTbZeYxv
        chmod +x ~/bin/ipfs-cluster-ctl
        check_hash ~/bin/ipfs-cluster-ctl "6a63bb284574924a931e4d1f165b4be1d17b42d8"
    fi

    if [ ! -f ~/bin/ipfs-cluster-service ] ; then
        wget -O ~/bin/ipfs-cluster-service https://gateway-test.shiftnrg.org/ipfs/QmRrGMVEmSm9kGiFBYb5RB9rwtLphvDeucbPT1eVhR3q8u
        chmod +x ~/bin/ipfs-cluster-service
        check_hash ~/bin/ipfs-cluster-service "be68fd0c1f6a565a3d1315546b76d0930457440f"
    fi
}

# Sets up a self-signed certificate for https
install_certificate() {
    sudo openssl genrsa -out shift.key 2048
    openssl req -nodes -newkey rsa:2048 -key shift.key -out shift.csr -subj "/C=NL/O=Shift/CN=shiftnrg.org"
    sudo openssl x509 -req -days 365 -in shift.csr -signkey shift.key -out shift.crt
    sudo bash -c 'cat shift.key shift.crt >> /etc/ssl/private/shift.pem'
    rm -f shift.key shift.csr shift.crt
}

# @todo pull the HAProxy config from somewhere else â€“Â maybe IPFS?
update_haproxy_config() {
    # Configure HAProxy
    sudo rm /etc/haproxy/haproxy.cfg

    conf="global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

    # Default SSL material locations
    ca-base /etc/ssl/certs
    crt-base /etc/ssl/private

    # Default ciphers to use on SSL-enabled listening sockets.
    # For more information, see ciphers(1SSL). This list is from:
    #  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
    # An alternative list with additional directives can be obtained from
    #  https://mozilla.github.io/server-side-tls/ssl-config-generator/?server=haproxy
    ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS
    ssl-default-bind-options no-sslv3
    tune.ssl.default-dh-param 2048

defaults
    log global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend https-in
    bind *:443 ssl crt /etc/ssl/private/shift.pem
    bind *:80
    mode http
    # Add CORS headers when Origin header is present
    capture request header origin len 128
    http-response add-header Access-Control-Allow-Origin %[capture.req.hdr(0)] if { capture.req.hdr(0) -m found }
    rspadd Access-Control-Allow-Methods:\ GET,\ HEAD,\ OPTIONS,\ POST,\ PUT  if { capture.req.hdr(0) -m found }
    rspadd Access-Control-Allow-Credentials:\ true  if { capture.req.hdr(0) -m found }
    rspadd Access-Control-Allow-Headers:\ Origin,\ Accept,\ X-Requested-With,\ Content-Type,\ Access-Control-Request-Method,\ Access-Control-Request-Headers,\ Authorization  if { capture.req.hdr(0) -m found }

    acl is_cluster_api_path path_beg /pins
    acl is_api_path path_beg /api/v0/ls /api/v0/add /api/v0/cat /api/v0/object/links /api/v0/object/patch/add-link /api/v0/object/patch/rm-link /api/v0/key/gen /api/v0/key/list /api/v0/name/publish /api/v0/name/resolve /api/v0/ping /api/v0/version /api/v0/swarm/peers /api/v0/swarm/addrs/local /api/v0/stats/bw /api/v0/stats/repo /api/v0/stats/
    use_backend ipfs_cluster_api if is_cluster_api_path
    use_backend ipfs_api if is_api_path
    default_backend ipfs

backend ipfs_api
    mode http
    http-request del-header Origin
    http-request del-header Referer
    http-response add-header Server ipfs_api
    server api 127.0.0.1:5001

backend ipfs_cluster_api
    mode http
    acl is_options method OPTIONS
    http-request set-method GET if is_options
    http-response add-header Server ipfs_cluster_api
    server api 127.0.0.1:9094

backend ipfs
    mode http
    balance roundrobin
    option forwardfor
    option httpchk GET / HTTP/1.1\\\r\\\nHost:localhost
    server ipfs 127.0.0.1:8080
    http-request set-header X-Forwarded-Port %[dst_port]
    http-request add-header X-Forwarded-Proto https if { ssl_fc }"

    echo "$conf" | sudo tee -a /etc/haproxy/haproxy.cfg > /dev/null
}

# Installs HAProxy and the HAProxy config for routing requests to IPFS
install_haproxy() {
    sudo add-apt-repository -y ppa:vbernat/haproxy-1.7
    sudo apt update
    sudo apt install -y haproxy
    update_haproxy_config
}

initialize_ipfs() {
    ipfs init
    tmp=$(mktemp)
    jq '.Bootstrap = []' ~/.ipfs/config > "$tmp" && mv "$tmp" ~/.ipfs/config

    CLUSTER_SECRET=5cdc314d43fb59f3fcd55a947d3aabec63c62b7ca1d77a5c1354a3a16a563faa ipfs-cluster-service init
    jq '.cluster.peers = [] | .cluster.bootstrap = [] | .consensus.raft.heartbeat_timeout = "10s" | .consensus.raft.election_timeout = "10s" | .cluster.leave_on_shutdown = true | .cluster.replication_factor = 3' ~/.ipfs-cluster/service.json > "$tmp" && mv "$tmp" ~/.ipfs-cluster/service.json

    swarm_key="/key/swarm/psk/1.0.0/
/base16/
4e8b30ef49efcba7cca26690d4c6a743a955b40a01675bfe3de481e7dad0d95f"
    echo "$swarm_key" | sudo tee -a ~/.ipfs/swarm.key > /dev/null

    user=`whoami`
    sudo chown $user:$user ~/.ipfs
    sudo chown $user:$user ~/.ipfs-cluster
}

# Installs what is necessary to run a storage node
# @todo support different operating systems (linux only now)
install() {
    cd ~/
    install_binaries
    initialize_ipfs
    install_certificate
    install_haproxy
    echo "SHIFT cluster successfully installed"
}

remove() {
    stop
    rm ~/bin/ipfs
    rm ~/bin/ipfs-cluster-ctl
    rm ~/bin/ipfs-cluster-service
    sudo apt-get -y remove haproxy
    sudo rm /etc/ssl/private/shift.pem
    rm -rf ~/.ipfs
    rm -rf ~/.ipfs-cluster
    echo "SHIFT cluster successfully removed"
}

start() {
    mkdir -p ~/logs
    sudo /etc/init.d/haproxy restart
    nohup ipfs daemon > ~/logs/ipfs.log 2>&1 &
    sleep 2
    nohup ipfs-cluster-service --bootstrap /ip4/45.77.136.112/tcp/9096/ipfs/QmQzQfwpnHC4KvVYJpXkje5snjg7a8nHYSrHhMA4qhzfHG -f > ~/logs/ipfs-cluster.log 2>&1 &
    echo "SHIFT cluster started"
}

stop() {
    sudo /etc/init.d/haproxy stop
    pkill -f ipfs
    echo "SHIFT cluster stopped"
}

# Checks if IPFS is running
#
# @todo make sure it is actually serving traffic
check() {
    netstat -tlnp 2>/dev/null | grep 8080 -q
    if [ $? -eq 0 ] ; then
        echo "IPFS is running"
    else
        echo "IPFS is not running"
    fi
}

case $1 in
    "install")
      install
    ;;
    "remove")
      remove
    ;;
    "start")
      start
    ;;
    "stop")
      stop
    ;;
    "check")
      check
    ;;
    *)
    echo "Available options: install, remove, start, stop, check"
    echo "Usage: ./shift-cluster install"
    exit 1
    ;;
esac
