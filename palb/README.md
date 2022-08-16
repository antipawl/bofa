# Bank of America

## Per-app load-balancing PoC

The repository contains the Bank of America use cases.


## NGINX Ingress Operator deployment

### Pre-requisites

OpenShift 4.10 cluster installed, here's the documentation, https://access.redhat.com/documentation/en-us/openshift_container_platform/4.10

### Approve CSRs after (re)starting the cluster

Once the cluster is up, usually it takes 15 minutes, please check pending CSRs:

```
$ oc get csr | grep Pending
```

and approve those:

```
$ oc get csr | grep Pending | awk '{print $1}' | xargs oc adm certificate approve
```

Internal registry is provisioned, please follow the procedure described here,
https://access.redhat.com/documentation/en-us/openshift_container_platform/4.10/html/registry/setting-up-and-configuring-the-registry#configuring-registry-storage-baremetal

#### Change the image registry’s management state

```
$ oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'
config.imageregistry.operator.openshift.io/cluster patched
```

#### Configure storage for the image registry in non-production clusters

```
$ oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
config.imageregistry.operator.openshift.io/cluster patched
```

#### Deploy Manifests

The following manifests need to be deployed before the NGINX Plus Ingress Operator.

```
$ oc apply -f k8s/scc.yaml
```


### Deployment

Use OpenShift UI to install the NGINX Ingress Operator.
Login to the OpenShift Administrator account, usually `kube-admin`, navigate to `OperatorHub`,
search for NGINX and find NGINX Ingress Operator, click install.  By default, the
NGINX Ingress Operator installs into the `nginx-ingress` namespace.

Validate installation of the NGINX Ingress Operator, installed into the
`nginx-ingress` namespace:

```
$ oc get pods -n nginx-ingress
NAME                                                         READY   STATUS    RESTARTS   AGE
nginx-ingress-operator-controller-manager-74ccfdc4d4-8xmmm   2/2     Running   0          11m
```

## Obtain `nginx-ingress-plus` image

### Pre-requisites

