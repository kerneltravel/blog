## nginx 安装与配置详解

分类：服务器 | 标签：nginx、安装、配置 | 发布时间：2013-09-21 11:28:00

___

#### 安装（需要支持 php）：

    sudo apt-get install nginx
    sudo apt-get install php5-fpm
    
#### nginx 的基本配置（/etc/nginx/nginx.conf）与参数说明：

    # 运行用户 www-data
    user www-data;
    # 启动进程,通常设置成和 cpu 的数量相等
    worker_processes 4;
    # PID文件
    pid /var/run/nginx.pid;
    
    # 工作模式及连接数上限
    events {
        # epoll是多路复用IO(I/O Multiplexing)中的一种方式,
        # 仅用于linux2.6以上内核,可以大大提高nginx的性能
        use epoll; 
    
        # 单个后台worker process进程的最大并发链接数    
        worker_connections 768;    
        
        # 是否允许在得到一个新连接的通知时，接收尽可能更多的连接
        # multi_accept on;
    }
    
    http {
    
        # 指定 nginx 是否调用 sendfile 函数（zero copy 方式）来输出文件，
        # 对于普通应用，必须设为 on,
        # 如果用来进行下载等应用磁盘IO重负载应用，可设置为 off，
        # 以平衡磁盘与网络I/O处理速度，降低系统的uptime。
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        # 连接超时时间
        keepalive_timeout 65;
        
        # 设定mime类型,类型由mime.type文件定义
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        
        #设定日志格式
        log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';
    
        # 全局错误日志
        access_log /var/log/nginx/access.log main;
        error_log /var/logs/error.log;
    
        # 开启 gzip 压缩
        gzip on;
        gzip_disable "MSIE [1-6].";

        gzip_vary off;
        gzip_comp_level 6;
        gzip_buffers 4 16k;
        gzip_http_version 1.1;
        gzip_min_length 1k;
        gzip_types text/plain application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
    
        # 设定请求缓冲
        client_header_buffer_size 128k;
        large_client_header_buffers 4 128k;
    
        # 设定虚拟主机配置
        server {
            # 侦听80端口
            listen 80;
            # server 名称
            server_name wenzhixin.net.cn;
    
            # 定义服务器的默认网站根目录位置
            root /var/www;
    
            # 默认请求
            location / {
                #定义首页索引文件的名称
                index index.php index.html index.htm;
                try_files $uri $uri/ /index.php;
                
                # 假如目录或者文件不存在，重写规则到 api.php 中
                if (!-e $request_filename) {
                    rewrite ^(.*)$ /api.php?url=$1 last;
                }
            }
    
            # 定义错误提示页面
            error_page 500 502 503 504 /50x.html;
            location = /50x.html {
            }
    
            # 静态文件，nginx自己处理
            location ~ ^/(images|javascript|js|css|flash|media|static|sea-module)/ {
                # 过期30天，静态文件不怎么更新，过期可以设大一点，
                # 如果频繁更新，则可以设置得小一点。
                expires 30d;
            }
    
            # PHP 脚本请求全部转发到 fastcgi 来处理。           
            location ~ .php$ {
                fastcgi_pass 127.0.0.1:9001;
                fastcgi_index index.php;
                fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
                
                fastcgi_buffer_size 128k;
                fastcgi_buffers 4 256k;
                fastcgi_busy_buffers_size 256k;
            }
    
            # 禁止访问 .htxxx 文件
                location ~ /.ht {
                deny all;
            }
    
            # 代理设置
            location /proxy {
                proxy_pass http://wenyi.tk;
                access_log off;
            }
        }
    }
 
