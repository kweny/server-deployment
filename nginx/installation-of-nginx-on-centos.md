# Installation of Nginx on CentOS

## 环境准备

```txt
4C8G
CentOS 7 64
```

### 时间同步

```shell
# 安装 ntpdate、设置时区、同步时间
yum install -y ntpdate
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
ntpdate cn.pool.ntp.org

# 以系统时间为基准，修改硬件时间
hwclock --systohc
hwclock -w

# 安装 ntp、配置、启动
yum install -y ntp
# 配置/etc/ntp.conf
systemctl start ntpd
systemctl enable ntpd
```

### 安装依赖

```shell
yum install -y wget zip unzip patch
yum install -y gcc gcc-c++
yum install -y pcre pcre-devel
yum install -y zlib zlib-devel
yum install -y openssl openssl-devel
```

## Nginx

### 下载解压

```bash
mkdir /root/nginx
cd /root/nginx

wget https://github.com/yaoweibin/nginx_upstream_check_module/archive/master.zip -O nginx_upstream_check_module.zip
wget https://github.com/gnosek/nginx-upstream-fair/archive/master.zip -O nginx-upstream-fair.zip
wget https://github.com/vozlt/nginx-module-vts/archive/master.zip -O nginx-module-vts.zip
wget https://nginx.org/download/nginx-1.16.1.tar.gz

unzip nginx_upstream_check_module.zip
unzip nginx-upstream-fair.zip
unzip nginx-module-vts.zip
tar zxvf nginx-1.16.1.tar.gz
```

[nginx_upstream_check_module][1]：上游服务健康检查模块
[nginx-upstream-fair][2]：负载均衡模块
[nginx-module-vts][3]：状态监控模块

### 编译安装

```bash
cd /root/nginx/nginx-upstream-fair-master
sed -i 's/default_port/no_port/g' ngx_http_upstream_fair_module.c
patch -p1 < /root/nginx/nginx_upstream_check_module-master/upstream_fair.patch

cd /root/nginx/nginx-1.16.1
patch -p1 < /root/nginx/nginx_upstream_check_module-master/check_1.16.1+.patch
```

```bash
./configure \
--prefix=/usr/local/nginx \
--with-http_realip_module \
--with-http_ssl_module \
--with-http_addition_module \
--with-http_sub_module \
--with-http_dav_module \
--with-http_gzip_static_module \
--with-http_stub_status_module \
--with-stream \
--with-http_stub_status_module \
--add-module=/root/nginx/nginx_upstream_check_module-master \
--add-module=/root/nginx/nginx-upstream-fair-master \
--add-module=/root/nginx/nginx-module-vts-master

make
make install
```

## 配置

### nginx.conf

编辑配置文件：/usr/local/nginx/conf/nginx.conf

```bash
pid /usr/local/nginx/nginx.pid;

worker_processes 4;
worker_cpu_affinity 1000 0100 0010 0001;
worker_rlimit_nofile 102400;

events {
    worker_connections 102400;
    multi_accept on;
    use epoll;
}

http {

    vhost_traffic_status_zone;

    log_format json_format '{"timestamp":"$msec",'
        '"time_iso":"$time_iso8601",'
        '"time_local":"$time_local",'
        '"request_time":"$request_time",'
        '"remote_user":"$remote_user",'
        '"remote_addr":"$remote_addr",'
        '"http_x_forwarded_for":"$http_x_forwarded_for",'
        '"request":"$request",'
        '"status":"$status",'
        '"body_bytes_send":"$body_bytes_sent",'
        '"upstream_addr":"$upstream_addr",'
        '"upstream_response_time":"$upstream_response_time",'
        '"upstream_http_content_type":"$upstream_http_content_type",'
        '"upstream_http_content_disposition":"$upstream_http_content_disposition",'
        '"upstream_status":"$upstream_status",'
        '"http_user_agent":"$http_user_agent",'
        '"http_referer":"$http_referer",'
        '"connection":"$connection",'
        '"connection_requests":"$connection_requests",'
        '"scheme":"$scheme",'
        '"host":"$host",'
        '"http_via":"$http_via",'
        '"request_id":"$request_id"}';

    access_log /usr/local/nginx/logs/access.log json_format buffer=16k;
    error_log /usr/local/nginx/logs/error.log error;

    server_tokens off;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 30;
    keepalive_requests 100000;
    client_header_timeout 10;
    client_body_timeout 10;
    client_max_body_size 100m;
    reset_timedout_connection on;
    send_timeout 10;
    limit_conn_zone $binary_remote_addr zone=addr:5m;
    limit_conn addr 100;
    include mime.types;
    default_type application/octet-stream;
    charset UTF-8;

    gzip on;
    gzip_vary on;
    gzip_disable "MSIE [1-6].";
    gzip_http_version 1.0;
    gzip_comp_level 4;
    # gzip_static on;
    gzip_min_length 1024;
    gzip_buffers 4 16k;
    gzip_proxied expired no-cache no-store private auth;
    gzip_types text/plain text/css text/javascript application/x-javascript application/xml application/json application/xml+rss;

    open_file_cache max=100000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
    proxy_connect_timeout 75;
    proxy_read_timeout 300;
    proxy_send_timeout 300;
    proxy_buffer_size 64k;
    proxy_buffers 4 64k;
    proxy_busy_buffers_size 128k;
    proxy_temp_file_write_size 128k;

    include /usr/local/nginx/servers/*.conf;
}
```

### 添加监控 server

编辑配置文件 /usr/local/nginx/servers/status.conf

```bash
server {
    listen 80;
    server_name localhost;

    location /vts {
        access_log off;
        vhost_traffic_status_display;
        vhost_traffic_status_display_format html;
    }

    location /stub {
        stub_status on;
        access_log off;
    }
}
```

### 服务脚本

编辑服务脚本 /usr/lib/systemd/system/nginx.service

```bash
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```bash
chmod +x /usr/lib/systemd/system/nginx.service
```

### 修改系统 ulimit

编辑 /etc/security/limits.conf，设置或添加以下配置，重新登录后使用 ulimit -a 命令查看是否生效。

```bash
*  soft    nofile  102400
*  hard    nofile  102400
*  soft    nproc  102400
*  hard    nproc  102400
```

### 内核优化

编辑 /etc/sysctl.conf，设置或添加以下配置，并执行 `sysctl -p /etc/sysctl.conf` 命令使之生效。

```bash
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 131072
net.core.somaxconn = 131072
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_tw_buckets = 524288
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.ip_local_port_range = 2000 65000
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.tcp_window_scaling = 1
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_default = 8388608
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_congestion_control = cubic
net.netfilter.nf_conntrack_tcp_timeout_established = 1200
net.netfilter.nf_conntrack_max = 655350
net.core.netdev_max_backlog = 524288
kernel.core_pattern = core_%e
kernel.panic = 1
vm.panic_on_oom = 1
vm.min_free_kbytes = 524288
vm.swappiness = 0
fs.inotify.max_user_watches = 8388608
fs.aio-max-nr = 4194304
fs.file-max = 1048560
```

  [1]: https://github.com/yaoweibin/nginx_upstream_check_module
  [2]: https://github.com/gnosek/nginx-upstream-fair
  [3]: https://github.com/vozlt/nginx-module-vts