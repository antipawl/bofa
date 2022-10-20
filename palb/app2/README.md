# Per-app load-balancing PoC

# NGINX Plus Ingress Controller deployment - `app2`

## Namespace creation

Let's create the namespace `app2`, where we deploy NGINX Plus Ingress Controller:

```
$ cd app2
$ oc create -f ns-app2.yaml 
namespace/app2 created
```

## Allow to get image

The policy list needs to be updated to allow to get the `nginx-ingress-plus` image from the `nginx-ingress` project:

```
$ oc policy add-role-to-group system:image-puller system:serviceaccounts:app2 --namespace=nginx-ingress
Warning: Group 'system:serviceaccounts:app2' not found
clusterrole.rbac.authorization.k8s.io/system:image-puller added: "system:serviceaccounts:app2"
```

## Deploy default server certificate and key

```
$ oc deploy -f default-server-secret.yaml
```

## Deploy NGINX Plus Ingress Controller

Now everything is ready to deploy NGINX Plus Ingress Controller into namespace `app2`

```
$ oc apply -f nginx-ingress-controller-app2.yaml 
nginxingresscontroller.k8s.nginx.org/nginx-ingress-controller-app2 created
```

Let's verify

```
$ oc apply -f nginx-ingress-controller-app2.yaml 
nginxingresscontroller.k8s.nginx.org/nginx-ingress-controller-app2 created

$ oc get pods -n app2
NAME                                             READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-app2-597bfcf899-pbgjm   1/1     Running   0          41s
$ oc exec nginx-ingress-controller-app2-597bfcf899-pbgjm -n app2 -- nginx -v
nginx version: nginx/1.21.5 (nginx-plus-r26)
```


In case the NGINX Plus Ingress Controller has been deployed without such `ConfigMap`, it
needs to be redeployed.

Verify the `ConfigMap` is in place and it's not empty:

```
$ oc describe cm app2-nginx-ingress
Name:         app2-nginx-ingress
Namespace:    app2
Labels:       app.kubernetes.io/instance=app2
              app.kubernetes.io/managed-by=Helm
              app.kubernetes.io/name=app2-nginx-ingress
              helm.sh/chart=nginx-ingress-0.13.0
Annotations:  meta.helm.sh/release-name: app2
              meta.helm.sh/release-namespace: app2

Data
====
resolver-addresses:
----
8.8.8.8

BinaryData
====

Events:
  Type    Reason   Age    From                      Message
  ----    ------   ----   ----                      -------
  Normal  Updated  9m23s  nginx-ingress-controller  Configuration from app2/app2-nginx-ingress was updated
```

Deploy additional manifests:

```
$ oc apply -f external-service-app2.yaml
service/nginx-org created
$ oc apply -f external-ingress-app2.yaml
ingress.networking.k8s.io/nginx-org-ingress created
$ oc apply -f node-port-nginx-org.yaml
service/nginx-ingress-app2 created
```

Let's verify:

```
$ oc get svc
NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-ingress-app2   NodePort       192.168.1.55   <none>        80:30341/TCP   5m40s
nginx-org            ExternalName   <none>         nginx.org     <none>         9m15s
```

Construct the `NodeIP:NodePort` pair and send a request:

