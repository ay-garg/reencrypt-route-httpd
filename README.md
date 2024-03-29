## Create a private key for self-signed CA
```
# openssl genrsa -out RootCA.key 2048
```

## Create a CSR for self-signed CA with CN=customCA
```
# openssl req -new -key RootCA.key -out RootCA.csr -subj "/CN=customCA"
```

## You can set the CN (Common Name) as per your requirement

## Create an extension file to use while obtaining the CA cert with CA:true extension
```
# cat extension-file 
keyUsage               = critical,digitalSignature,keyEncipherment,keyCertSign
basicConstraints       = critical,CA:true
```

## Obtain the CA cert (it will be used as dest-ca-cert for route)
```
# openssl x509 -req -days 1460 -in RootCA.csr -signkey RootCA.key -out RootCA.crt -sha256 -extfile extension-file
```

## Create a private key for the certificate which will be used by the application
```
# openssl genrsa -out ayush.key 2048
```

## Create a CSR for the application certificate
``` 
# openssl req -new -key ayush.key -out ayush.csr -subj "/CN=httpd-rencrypt.ayush.com"
```

## The CN set here is same to the route hostname that will be created

## Obtain the application certificate
```
# openssl x509 -req -in ayush.csr -CA RootCA.crt -CAkey RootCA.key -CAcreateserial -out ayush.crt
```

## Create index.html
```
# echo "Hello" >> index.html
```

## Here's the Dockerfile
```
# Centos base images
FROM centos:centos7

# Update currently installed package and install httpd, mod_ssl, ca-certificates packages
RUN yum -y update && yum -y install \
    httpd \
    mod_ssl \
    ca-certificates

# Copy the index.html file
COPY index.html /var/www/html/

# Copy the certificate which will be used by the application
COPY ayush.crt /etc/pki/tls/certs/localhost.crt

# Copy the key which will be used by the application
COPY ayush.key /etc/pki/tls/private/localhost.key

# Copy the CA certificate
COPY RootCA.crt /usr/local/share/ca-certificates/RootCA.crt

RUN chmod 644 /usr/local/share/ca-certificates/RootCA.crt

RUN chmod 0710 /etc/pki/tls/private/localhost.key 

COPY RootCA.crt /etc/pki/ca-trust/source/anchors/

EXPOSE 443

CMD ["httpd", "-D", "FOREGROUND"]
```

## This pod will run with root user, so anyuid scc needs to be added to default serviceaccount before running the pod
```
# oc adm policy add-scc-to-user anyuid -z default
```

## Create the application with the docker image
```
# oc new-app --name httpd --image=docker-image-url
```

## curl the pod IP
```
# oc get po -o wide | grep -i httpd
httpd-1-5lqtc   1/1       Running   0          1m        10.130.3.112   xyz-node-0.com   <none>

# curl -kvv https://10.130.3.112
* About to connect() to 10.130.3.112 port 443 (#0)
*   Trying 10.130.3.112...
* Connected to 10.130.3.112 (10.130.3.112) port 443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
* skipping SSL peer certificate verification
* SSL connection using TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
* Server certificate:
* 	subject: CN=httpd-rencrypt.ayush.com
* 	start date: Nov 19 05:41:16 2019 GMT
* 	expire date: Dec 19 05:41:16 2019 GMT
* 	common name: httpd-rencrypt.ayush.com
* 	issuer: CN=customCA
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 10.130.3.112
> Accept: */*
> 
< HTTP/1.1 200 OK
< Date: Tue, 19 Nov 2019 05:53:22 GMT
< Server: Apache/2.4.37 (centos) OpenSSL/1.1.1
< Last-Modified: Tue, 19 Nov 2019 05:47:45 GMT
< ETag: "6-597ac9a73c240"
< Accept-Ranges: bytes
< Content-Length: 6
< Content-Type: text/html; charset=UTF-8
< 
Hello
* Connection #0 to host 10.130.3.112 left intact
```

## curl the cluster IP
```
# oc get svc
NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
httpd     ClusterIP   172.30.195.216   <none>        443/TCP   5m

# curl -kvv https://172.30.195.216
* About to connect() to 172.30.195.216 port 443 (#0)
*   Trying 172.30.195.216...
* Connected to 172.30.195.216 (172.30.195.216) port 443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
* skipping SSL peer certificate verification
* SSL connection using TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
* Server certificate:
* 	subject: CN=httpd-rencrypt.ayush.com
* 	start date: Nov 19 05:41:16 2019 GMT
* 	expire date: Dec 19 05:41:16 2019 GMT
* 	common name: httpd-rencrypt.ayush.com
* 	issuer: CN=customCA
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 172.30.195.216
> Accept: */*
> 
< HTTP/1.1 200 OK
< Date: Tue, 19 Nov 2019 05:56:25 GMT
< Server: Apache/2.4.37 (centos) OpenSSL/1.1.1
< Last-Modified: Tue, 19 Nov 2019 05:47:45 GMT
< ETag: "6-597ac9a73c240"
< Accept-Ranges: bytes
< Content-Length: 6
< Content-Type: text/html; charset=UTF-8
< 
Hello
* Connection #0 to host 172.30.195.216 left intact
```

## Create a reencrypt route
```
# oc create route reencrypt httpd --service=httpd --hostname=httpd-rencrypt.ayush.com --dest-ca-cert=RootCA.crt
```

## curl the route
```
# oc get route
NAME      HOST/PORT                  PATH      SERVICES   PORT      TERMINATION   WILDCARD
httpd     httpd-rencrypt.ayush.com             httpd      443-tcp   reencrypt     None

# curl -kvv https://httpd-rencrypt.ayush.com
* About to connect() to httpd-rencrypt.ayush.com port 443 (#0)
*   Trying 10.74.249.97...
* Connected to httpd-rencrypt.ayush.com (10.74.249.97) port 443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
* skipping SSL peer certificate verification
* SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* Server certificate:
* 	subject: CN=*.cloudapps.ayush.com
* 	start date: Nov 04 19:45:34 2019 GMT
* 	expire date: Nov 03 19:45:35 2021 GMT
* 	common name: *.cloudapps.ayush.com
* 	issuer: CN=rootCA
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: httpd-rencrypt.ayush.com
> Accept: */*
> 
< HTTP/1.1 200 OK
< Date: Tue, 19 Nov 2019 06:08:35 GMT
< Server: Apache/2.4.37 (centos) OpenSSL/1.1.1
< Last-Modified: Tue, 19 Nov 2019 05:47:45 GMT
< ETag: "6-597ac9a73c240"
< Accept-Ranges: bytes
< Content-Length: 6
< Content-Type: text/html; charset=UTF-8
< Set-Cookie: 0112d3b91eee77fe327ded7f932d7f1d=fac56a2b62d5ec9a227c0e573df9927c; path=/; HttpOnly; Secure
< Cache-control: private
< 
Hello
* Connection #0 to host httpd-rencrypt.ayush.com left intact
```
