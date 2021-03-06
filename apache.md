### Apache setting with latest lets-encrypt-x3-cross-signed.pem, 
### Got hints from the following link:

https://github.com/diafygi/acme-tiny/issues/79

#### commands used in setting up the ssl, system: ubuntu 14.04 + virtualmin + apache2:


```
openssl genrsa 4096 > account.key
openssl genrsa 4096 > domain.key
openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:xxxxxx.com,DNS:www.xxxxxx.com")) > domain.csr
mkdir -p /home/xxxxxxx/public_html/challenges/
wget -O - https://raw.githubusercontent.com/diafygi/acme-tiny/master/acme_tiny.py > acme_tiny.py
```

#### before get certificate signed, we need following config into apache2's http(port 80's setup) setup:
```
   Alias /.well-known/acme-challenge/ /home/xxxxxxx/public_html/challenges/
   <Directory /home/xxxxxxx/public_html/challenges/>
      AllowOverride None
      Require all granted
      Satisfy Any
   </Directory>
```

#### then we need restart apache2 and get certificate signed:
```
service apache2 restart
python acme_tiny.py --account-key ./account.key --csr ./domain.csr --acme-dir /home/xxxxxx/public_html/challenges/ > ./signed.crt
wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > lets-encrypt-x3-cross-signed.pem
a2enmod headers
service apache2 restart

```

#### renew ssl every month: renew_cert.sh


```
nano renew_cert.sh

```

```

#!/bin/sh
python acme_tiny.py --account-key ./account.key --csr ./domain.csr --acme-dir /home/xxxxxx/public_html/challenges/ > ./signed.crt || exit
wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > lets-encrypt-x3-cross-signed.pem
service apache2 reload

```

#### make renew_cert.sh executable

```
chmod +x renew_cert.sh
```
#### test executing renew_cert.sh to see if there is any errors

```
./renew_cert.sh
```


#### crontab -e

```
0 0 1 * * /usr/local/etc/apache/keys/renew_cert.sh 2>> /var/log/acme_tiny.log


```




#### Apache2 conf settings:
##### If you do not have the rest of config for 443, then just copy 80's rest of config  to 443's:



```
<VirtualHost *:80>
   ServerName www.yoursite.com
   ServerAlias yoursite.com

   Alias /.well-known/acme-challenge/ /home/xxxxxxx/public_html/challenges/
   <Directory /home/xxxxxxx/public_html/challenges/>
      AllowOverride None
      Require all granted
      Satisfy Any
   </Directory>

   # rest of your config for this server
   # DocumentRoot, ErrorLog, CustomLog...
</VirtualHost>

<VirtualHost _default_:443>
   ServerName www.yoursite.com
   ServerAlias yoursite.com

   SSLEngine On
   SSLCertificateFile "/usr/local/etc/apache/keys/signed.crt"
   SSLCertificateKeyFile "/usr/local/etc/apache/keys/domain.key"
   # CA certificate from https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem
   SSLCertificateChainFile "/usr/local/etc/apache/keys/lets-encrypt-x3-cross-signed.pem"

   # SSL config according to https://bettercrypto.org/static/applied-crypto-hardening.pdf
   SSLProtocol All -SSLv2 -SSLv3
   SSLHonorCipherOrder On
   SSLCompression Off
   Header always add Strict-Transport-Security "max-age=15768000"
   SSLCipherSuite 'EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:+CAMELLIA256:+AES256:+CAMELLIA128:+AES128:+SSLv3:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!PSK:!DSS:!RC4:!SEED:!ECDSA:CAMELLIA256-SHA:AES256-SHA:CAMELLIA128-SHA:AES128-SHA'
   BrowserMatch "MSIE [2-6]" nokeepalive ssl-unclean-shutdown downgrade-1.0 force-response-1.0
   BrowserMatch "MSIE [17-9]" ssl-unclean-shutdown

   # rest of your SSL/TLS config
   # DocumentRoot, ErrorLog, CustomLog...
</VirtualHost>
```
##### Forward http request to https:
##### Add following line into the prot 80 section in the conf file
```
RewriteRule ^/(.*):80$ https://%{SERVER_NAME}%{REQUEST_URI} [L,R]

```
##### If you have old ssl setting in the 443 part, then delete them first