```
$ curl -v -H "Host: nginx.org" http://10.1.1.13:30341/
*   Trying 10.1.1.13:30341...
* TCP_NODELAY set
* Connected to 10.1.1.13 (10.1.1.13) port 30341 (#0)
> GET / HTTP/1.1
> Host: nginx.org
> User-Agent: curl/7.68.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.21.5
< Date: Wed, 13 Jul 2022 22:12:36 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 8167
< Connection: keep-alive
< Last-Modified: Thu, 02 Jul 2022 15:03:58 GMT
< ETag: "6298d15e-1fe7"
< Accept-Ranges: bytes
< 
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html><head><meta http-equiv="Content-Type" content="text/html; charset=utf-8"><link rel="alternate" type="application/rss+xml" title="nginx news" href="http://nginx.org/index.rss"><title>nginx news</title><style type="text/css">body { background: white; color: black; font-family: sans-serif; line-height: 1.4em; text-align: center; margin: 0; padding: 0; } #banner { background: black; color: #F2F2F2; line-height: 1.2em; padding: .3em 0; box-shadow: 0 5px 10px black; } #banner a { color: #00B140; } #main { text-align: left; margin: 0 auto; min-width: 32em; max-width: 64em; } #menu { float: right; width: 11em; padding: 0 .5em 1em .5em; border-left: 2px solid #DDD; } #content { margin-right: 13.5em; padding: 0 .2em 0 1.5em; } h1 { display: block; font-size: 3em; text-align: left; height: .7em; margin: 0; margin-bottom: .5em; } h1 img { width: 100%; } h2 { text-align: center; } p { text-align: justify; } table.news p { margin-top: 0; } table.news td { vertical-align: baseline; } table.news .date { text-align: right; padding-right: 0.5em; white-space: nowrap; } table.donors td { vertical-align: baseline; } table.donors li { text-align: left; } div.directive { background: #F2F2F2; line-height: 1em; margin: 1em 0 1em -1em; padding: .7em .7em .7em 1em; border-top: 2px solid #DDD; } div.directive th { padding-left: 0; padding-right: .5em; vertical-align: baseline; text-align: left; font-weight: normal; } div.directive td { vertical-align: baseline; } div.directive pre { padding: 0; margin: 0; } div.directive p { margin: .5em 0 0 .1em; font-size: .8em; } a.notrans { color: gray; text-decoration:none; } span.initial { font-size: 200%; float: left; padding-right: 10pt;} ul, ol { margin: .5em 0 1em 1em; padding: 0 .5em; } ol { list-style-position: inside; } li { text-align: justify; padding: .5em 0 0 1px; } .compact li { padding-top: 0; } dl { margin: .5em 0 1em 0; } dt { margin: .5em 0; } .compact dt { margin-bottom: .2em; } dd { margin-left: 1.5em; padding-left: 1px; text-align: justify; } td.list { background: #F2F2F2; } blockquote { margin: 1em 0 1em 1em; padding: .5em; } li blockquote, dd blockquote { margin: .7em 0; } blockquote.note { border: 1px dotted #999; line-height: 1.2em; text-align: justify; } blockquote.example { line-height: 1em; border-left: 1px solid #BBB; } blockquote.example pre { padding: 0; margin: 0; } sup { font-size: 50%; } .video { position: relative; padding-bottom: 56.25%; overflow: hidden; } .video iframe, .video object, .video embed { position: absolute; top:0; left:0; width:100%; height:100%; }</style><script>
        window.addEventListener("load", function(e) {
            fetch("banner/banner.html")
                .then((response) => response.text())
                .then((resp) => {
                    document.getElementById("banner").innerHTML = resp;
                })
                .catch((error) => {
                    console.warn(error);
                });
        });
    </script><script>
        (function(w, d, s, l, i) {
            w[l] = w[l] || [];
            w[l].push({
                'gtm.start': new Date().getTime(),
                event: 'gtm.js'
            });
            var f = d.getElementsByTagName(s)[0],
                j = d.createElement(s),
                dl = l != 'dataLayer' ? '&l=' + l : '';
            j.async = true;
            j.src = '//www.googletagmanager.com/gtm.js?id=' + i + dl;
            f.parentNode.insertBefore(j, f);
        })(window, document, 'script', 'dataLayer', 'GTM-TPSP33');
    </script></head><body><div id="banner"></div><div id="main"><div id="menu"><h1><a href="/"><img src="/nginx.png" alt="nginx"></a></h1><div>english<br><a href="ru/">русский</a><br><br>news<br><a href="2021.html">2021</a><br><a href="2020.html">2020</a><br><a href="2019.html">2019</a><br><a href="2018.html">2018</a><br><a href="2017.html">2017</a><br><a href="2016.html">2016</a><br><a href="2015.html">2015</a><br><a href="2014.html">2014</a><br><a href="2013.html">2013</a><br><a href="2012.html">2012</a><br><a href="2011.html">2011</a><br><a href="2010.html">2010</a><br><a href="2009.html">2009</a><br><br><a href="en/">about</a><br><a href="en/download.html">download</a><br><a href="en/security_advisories.html">security</a><br><a href="en/docs/">documentation</a><br><a href="en/docs/faq.html">faq</a><br><a href="en/books.html">books</a><br><a href="en/support.html">support</a><br><br><a href="http://trac.nginx.org/nginx">trac</a><br><a href="http://twitter.com/nginxorg">twitter</a><br><a href="https://www.nginx.com/blog/">blog</a><br><br><a href="https://unit.nginx.org/">unit</a><br><a href="en/docs/njs/">njs</a><br></div></div><div id="content"><h2>nginx news</h2>
            <table class="news">
        <tr><td class="date"><a name="2022-06-02"></a>2022-06-02</td><td><p><a href="https://unit.nginx.org/">unit-1.27.0</a> version has been
<a href="https://unit.nginx.org/news/2022/unit-1.27.0-released/">released</a>.
</p></td></tr><tr><td class="date"><a name="2022-05-24"></a>2022-05-24</td><td><p><a href="en/docs/njs/index.html">njs-0.7.4</a>
version has been
<a href="en/docs/njs/changes.html#njs0.7.4">released</a>,
featuring extended directives for
<a href="en/docs/njs/reference.html#ngx_fetch">Fetch</a> API:
<a href="en/docs/http/ngx_http_js_module.html#js_fetch_timeout">js_fetch_timeout</a>,
<a href="en/docs/http/ngx_http_js_module.html#js_fetch_verify">js_fetch_verify</a>,
<a href="en/docs/http/ngx_http_js_module.html#js_fetch_buffer_size">js_fetch_buffer_size</a>,
<a href="en/docs/http/ngx_http_js_module.html#js_fetch_max_response_buffer_size">js_fetch_max_response_buffer_size</a>.
</p></td></tr><tr><td class="date"><a name="2022-05-24"></a>2022-05-24</td><td><p><a href="en/download.html">nginx-1.22.0</a>
stable version has been released,
incorporating new features and bug fixes from the 1.21.x mainline branch  — 
including
hardening against potential requests smuggling
and cross-protocol attacks,
<a href="en/docs/stream/ngx_stream_ssl_module.html#ssl_alpn">ALPN
support</a> in the stream module,
better distribution of connections among worker processes on Linux,
support for the PCRE2 library,
support for OpenSSL 3.0 and <code>SSL_sendfile()</code>,
improved
<a href="en/docs/http/ngx_http_core_module.html#sendfile">sendfile</a>
handling on FreeBSD,
the
<a href="en/docs/http/ngx_http_mp4_module.html#mp4_start_key_frame">
mp4_start_key_frame</a> directive,
and more.
</p></td></tr><tr><td class="date"><a name="2022-04-12"></a>2022-04-12</td><td><p><a href="en/docs/njs/index.html">njs-0.7.3</a>
version has been
<a href="en/docs/njs/changes.html#njs0.7.3">released</a>.
</p></td></tr><tr><td class="date"><a name="2022-01-25"></a>2022-01-25</td><td><p><a href="en/download.html">nginx-1.21.6</a>
mainline version has been released.
</p></td></tr><tr><td class="date"><a name="2022-01-25"></a>2022-01-25</td><td><p><a href="en/docs/njs/index.html">njs-0.7.2</a>
version has been
<a href="en/docs/njs/changes.html#njs0.7.2">released</a>.
</p></td></tr><tr><td class="date"><a name="2021-12-28"></a>2021-12-28</td><td><p><a href="en/docs/njs/index.html">njs-0.7.1</a>
version has been
<a href="en/docs/njs/changes.html#njs0.7.1">released</a>.
</p></td></tr><tr><td class="date"><a name="2021-12-28"></a>2021-12-28</td><td><p><a href="en/download.html">nginx-1.21.5</a>
mainline version has been released.
</p></td></tr><tr><td class="date"><a name="2021-12-02"></a>2021-12-02</td><td><p><a href="https://unit.nginx.org/">unit-1.26.1</a> bugfix version has been
<a href="https://mailman.nginx.org/pipermail/unit/2021-December/000292.html">released</a>.
</p></td></tr><tr><td class="date"><a name="2021-11-18"></a>2021-11-18</td><td><p><a href="https://unit.nginx.org/">unit-1.26.0</a> version has been
<a href="https://mailman.nginx.org/pipermail/unit/2021-November/000288.html">released</a>,
featuring multiple improvements in static content serving, application-wide PHP
opcache, and a number of bugfixes.
</p></td></tr>
            </table>
        </div></div></body></html>
* Connection #0 to host 10.1.1.13 left intact
```
