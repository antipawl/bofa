# Bank of America

## Per-app load-balancing PoC

## NGINX Plus Ingress Controller deployment - `app4`

### Namespace creation

Let's create the namespace `app4`, where we deploy NGINX Plus Ingress Controller:

```
$ oc create -f ns-app4.yaml 
namespace/app4 created
```

### Allow to get image

The policy list needs to be updated to allow to get the `nginx-ingress-plus` image from the `nginx-ingress` project:

```
$ oc policy add-role-to-group system:image-puller system:serviceaccounts:app4 --namespace=nginx-ingress
Warning: Group 'system:serviceaccounts:app4' not found
clusterrole.rbac.authorization.k8s.io/system:image-puller added: "system:serviceaccounts:app4"
```

### Deploy Default Secret

```
oc apply -f default-server-secret.yaml;
```

### Deploy NGINX Plus Ingress Controller

Now everything is ready to deploy NGINX Plus Ingress Controller into namespace `app4`

```
$ oc apply -f nginx-ingress-controller-app4.yaml 
nginxingresscontroller.k8s.nginx.org/nginx-ingress-controller-app4 created
```

Let's verify

```
$ oc apply -f nginx-ingress-controller-app4.yaml 
nginxingresscontroller.k8s.nginx.org/nginx-ingress-controller-app4 created
$ oc get pods -n app4
NAME                                             READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-app4-597bfcf899-pbgjm   1/1     Running   0          41s
$ oc exec nginx-ingress-controller-app4-597bfcf891-abgxy -n app4 -- nginx -v
nginx version: nginx/1.21.5 (nginx-plus-r26)
```

### Deploy Endpoints

```
oc apply -f external-endpoint-tcp.yaml
oc apply -f external-endpoint-udp.yaml
```

### Verify Endpoints Deployment

```
oc get endpoints -n app4
```

### Deploy Services For Endpoints
```
oc apply -f external-service-tcp.yaml
oc apply -f external-service-udp.yaml
```

### Verify 'Services For Endpoint's' Deployment

```
oc get svc nginx-org -n app4
```

### Deploy Transport Servers

```
oc apply -f ts-nginx-org-tcp.yaml
oc apply -f ts-nginx-org-udp.yaml
```

### Verify Transport Server Deployment

```
oc get ts -n app4
```

### Deploy Nodeport

```
oc apply -f nodeport-tcp.yaml
oc apply -f nodeport-udp.yaml
```

### Verify Nodeports Deployment

```
oc get svc -n app4 | grep NodePort
```

### Test With DIG

Test dns query with DIG via Ingress Controller L4 load balancing, for both dns over TCP and UDP.
```
dig @10.1.1.13 -p 30355 nginx.org +tcp
dig @10.1.1.13 -p 30356 nginx.org
```


