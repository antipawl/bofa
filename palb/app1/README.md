
# Per-app load-balancing PoC

# NGINX Plus Ingress Controller deployment - `app1`

## Namespace creation

Create the namespace `app1`, where we deploy NGINX Plus Ingress Controller:

```
oc create -f ns-app1.yaml 
```

Response:

namespace/app1 created


## Allow to get image

The policy list needs to be updated to allow to get the `nginx-ingress-plus` image from the `nginx-ingress` project:

```
oc policy add-role-to-group system:image-puller system:serviceaccounts:app1 --namespace=nginx-ingress
```
Response:

Warning: Group 'system:serviceaccounts:app1' not found
clusterrole.rbac.authorization.k8s.io/system:image-puller added: "system:serviceaccounts:app1"


## Deploy TLS certificate and key
```
oc deploy -f default-server-secret.yaml
```

## Deploy NGINX Plus Ingress Controller

Now everything is ready to deploy NGINX Plus Ingress Controller into namespace `app1`

```
oc apply -f nginx-ingress-controller-app1.yaml 
```

Response:
nginxingresscontroller.k8s.nginx.org/nginx-ingress-controller-app1 created

Let's verify

```
$ oc get pods -n app1
```
Response:
| NAME                                            | READY | STATUS    | RESTARTS | AGE |
|-------------------------------------------------|-------|-----------|----------|-----|
| nginx-ingress-controller-app1-6c78864c45-6q2rb  | 1/1   | Running   | 0        | 42s |



```
oc exec nginx-ingress-controller-app1-6c78864c45-6q2rb -n app1 -- nginx -v
```
Response

nginx version: nginx/1.21.5 (nginx-plus-r26)


Deploy the `bank.example.com` application.


```
oc apply -f bank.yam
```

Response:

deployment.apps/credit created

service/credit-svc created

deployment.apps/debit created

service/debit-svc created

```
oc apply -f bank-secret.yaml
```

Response:

secret/bank-secret created


```
oc apply -f bank-ingress.yaml
```
Response:

ingress.networking.k8s.io/bank-ingress created

```
oc apply -f node-port-bank.yaml
```

Response:

service/nginx-ingress-app1 created

Check therunning pods in the `app1` namespace:
```
oc get pods -n app1
```

Response:

| NAME                                            | READY | STATUS    | RESTARTS | AGE |
|-------------------------------------------------|-------|-----------|----------|-----|
| credit-74d96d5df5-6s2vl                         | 1/1   | Running   | 0        | 23s |
| credit-74d96d5df5-xmk89                         | 1/1   | Running   | 0        | 23s |
| debit-745569f555-9jdkh                          | 1/1   | Running   | 0        | 25s |
| debit-745569f555-sgg92                          | 1/1   | Running   | 0        | 25s |
| debit-745569f555-v56c4                          | 1/1   | Running   | 0        | 25s |
| nginx-ingress-controller-app1-6c78864c45-6q2rb  | 1/1   | Running   | 0        | 3h  |
  

Check the services that are running in `app1` namespace:

```
oc get svc -n app1
```

Response:

| NAME                          | TYPE          | CLUSTER-IP    | EXTERNAL-IP | PORT(S)                    | AGE |
|-------------------------------|---------------|---------------|-------------|----------------------------|-----|
| credit-svc                    | ClusterIP     | 192.168.1.190 | <none>      | 80/TCP                     | 26s |
| debit-svc                     | ClusterIP     | 192.168.1.30  | <none>      | 80/TCP                     | 26s |
| nginx-ingress-controller-app1 | NodePort      | 192.168.1.82  | <none>      | 80:31385/TCP,443:30433/TCP | 3h  |
  

The `nginx-ingress-controller-app1` service type is `NodePort`, so let's construct the`NodeIP:NodePort` and access to that pair: for `HTTP` protocol let's use `31385`, for
`HTTPS` - `30433`.  Please keep that in mind for future references.

  
Let's get the list of IP addresses of the cluster nodes:

```
oc get nodes -o wide
```

Response:
  
