# NGINX Plus Ingress Controller deployment - `app3`

## Namespace creation

Let's create the namespace `app3`, where we deploy NGINX Plus Ingress Controller:

```
$ oc create -f ns-app3.yaml 
namespace/app3 created
```

## Allow to get image

The policy list needs to be updated to allow to get the `nginx-ingress-plus` image from the `nginx-ingress` project:

```
$ oc policy add-role-to-group system:image-puller system:serviceaccounts:app3 --namespace=nginx-ingress
Warning: Group 'system:serviceaccounts:app3' not found
clusterrole.rbac.authorization.k8s.io/system:image-puller added: "system:serviceaccounts:app3"
```
### Deploy Secret for Default Backend
NGINX Ingress Controller Requires a default backend secret, lets deploy that.

```
oc apply -f default-server-secret.yaml;
```

## Deploy NGINX Plus Ingress Controller

Now everything is ready to deploy NGINX Plus Ingress Controller into namespace `app3`

```
$ oc apply -f nginx-ingress-controller-app3.yaml 
nginxingresscontroller.k8s.nginx.org/nginx-ingress-controller-app3 created
```

Let's verify

```
$ oc apply -f nginx-ingress-controller-app3.yaml 
nginxingresscontroller.k8s.nginx.org/nginx-ingress-controller-app3 created
$ oc get pods -n app3
NAME                                             READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-app3-597bfcf899-pbgjm   1/1     Running   0          41s
$ oc exec nginx-ingress-controller-app3-597bfcf891-abgxy -n app3 -- nginx -v
nginx version: nginx/1.21.5 (nginx-plus-r26)
```

## Deploy & Apply Security Contstraints for CoreDNS.

This allows CoreDNS to bind to ports. 

```
oc apply -f anyuid_bind_scc.yaml;
oc apply -f sa-anyuid-netbind-acc.yaml;
```

## Deploy CoreDNS

```
oc apply -f dns.yaml;
```

## Verify CoreDNS Deployment

```
oc get pods -n app3 | grep 'core'
```

## Deploy Transport Servers

```
oc apply -f ts-dns-tcp.yaml;
oc apply -f ts-dns-udp.yaml;
```

## Verify Deployment of Transport Servers

```
oc get ts -n app3
```

## Deploy Nodeports

```
oc apply -f nodeport-tcp.yaml;
oc apply -f nodeport-udp.yaml;
```

## Verify Deployment of Nodeports

```
oc get svc -n app3 | grep NodePort
```

## Test DNS Queries

Test DNS Over TCP query
```
dig @10.1.1.13 -p 30255 nginx.org +tcp;
```

Test DNS Over UDP query
```
dig @10.1.1.13 -p 30256 nginx.org;
```
