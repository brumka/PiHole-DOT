
## Configuring PiHole as DNS-over-TLS recursive DNS resolver


Using [PiHole](https://pi-hole.net/) is a  popular way to filter our ads, malware, and trackers.  It is easy to install and has excellent UI.  PiHole comes with the built in [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq) DNS resolver as well as the [lighttpd](https://en.wikipedia.org/wiki/Lighttpd) web server.

Over the past few years we start seeing [DNS-over-TLS](https://en.wikipedia.org/wiki/DNS_over_TLS) gaining more popularity.  For example: most of Android phones now come with so-called `secure DNS` enabled by default, which is a mere DOT.

For quite a few years, PiHole been doing great job for me.  I first installed it, just like many, on RasberryPi 3+ SBC, but later switched to [AtomicPI](https://ameridroid.com/products/atomic-pi) to get a full 1GB NIC and to avoid RPi SDcard's imminent failure (despite wear-leveling), since AtomicPI comes with the on-board eMMC storag, which is way faster and despite small size still sufficient for my purpose.  I could now turning the logging on without worrying of excessive file I/O killing the SD card.

However, with the advent of DOT and having mostly Android devices in my home network, I started noticing that PiHole becomes less efficient just due to the fact that my devices were using the default Google's DOT (8.8.8.8).  Thus, the task was to make PiHole relevant again.  To my surprise, PiHole developers refuse to add DOT and DOH functionality.

Luckily, there are quite a few resources on how to configure it.  Most are generic and required some tinkering.  Some PiHole users are opting out for using a cloudflare DOT/DOH gateway.  So, if you do not want to trust your ISP with collecting your browsing history, why would you trust Cloudflare, which is a for-profit company as well?

After some research I came up with switching from the default lighttpd webserver to nginx and, why not to install a proper DNS server as well?  Switching to UNBOUND is pretty easy.  As a result, I came up with the following design

![image](https://raw.githubusercontent.com/brumka/PiHole-DOT/main/ArchDiagram.png)


##Installing UNBOUND
======

There is an easy to follow [guide on pihole pages](https://docs.pi-hole.net/guides/dns/unbound/).  Once it is installed you can validate it running by either running 

```
$ dig @YourDNSServer -P 5335 www.github.com +nocomment

; <<>> DiG 9.11.3-1Ubuntu1.14-Ubuntu <<>> @YourDNSServer -p 5335 www.github.com +nocomment
; (1 server found)
;; global options: +cmd
;www.github.com.                        IN      A
www.github.com.         2101    IN      CNAME   github.com.
github.com.             60      IN      A       140.82.114.3
;; Query time: 15 msec
;; SERVER: YourDNSServer#5335(YourDNSServerIP)
;; WHEN: Wed Apr 07 12:44:59 DST 2021
;; MSG SIZE  rcvd: 73
```
Note the port number 5335 - it is defined in `/etc/unbound/unbound.conf`.  


##Installing NGINX
===============

The PiHole's [guide on installing NGINX](https://docs.pi-hole.net/guides/webserver/nginx/) is pretty straightforward as well.  Note, that we will need to install a valid TLS certificate.  I used Les's Encrypt, which is quite simple to get installed using [these instructions](https://letsencrypt.org/getting-started/).

Next we will follow the example of configuring the DNS-over-TLS gateway using the excellent [Mark Boddington's example](https://github.com/TuxInvader/nginx-dns/blob/master/examples/nginx-dot-to-dns-simple.conf).

Note that you will need to install the NGINX NJS module.  To do that on my Ubuntu 18.04 I had to add NGINX repository to the list of sources first by adding the following lines to  `/etc/apt/sources.list`

```
deb http://nginx.org/packages/mainline/ubuntu/ bionic nginx
deb-src http://nginx.org/packages/mainline/ubuntu/ bionic nginx
```
<br>and only then running 

```
$ sudo apt-get install nginx-module-njs
```
<br>
<br>
As a result, my `dot.conf` looked like this 

```
stream {
    # DNS logging
    log_format  dns   '$remote_addr [$time_local] $protocol "$dns_qname"';
    access_log /var/log/nginx/dns-access.log dns;  # this log file will show your the requests geting forwarded to UNBOUND

    # Include the NJS module.  Get the file from  https://github.com/TuxInvader/nginx-dns/tree/master/njs.d
    js_include /etc/nginx/njs.d/nginx_stream.js;

    # The $dns_qname variable can be populated by preread calls, and can be used for DNS routing
    js_set $dns_qname dns_get_qname;

    # DNS upstream pool.
    upstream dns {
        zone dns 64k;
        server 127.0.0.1:53;  # local PIHOLE instance.  For UNBOUND bypassing PIHOLE change the port to 5335
    }

    # DNS over TLS (DoT) gateway
    # Terminate DoT TCP, and proxy the traffic onto standard DNS
    server {
        listen 853 ssl;
        # Note that we're re-using the Let's Encrypt certificates
        ssl_certificate /etc/letsencrypt/live/YourDNSServer/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/YourDNSServer/privkey.pem;
        js_preread dns_preread_dns_request;
        proxy_pass dns;
    }
}
```
<br>
<br>Added the call to load NJS modules and referenced the dot.conf at the bottom to my current /etc/nginx/nginx.conf

```	
	user  nginx;
	worker_processes  1;
	
	error_log  /var/log/nginx/error.log warn;
	pid        /var/run/nginx.pid;
	
	# This is where we load NJS modules 
	load_module modules/ngx_http_js_module.so;
	load_module modules/ngx_stream_js_module.so;
	
	events {
	    worker_connections  1024;
	}
	
	http {
	    include       /etc/nginx/mime.types;
	    default_type  application/octet-stream;
	
	    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
	                      '$status $body_bytes_sent "$http_referer" '
	                      '"$http_user_agent" "$http_x_forwarded_for"';
	
	    access_log  /var/log/nginx/access.log  main;
	
	    sendfile        on;
	    keepalive_timeout  65;
	
	    include /etc/nginx/conf.d/*.conf;
	}
  
	include /etc/nginx/dns.d/*.conf;  # this is the way to include dot.conf
  
```
<br>
And the final NGINX `/etc/nginx/conf.d/default.conf` file.  Note the comments explaining redirects, TLS1.2 and H2

```
        server {
            listen 80 default_server;
            listen [::]:80 default_server;

            server_name YourDNSServer;

            root /var/www/html;

            autoindex off;

            index pihole/index.php index.php index.html index.htm;

            location / {
                expires max;
                try_files $uri $uri/ =404;
            }

            location ~ \.php$ {
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root/$fastcgi_script_name;
                fastcgi_pass unix:/run/php/php7.2-fpm.sock;  # Ensure you use the correct PHP-FPM version here
                fastcgi_param FQDN true;
            }

            location /*.js {
                index pihole/index.js;
            }

            location /admin {
                root /var/www/html;
                index index.php index.html index.htm;
            }

            location ~ /\.ht {
                deny all;
            }

            # turn on the HTTP-to-HTTPS 302 redirect
            if ($host = YourDNSServerIPaddress) {
                return 302 http://YourDNSServer$request_uri;
            }
            return 302 https://YourDNSServer$request_uri;
        }

        ############################################################################
        ############# now let's do the same same for the port 443  #################
        server {
            # note the http2 verb added here
            listen [::]:443 http2 ssl ipv6only=on;  # Note that we enable H2 support, because why not 
            listen 443 ssl http2 default_server; 

            server_name YourDNSServer;
            
            # I always like using the FQDN instead of an IP address
            if ($host = YourDNSServerIPaddress) {
                return 302 https://YourDNSServer$request_uri;
            }
            # Here comes our Let's Encrypt TLS certificate.  These lines were automagically added by CERTBOT as I was running
            # its script to generate the cert
            ssl_certificate /etc/letsencrypt/live/YourDNSServer/fullchain.pem; 
            ssl_certificate_key /etc/letsencrypt/live/YourDNSServer/privkey.pem;
            include /etc/letsencrypt/options-ssl-nginx.conf;
            ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; 

            # enable TLSv1.2.  I am seeing it negotiating TLS 1.3 as well
            ssl_protocols TLSv1.2;

            root /var/www/html;

            autoindex off;

            index pihole/index.php index.php index.html index.htm;

            location / {
                expires max;
                try_files $uri $uri/ =404;
            }

            location ~ \.php$ {
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root/$fastcgi_script_name;
                fastcgi_pass unix:/run/php/php7.2-fpm.sock;  # again, ensure you have the correct PHP-FPM version
                fastcgi_param FQDN true;
            }

            location /*.js {
                index pihole/index.js;
            }

            location /admin {
                root /var/www/html;
                index index.php index.html index.htm;
            }

            location ~ /\.ht {
                deny all;
            }
        }
```
<br><br>
##Testing DNS-over-TLS
=======================

First start with installing the enhanced DIG by running<br>
`$ apt-get install knot-dnsutils`
<br>
Now we can test it
```
$ kdig -d @YourDNSServerIPaddress +tls-ca +tls-host=YourDNSServer www.github.com

;; DEBUG: Querying for owner(www.github.com.), class(1), type(1), server(YourDNSServerIPaddress), port(853), protocol(TCP)
;; DEBUG: TLS, imported 129 system certificates
;; DEBUG: TLS, received certificate hierarchy:
;; DEBUG:  #1, CN=YourDNSServer
;; DEBUG:      SHA-256 PIN: WV+yn9eTQPtc6EZ8hdOE3VUlBGVt2G2ehX9kEVcrF+E=
;; DEBUG:  #2, C=US,O=Let's Encrypt,CN=R3
;; DEBUG:      SHA-256 PIN: jQJTbIh0grw0/1TkHSumWb+Fs0Ggogr621gT3PvPKG0=
;; DEBUG: TLS, skipping certificate PIN check
;; DEBUG: TLS, The certificate is trusted.
;; TLS session (TLS1.2)-(ECDHE-SECP256R1)-(RSA-SHA256)-(AES-256-GCM)
;; ->>HEADER<<- opcode: QUERY; status: NOERROR; id: 58
;; Flags: qr rd ra; QUERY: 1; ANSWER: 2; AUTHORITY: 0; ADDITIONAL: 1

;; EDNS PSEUDOSECTION:
;; Version: 0; flags: ; UDP size: 1472 B; ext-rcode: NOERROR

;; QUESTION SECTION:
;; www.github.com.              IN      A

;; ANSWER SECTION:
www.github.com.         0       IN      CNAME   github.com.
github.com.             0       IN      A       140.82.113.4

;; Received 73 B
;; Time 2021-04-07 13:53:23 DST
;; From YourDNSServerIPaddress@853(TCP) in 18.6 ms
```
Note how it uses port `853` and the details of TLS handshake.