| NAME                     | STATUS | ROLES | AGE    | VERSION         | INTERNAL-IP | EXTERNAL-IP  |  OS-IMAGE           |KERNEL-VERSION | CONTAINER-RUNTIME |
|-------------------------|--------|--------|--------|-----------------|-------------|--------------|---------------------|---------------|-------------------|
| master-1.ocp.f5-udf.com | Ready  | master | 2d22h  | v1.20.0+c8905da | 10.1.1.10   | <none>       | RHEL CoreOS (Ootpa) | SNIP!         |   SNIP!           |
| master-2.ocp.f5-udf.com | Ready  | master | 2d22h  | v1.20.0+c8905da | 10.1.1.11   | <none>       | RHEL CoreOS (Ootpa) | SNIP!         |   SNIP!           |
| master-3.ocp.f5-udf.com | Ready  | master | 2d22h  | v1.20.0+c8905da | 10.1.1.12   | <none>       | RHEL CoreOS (Ootpa) | SNIP!         |   SNIP!           |
| worker-1.ocp.f5-udf.com | Ready  | master | 2d22h  | v1.20.0+c8905da | 10.1.1.13   | <none>       | RHEL CoreOS (Ootpa) | SNIP!         |   SNIP!           |
| worker-2.ocp.f5-udf.com | Ready  | master | 2d22h  | v1.20.0+c8905da | 10.1.1.14   | <none>       | RHEL CoreOS (Ootpa) | SNIP!         |   SNIP!           |
| worker-3.ocp.f5-udf.com | Ready  | master | 2d22h  | v1.20.0+c8905da | 10.1.1.15   | <none>       | RHEL CoreOS (Ootpa) | SNIP!         |   SNIP!           |
  
  Take a look on the `INTERNAL-IP` column here.  It's possible to use any of those addresses
(`10.1.1.10` - `10.1.1.15`) to connect to the `bank.example.com` application.
  
  The NGINX Plus Ingress Contoller running inside the OpenShift cluster in `app1` namespace
expects to reply for a request with the `Host` header with the `bank.example.com` value.
Also, there are two different contexts are defined: `/credit` and `/debit`.
  
```
curl -v -H "Host: bank.example.com" http://10.1.1.13:31385/credit
```

  Response:
  
*   Trying 10.1.1.13:31385...
* TCP_NODELAY set
* Connected to 10.1.1.13 (10.1.1.13) port 31385 (#0)
> GET /credit HTTP/1.1
> Host: bank.example.com
> User-Agent: curl/7.68.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 301 Moved Permanently
< Server: nginx/1.21.5
< Date: Wed, 08 Jun 2022 18:37:38 GMT
< Content-Type: text/html
< Content-Length: 169
< Connection: keep-alive
< Location: https://bank.example.com:443/credit
< 
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx/1.21.5</center>
</body>
</html>


The response was expected, the NGINX Plus Ingress Controller redirects the request to use secure `HTTPS` protocol instead of plain `HTTP`.  
  
Do another request and see the response, so use the same IP address `10.1.1.13` and another `30443` port to get access to the application.
  
```
curl -v -k -H "Host: bank.example.com" https://10.1.1.13:30433/credit
```

*   Trying 10.1.1.13:30433...
* TCP_NODELAY set
* Connected to 10.1.1.13 (10.1.1.13) port 30433 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
* ALPN, server accepted to use http/1.1
* Server certificate:
*  subject: O=NGINX Inc; CN=example.com
*  start date: Jun  7 18:47:11 2022 GMT
*  expire date: Jun  7 18:47:11 2023 GMT
*  issuer: O=NGINX Inc; CN=example.com
*  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
  
> GET /credit HTTP/1.1
> Host: bank.example.com
> User-Agent: curl/7.68.0
> Accept: */*
> 
  
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.21.5
< Date: Wed, 08 Jun 2022 18:00:42 GMT
< Content-Type: text/plain
< Content-Length: 162
< Connection: keep-alive
< Expires: Wed, 08 Jun 2022 18:00:41 GMT
< Cache-Control: no-cache
< 
  
  
Server address: 10.244.10.36:8080
        
Server name: credit-74d96d5df5-6s2vl
        
Date: 08/Jun/2022:18:00:42 +0000
        
URI: /credit
        
Request ID: d5d8686e63a2babedca1bbe350bf7013
        
* Connection #0 to host 10.1.1.13 left intact