#### php-fpm 的基本配置（/etc/php5/fpm/php-fpm.conf）与参数说明：
    
	# PID 设置
	pid = /var/run/php-fpm.pid
	 
	# 错误日志
	error_log = /var/log/php-fpm.log
	 
	# 错误级别。s可用级别为: alert（必须立即处理）, error（错误情况）, warning（警告情况）, notice（一般重要信息）, debug（调试信息）。s默认: notice。
	log_level = notice
	 
	# 表示在emergency_restart_interval所设值内出现SIGSEGV或者SIGBUS错误的php-cgi进程数如果超过 emergency_restart_threshold个，php-fpm就会优雅重启。这两个选项一般保持默认值。
	emergency_restart_threshold = 60
	emergency_restart_interval = 60s
	 
	# 设置子进程接受主进程复用信号的超时时间。可用单位: s(秒), m(分), h(小时), 或者 d(天) 默认单位: s(秒)。默认值: 0。
	process_control_timeout = 0
	 
	# 后台执行fpm,默认值为yes，如果为了调试可以改为no。在FPM中，可以使用不同的设置来运行多个进程池。这些设置可以针对每个进程池单独设置。
	daemonize = yes
	 
	# fpm监听端口，即nginx中php处理的地址，一般默认值即可。可用格式为: 'ip:port', 'port', '/path/to/unix/socket'。每个进程池都需要设置。
	listen = 127.0.0.1:9001
	 
	# backlog数，-1表示无限制，由操作系统决定，此行注释掉就行。
	listen.backlog = -1
	 
	# 允许访问FastCGI进程的IP，设置any为不限制IP，如果要设置其他主机的nginx也能访问这台FPM进程，listen处要设置成本地可被访问的IP。默认值是any。每个地址是用逗号分隔。如果没有设置或者为空，则允许任何服务器请求连接
	listen.allowed_clients = 127.0.0.1
	 
	listen.owner = www
	listen.group = www
	listen.mode = 0666
	 
	# 启动进程的帐户和组
	user = www
	group = www
	
	# 如何控制子进程，选项有static和dynamic。如果选择static，则由pm.max_children指定固定的子进程数。如果选择dynamic，则由下开参数决定：
	pm = dynamic
	
	pm.max_children # 子进程最大数
	pm.start_servers # 启动时的进程数
	pm.min_spare_servers # 保证空闲进程数最小值，如果空闲进程小于此值，则创建新的子进程
	pm.max_spare_servers # 保证空闲进程数最大值，如果空闲进程大于此值，此进行清理
	 
	# 设置每个子进程重生之前服务的请求数。对于可能存在内存泄漏的第三方模块来说是非常有用的。如果设置为 '0' 则一直接受请求。等同于 PHP_FCGI_MAX_REQUESTS 环境变量。默认值: 0。
	pm.max_requests = 1000
	 
	# FPM状态页面的网址。如果没有设置, 则无法访问状态页面。默认值: none。munin监控会使用到
	pm.status_path = /status
	 
	# FPM监控页面的ping网址。如果没有设置, 则无法访问ping页面。该页面用于外部检测FPM是否存活并且可以响应请求。请注意必须以斜线开头 (/)。
	ping.path = /ping
	 
	# 用于定义ping请求的返回相应。返回为 HTTP 200 的 text/plain 格式文本。默认值: pong。
	ping.response = pong
	 
	# 设置单个请求的超时中止时间。该选项可能会对php.ini设置中的'max_execution_time'因为某些特殊原因没有中止运行的脚本有用。设置为 '0' 表示 'Off'.当经常出现502错误时可以尝试更改此选项。
	request_terminate_timeout = 0
	 
	# 当一个请求该设置的超时时间后，就会将对应的PHP调用堆栈信息完整写入到慢日志中。设置为 '0' 表示 'Off'
	request_slowlog_timeout = 10s
	 
	# 慢请求的记录日志,配合request_slowlog_timeout使用
	slowlog = log/$pool.log.slow
	 
	# 设置文件打开描述符的rlimit限制。默认值: 系统定义值默认可打开句柄是1024，可使用 ulimit -n查看，ulimit -n 2048修改。
	rlimit_files = 1024
	 
	# 设置核心rlimit最大限制值。可用值: 'unlimited' 、0或者正整数。默认值: 系统定义值。
	rlimit_core = 0
	 
	# 启动时的Chroot目录。所定义的目录需要是绝对路径。如果没有设置, 则chroot不被使用。
	chroot =
	 
	# 设置启动目录，启动时会自动Chdir到该目录。所定义的目录需要是绝对路径。默认值: 当前目录，或者/目录（chroot时）
	chdir =
	 
	# 重定向运行过程中的stdout和stderr到主要的错误日志文件中。如果没有设置, stdout 和 stderr 将会根据FastCGI的规则被重定向到 /dev/null 。默认值: 空。
	catch_workers_output = yes