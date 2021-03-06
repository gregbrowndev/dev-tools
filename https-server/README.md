# HTTPS Server Configuration

## Overview

This project deploys a simple Nginx proxy + Apache web server configuration over HTTPS only. The frontend Nginx server
is configured to redirect HTTP requests to HTTPS. The Apache web server hosts a simple index.html.

To enable HTTPS locally, a self-signed certificate must be installed and configured on the Nginx server. We must then
trust this certificate locally to connect to Nginx securely over HTTPS via our web browser.

Recommend reading this article on [OpenSSL essentials](https://www.digitalocean.com/community/tutorials/openssl-essentials-working-with-ssl-certificates-private-keys-and-csrs). 
OpenSSL is a commandline program which can be used to create certificate signing requests (CSRs) and SSL certificates.

## Create Root SSL Certificate Authority (CA)

We can create a root certificate which can be used to sign other 'domain' certificates. Then by trusting the root
certificate on a given platform also trusts the certificates we created with the root certificate.

Create RSA-2048 key:

```bash
openssl genrsa -des3 -out rootCA.key 2048
```

Create Root SSL Certificate:

```bash
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 365 -out rootCA.pem
```

Here:

* `-x509` tells `req` to create a self-signed certificate
* `-new` indicates an CSR is being generated
* `-nodes` indicates the private key should _not_ be encrypted with a pass phrase
* `-key` option specifies an existing private key
* `-sha256` signs the CSR with SHA-2
* `-days` specifies the number of days the certificate is valid
* `-out` specifies the output name of the certificate

Enter information in prompt. This is known as a Distinguished Name:

```commandline
Country Name (2 letter code) [XX]:GB
State or Province Name (full name) []:Cambridgeshire
Locality Name (eg, city) [Default City]:Cambridge
Organization Name (eg, company) [Default Company Ltd]: My Company
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:Local Certificate
Email Address []:
```

We could also create the certificate with a single command via:

```bash
openssl req \
       -newkey rsa:2048 -nodes -keyout rootCA.key \
       -x509 -days 365 -out rootCA.pem
```

Note:

* `-newkey rsa:2048` creates a new 2048-bit key generated by the RSA algorithm
* The private key will not be passphrase protected

## Trust Root CA

Highly recommend reading [this article](https://www.bounca.org/tutorials/install_root_certificate.html) which covers how to trust the Root CA
on many platforms, including Chrome, Firefox, Linux, iOS, etc.

In Chrome:

* Go to Settings -> Advanced -> Manage Certificates (or go to `chrome://settings/certificates`)
* Go to Authorities Tab
* Click Import and select `rootCA.pem`
* Select all trust levels

## Create SSL Domain Certificate

Create configuration file `localhost.conf`:

```text
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[dn]
C=US
ST=RandomState
L=RandomCity
O=RandomOrganization
OU=RandomOrganizationUnit
emailAddress=hello@example.com
CN = localhost
```

Importantly, it specifies `subjectAltName` (SAN). Since January 2017 Chrome has dropped support for commonName as a fallback.

Create domain SSL certificate signing request:

```bash
openssl req -new -sha256 -nodes -config localhost.conf -out localhost.csr -newkey rsa:2048 -keyout localhost.key
```

Create `localhost.ext`:

```text
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
```

Create certificate using our Root CA:

```bash
openssl x509 -req -in localhost.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out localhost.crt -days 500 -sha256 -extfile localhost.ext
```

## Install Certificates to Server

This step depends on the server environment which is handling the HTTPS connections, e.g. Nginx, Apache, Node.js.

### Nginx

For an Nginx reverse proxy, `nginx.conf` may look like:

```text
server {
	listen 80;
	server_name example.com;
    return 301 https://$host$request_uri/;
}


server {
    listen 443 default ssl;

    ssl_certificate /etc/ssl/certs/server.crt;
    ssl_certificate_key /etc/ssl/private/server.key;

   location / {
		proxy_pass         http://web;
		proxy_redirect     off;
		proxy_set_header   Host $host;
		proxy_set_header   X-Real-IP $remote_addr;
		proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header   X-Forwarded-Host $server_name;
	}
}
```

### HAProxy

HAProxy requires the certificate and key to be bundled together. This
is achievable simply by:

```bash
cat localhost.crt localhost.key > localhost.pem
```

Install the certificate on the server at `/etc/ssl/private/server.pem`.

Then `haproxy.cfg` will look something like:

```text
global
    daemon
    maxconn 2048

defaults
    mode http
    option forwardfor
    option http-server-close

frontend proxy
    bind *:80
    bind *:443 ssl crt /etc/ssl/private/server.pem
    redirect scheme https if !{ ssl_fc }
    reqadd X-Forwarded-Proto:\ https
    default_backend webservers

backend webservers
    server app1 django:5000 maxconn 32
```