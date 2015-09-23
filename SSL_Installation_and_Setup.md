
```

OS:  CentOS 6.5 x86_x64

Required:  ELSA installed and working.

    Install the following:
   # yum install mod_ssl
   # cpanm AnyEvent::TLS
   # cpanm Net::SSLeay
   Needed or you might get the following:  "TLS support not available on this system" after everything is configured.
    Generate a SSL certificate
   # openssl genrsa -out /etc/pki/tls/private/elsa01.key 2048
   # openssl req -new -out /etc/pki/tls/private/elsa01.csr -key /etc/pki/tls/private/elsa01.key
   # openssl x509 -req -days 3650 -in /etc/pki/tls/private/elsa01.csr -signkey /etc/pki/tls/private/elsa01.key -out /etc/pki/tls/private/elsa01.crt
    Edit the ZZelsa.conf file and add the following information
   # vi /etc/httpd/conf.d/ZZelsa.conf
   NameVirtualHost *:80
   <VirtualHost *:80>
           #  General setup for the virtual host
           ServerName elsa
           Redirect permanent / https://elsa01/
   </VirtualHost>
   <VirtualHost *:443>
           #  General setup for the virtual host
           ServerName elsa
           SSLEngine on
           SSLCertificateFile      /etc/pki/tls/private/elsa01.crt
           SSLCertificateKeyFile   /etc/pki/tls/private/elsa01.key
           SSLHonorCipherOrder on
           DocumentRoot /usr/local/elsa/web/lib
           SetEnv ELSA_CONF /etc/elsa_web.conf
           <Location "/">
                   Order Allow,Deny
                   Allow from all
                   SetHandler perl-script
                   PerlResponseHandler Plack::Handler::Apache2
                   PerlSetVar psgi_app /usr/local/elsa/web/lib/Web.psgi
           </Location>
           # Cleanup proxied HTTP auth
           RewriteEngine on
           RewriteCond %{HTTP:Authorization} ^(.*)
           RewriteRule ^(.*) - [E=HTTP_AUTHORIZATION:%1]
           ErrorLog /var/log/elsa_web_error
   </VirtualHost>
    Edit the ssl.conf file and change the following
   # vi /etc/httpd/conf.d/ssl.conf
   SSLCertificateFile /etc/pki/tls/private/elsa01.crt
   SSLCertificateKeyFile /etc/pki/tls/private/elsa01.key
    Restart httpd service
   # service httpd restart
    Configure IPTABLES to allow port 80 and 443 with the interface matching the correct NIC (ie: eth0, eth1, eth2, etc.) & IP address range. (ie: 192.168.0.0/16) or turn IPTABLES off.
   -A INPUT -i eth0 -p tcp -s 192.168.0.0/16 --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
   -A INPUT -i eth0 -p tcp -s 192.168.0.0/16 --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
    Verify the certificate is installed
   # openssl s_client -connect localhost:443
```