Obtain a JSON Web Token, also knows as JWT, from the [MyF5 Portal](https://my.f5.com)
and create `regcred` secret inside the `nginx-ingress` project:

```
$ oc project nginx-ingress
$ oc create secret docker-registry regcred --docker-server="private-registry.nginx.com" \
  --docker-username="<insert your JWT here>" --docker-password=none
secret/regcred created
```

### Import `nginx-ingress-plus` image to the OpenShift registry

Pre-compiled `nginx-ingress-plus` image is available to download from the
[NGINX Private Registry](https://private-registry.nginx.com).
Another method is to build that image from the source code.  The source
code is available [here](https://github.com/nginxinc/kubernetes-ingress).

Let's import the `nginx-ingress-plus` image from the private registry.

```
$ oc import-image nginx/nginx-ingress-plus:2.3.0-ubi --from=private-registry.nginx.com/nginx-ic/nginx-plus-ingress:2.3.0-ubi \
> --confirm imagestream.image.openshift.io/nginx-ingress-plus imported
imagestream.image.openshift.io/nginx-ingress-plus imported

Name:                   nginx-ingress-plus
Namespace:              default
Created:                Less than a second ago
Labels:                 <none>
Annotations:            openshift.io/image.dockerRepositoryCheck=2022-07-14T00:21:07Z
Image Repository:       image-registry.openshift-image-registry.svc:5000/default/nginx-ingress-plus
Image Lookup:           local=false
Unique Images:          1
Tags:                   1

2.3.0-ubi
  tagged from private-registry.nginx.com/nginx-ic/nginx-plus-ingress:2.3.0-ubi

  * private-registry.nginx.com/nginx-ic/nginx-plus-ingress@sha256:c782f91eabfc4330734261f9098f1e82f6833d1edd0b7eab137eb771356b7322
      Less than a second ago

Image Name:     nginx-ingress-plus:2.3.0-ubi
Docker Image:   private-registry.nginx.com/nginx-ic/nginx-plus-ingress@sha256:c782f91eabfc4330734261f9098f1e82f6833d1edd0b7eab137eb771356b7322
Name:           sha256:c782f91eabfc4330734261f9098f1e82f6833d1edd0b7eab137eb771356b7322
Created:        Less than a second ago
Annotations:    image.openshift.io/dockerLayersOrder=ascending
Image Size:     112.1MB in 10 layers
Layers:         78.36MB sha256:0c673eb68f88b60abc0cba5ef8ddb9c256eaf627bfd49eb7e09a2369bb2e5db0
                1.78kB  sha256:028bdc977650c08fcf7a2bb4a7abefaead71ff8a84a55ed5940b6dbc7e466045
                7.653MB sha256:0be25904b477ffc97ffd164fba055cfc310388cc17882c86554f16a6e6c70ace
                3.955kB sha256:ca6b5364a2f70f4b8f272a04fce7c13f4a9d8427bc4e5e1bbf23c934a44cbf13
                10.76MB sha256:5b46651d92c029fe479e75a87202cf82432b4a27a3148edfc7854f33e3b9809f
                5.099kB sha256:f52b4ea583c9ad388b19d6a8fac0019fadecb62c4d64ef20764f20b661bc47cc
                32B     sha256:4f4fb700ef54461cfa02571ae0db9a0dc1e0cdb5577484a6d75e68dc38e8acc1
                32B     sha256:4f4fb700ef54461cfa02571ae0db9a0dc1e0cdb5577484a6d75e68dc38e8acc1
                1.697MB sha256:abb99c56bfa44d910d23547cca5644b5ac4571d29797b1c875c4a3bd950e1022
                13.61MB sha256:15b7aab376b9c3226873628777b21a042ae1f11321ec58aa7b9a75752db332ee
Image Created:  38 hours ago
Author:         <none>
Arch:           amd64
Entrypoint:     /nginx-ingress
Working Dir:    <none>
User:           101
Exposes Ports:  443/tcp, 80/tcp
Docker Labels:  architecture=x86_64
                build-date=2022-06-17T04:26:28.499070
                com.redhat.build-host=cpt-1006.osbs.prod.upshift.rdu2.redhat.com
                com.redhat.component=ubi8-container
                com.redhat.license_terms=https://www.redhat.com/en/about/red-hat-end-user-license-agreements#UBI
                description=The Ingress Controller is an application that runs in a cluster and configures an HTTP load balancer according to Ingress resources.
                distribution-scope=public
                io.k8s.description=The NGINX Ingress Controller is an application that runs in a cluster and configures an HTTP load balancer according to Ingress resources.
                io.k8s.display-name=Red Hat Universal Base Image 8
                io.openshift.expose-services=
                io.openshift.tags=nginx,ingress-controller,ingress,controller,kubernetes,openshift
                maintainer=kubernetes@nginx.com
                name=NGINX Ingress Controller
                org.nginx.kic.image.build.nginx.version=R27
                org.nginx.kic.image.build.os=ubi-plus
                org.nginx.kic.image.build.target=linux/amd64
                org.nginx.kic.image.build.version=goreleaser
                org.opencontainers.image.created=2022-07-12T10:35:36.566Z
                org.opencontainers.image.description=NGINX Plus Ingress Controller for Kubernetes
                org.opencontainers.image.documentation=https://docs.nginx.com/nginx-ingress-controller
                org.opencontainers.image.licenses=Apache-2.0
                org.opencontainers.image.revision=979db22d8065b22fedb410c9b9c5875cf0a6dc66
                org.opencontainers.image.source=https://github.com/nginxinc/kubernetes-ingress
                org.opencontainers.image.title=kubernetes-ingress
                org.opencontainers.image.url=https://github.com/nginxinc/kubernetes-ingress
                org.opencontainers.image.vendor=NGINX Inc <kubernetes@nginx.com>
                org.opencontainers.image.version=2.3.0-ubi
                release=1
                summary=The Ingress Controller is an application that runs in a cluster and configures an HTTP load balancer according to Ingress resources.
                url=https://access.redhat.com/containers/#/registry.access.redhat.com/ubi8/images/8.6-855
                vcs-ref=f1ee6e37554363ec55e0035aba1a693d3627fdeb
                vcs-type=git
                vendor=NGINX Inc
                version=v2.3.0-ubi
Environment:    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
                container=oci
```

So, the image has been imported and it's available as the following:
```
image-registry.openshift-image-registry.svc:5000/nginx-ingress/nginx-ingress-plus
```
inside the OpenShift cluster.


# Profit!
