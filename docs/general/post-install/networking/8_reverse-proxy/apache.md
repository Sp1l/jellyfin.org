---
uid: network-reverse-proxy-apache
title: Apache
---

"The [Apache HTTP Server Project](https://httpd.apache.org/) is an effort to develop and maintain an open-source HTTP server for modern operating systems including UNIX and Windows. The goal of this project is to provide a secure, efficient and extensible server that provides HTTP services in sync with the current HTTP standards."

```conf
Define VHOSTNAME jellyfin.example.org
Define SERVER_IP_ADDRESS 192.2.0.1

<IfModule md_module>
MDCertificateAuthority https://acme-v02.api.letsencrypt.org/directory
MDCertificateAgreement accepted

MDomain ${VHOSTNAME}
</IfModule>

<VirtualHost *:80>
    ServerName ${VHOSTNAME}

    # Comment to prevent HTTP to HTTPS redirect
    Redirect permanent / https://DOMAIN_NAME/

    ErrorLog /var/log/apache2/${VHOSTNAME}-error.log
    CustomLog /var/log/apache2/${VHOSTNAME}-access.log combined
</VirtualHost>

# If you are not using a SSL certificate, replace the 'redirect'
# line above with all lines below starting with 'Proxy'
<IfModule ssl_module>
<VirtualHost *:443>
    ServerName ${VHOSTNAME}
    # This folder exists just for certbot (You may have to create it, chown and chmod it to give apache permission to read it)
    DocumentRoot /var/www/html/jellyfin/public_html

    ProxyPreserveHost On
    <IfModule http2_module>
    Protocols h2 http/1.1
    </IfModule>
    <IfModule !http2_module>
    Protocols http/1.1
    </IfModule>

    # Letsencrypt's certbot will place a file in this folder when updating/verifying certs
    # This line will tell apache to not to use the proxy for this folder.
    ProxyPass "/.well-known/" "!"

    # Tell Jellyfin to forward requests that came from TLS connections
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Port "443"

    # Apache should be able to know when to change protocols (between WebSocket and HTTP)
    ProxyPass /socket/ http://${SERVER_IP_ADDRESS}:8096/socket/ upgrade=websocket
    ProxyPass / http://${SERVER_IP_ADDRESS}:8096/

    # Sometimes, Jellyfin requires clients to empty their cache to display and function correctly.
    # This header tells clients not to keep any cache and is quite strict on that.
    # This might also fix some syncplay issues (#5485 and #8140 @ https://github.com/jellyfin/jellyfin-web/issues/)
    # Header set Cache-Control "no-store, no-cache, must-revalidate, max-age=0"

    SSLEngine on
    <IfModule !md_module>
    SSLCertificateFile /etc/letsencrypt/live/DOMAIN_NAME/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/DOMAIN_NAME/privkey.pem
    </IfModule>

    # Enable only strong encryption ciphers and prefer versions with Forward Secrecy
    # See https://ssl-config.mozilla.org/#server=apache&version=2.4.47&config=intermediate&openssl=3.0.0
    SSLProtocol             -all +TLSv1.2 +TLSv1.3
    SSLOpenSSLConfCmd       Curves X25519:prime256v1:secp384r1
    SSLCipherSuite          ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305
    SSLHonorCipherOrder     off
    SSLSessionTickets       off
    
    ErrorLog /var/log/apache2/${VHOSTNAME}-error.log
    CustomLog /var/log/apache2/${VHOSTNAME}-access.log combined
</VirtualHost>
</IfModule>

Undefine VHOSTNAME
Undefine SERVER_IP_ADDRESS
```

If you encounter errors, you may have to enable `mod_proxy`, `mod_ssl`, `proxy_wstunnel`, `http2`, `headers` and `remoteip` support manually.

```bash
sudo a2enmod proxy proxy_http ssl proxy_wstunnel remoteip http2 headers
```

## Apache with Subpath (example.org/jellyfin)

When connecting to server from a client application, enter `http(s)://DOMAIN_NAME/jellyfin` in the address field.

Set the [base URL](../index.md#base-url) field in the Jellyfin server. This can be done by navigating to the Admin Dashboard -> Networking -> Base URL in the web client. Fill in this box with `/jellyfin` and click Save. The server will need to be restarted before this change takes effect.

:::caution

HTTP is insecure. The following configuration is provided for ease of use only. If you are planning on exposing your server over the Internet you should setup HTTPS. [Let's Encrypt](https://letsencrypt.org/getting-started/) can provide free TLS certificates which can be installed easily with Apache's [mod_md](https://httpd.apache.org/docs/2.4/mod/mod_md.html).

:::

The following configuration can be saved in `/etc/httpd/conf/extra/jellyfin.conf` and included in your vhost.

```conf
# Jellyfin hosted on http(s)://DOMAIN_NAME/jellyfin
ProxyPreserveHost On
ProxyPass "/jellyfin/socket" "http://${SERVER_IP_ADDRESS}:8096/jellyfin/socket" upgrade=websocket
ProxyPass "/jellyfin" "http://${SERVER_IP_ADDRESS}:8096/jellyfin"
ProxyPassReverse "/jellyfin" "http://${SERVER_IP_ADDRESS}:8096/jellyfin"
```
