# Bank of America

## Per-app load-balancing PoC

## NGINX Plus Ingress Controller deployment - `app5`

### Namespace creation

Let's create the namespace `app5`, where we deploy NGINX Plus Ingress Controller:

```
$ oc create -f ns-app5.yaml 
namespace/app3 created
```

### Allow to get image

The policy list needs to be updated to allow to get the `nginx-ingress-plus` image from the `nginx-ingress` project:

```
$ oc policy add-role-to-group system:image-puller system:serviceaccounts:app3 --namespace=nginx-ingress
Warning: Group 'system:serviceaccounts:app5' not found
clusterrole.rbac.authorization.k8s.io/system:image-puller added: "system:serviceaccounts:app5"
```

### Deploy Default Secret

```
oc apply -f default-server-secret.yaml;
```

### Deploy NGINX Plus Ingress Controller

Now everything is ready to deploy NGINX Plus Ingress Controller into namespace `app5`

```
$ oc apply -f nginx-ingress-controller-app5.yaml 
nginxingresscontroller.k8s.nginx.org/nginx-ingress-controller-app5 created
```

Let's verify

```
$ oc apply -f nginx-ingress-controller-app5.yaml 
nginxingresscontroller.k8s.nginx.org/nginx-ingress-controller-app5 created
$ oc get pods -n app5
NAME                                             READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-app5-597bfcf899-pbgjm   1/1     Running   0          41s
$ oc exec nginx-ingress-controller-app2-597bfcf891-abgxy -n app3 -- nginx -v
nginx version: nginx/1.21.5 (nginx-plus-r26)
```

### Deploy Nodeports

Deploy Nodeports for both TCP and UDP.

```
oc apply -f nodeport-tcp.yaml;
oc apply -f nodeport-udp.yaml;
```

### Verify Nodeport Deployment 
```
oc get svc -n app5 | grep NodePort
```

### Test With DIG

Test dns query with DIG via Ingress Controller L4 load balancing, for both dns over TCP and UDP.
```
dig @10.1.1.13 -p 30455 nginx.org +tcp
dig @10.1.1.13 -p 30456 nginx.org
```