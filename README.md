# docker-haproxy-letsencrypt
haproxy docker image based on Debian Stretch haproxy 1.7 package with built-in acme-plugin and zero-downtime auto-reload on configuration / certificate changes. This image was created for use with [letsencrypt-manager](https://github.com/bringnow/docker-letsencrypt-manager).

For integrating the acme-plugin, see [its documentation](https://github.com/janeczku/haproxy-acme-validation-plugin/).

Example haproxy config file using acme webroot plugin:

```
global
	log /dev/log local0
  log /dev/log local1 notice
	chroot /var/lib/haproxy

	# Default SSL material locations
	crt-base /etc/letsencrypt/live

	lua-load /etc/haproxy/acme-http01-webroot.lua

	# Default ciphers to use on SSL-enabled listening sockets.
	# For more information, see ciphers(1SSL).
	ssl-default-bind-ciphers AES256+EECDH:AES256+EDH:!aNULL;

	tune.ssl.default-dh-param 4096

defaults
  	log	global
  	mode	http

frontend http
	bind *:80
	mode http
	acl url_acme_http01 path_beg /.well-known/acme-challenge/
	http-request use-service lua.acme-http01 if METH_GET url_acme_http01
	redirect scheme https if !url_acme_http01

frontend https
	bind :443 ssl no-tls-tickets crt example.com/haproxy.pem no-sslv3 no-tls-tickets no-tlsv10 no-tlsv11
	mode http

  reqadd X-Forwarded-Proto:\ https
	rspadd Strict-Transport-Security:\ max-age=15768000

frontend http-in
  mode http
  bind *:443 ssl crt /etc/haproxy/certificate.pem
  default_backend rancher_servers

  # Add headers for SSL offloading
  http-request set-header X-Forwarded-Proto https if { ssl_fc }
  http-request set-header X-Forwarded-Ssl on if { ssl_fc }

  acl is_websocket hdr(Upgrade) -i WebSocket
  acl is_websocket hdr_beg(Host) -i ws
  use_backend rancher_servers if is_websocket

use_backend		www	if { hdr(host) example.com or www.example.com }

backend www
	redirect prefix	https://www.example.com code 301 unless { hdr(host) www.example.com }
	option forwardfor
	server node1 127.0.0.1:6000
backend rancher_servers
  server websrv1 <rancher_server_1_IP>:8080 weight 1 maxconn 1024

```

## Logging

This image is setup for haproxy logging to rsyslog. You can access the log file using the following command:
```
$ docker-compose exec haproxy tail -f /var/log/haproxy.log
```
