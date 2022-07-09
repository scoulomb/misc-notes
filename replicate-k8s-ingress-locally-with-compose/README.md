# README.md

Sources:

- https://morioh.com/p/e381c3309ef1
- https://codeburst.io/replicate-kubernetes-ingress-locally-with-docker-compose-2872e650af6b

## 1- Replicate Kubernetes Ingress locally with Docker Compose

### Article

[Article part 1](article-part-1.md)

### Comment

#### Misc

- Note https://github.com/scoulomb/myk8s/blob/master/Services/service_deep_dive.md#note-on-load-balancer-svc which is a nodeport 
- Ingress uses this nodeport as explained here: https://github.com/scoulomb/myk8s/blob/master/Services/service_deep_dive.md#when-using-ingress
- Service targetted by ingress are of type `ClusterIP`. See also [in part 2](./article-part-2.md#kubernetes-ingress-scenario).

- In compose part, `/etc/hosts/` is edited same needed in k8s setup
    - Alternative define a real FQDN

#### Proxy

Check proxy defnintion https://en.wikipedia.org/wiki/Proxy_server
> In computer networking, a proxy server is a server application that acts as an intermediary between a client requesting a resource and the server providing that resource.[1]

We have 2 types of proxies

#### Open 

> An open proxy is a forwarding proxy server that is accessible by any Internet user. In 2008, network security expert Gordon Lyon estimates that "hundreds of thousands" of open proxies are operated on the Internet.
We have
> - Anonymous proxy – This server reveals its identity as a proxy server but does not disclose the originating IP address of the client. Although this type of server can be discovered easily, it can be beneficial for some users as it hides the originating IP address.
> - Transparent proxy – This server not only identifies itself as a proxy server but with the support of HTTP header fields such as X-Forwarded-For, the originating IP address can be retrieved as well. The main benefit of using this type of server is its ability to cache a website for faster retrieval.

#### Reverse

> A reverse proxy (or surrogate) is a proxy server that appears to clients to be an ordinary server. Reverse proxies forward requests to one or more ordinary servers that handle the request. The response from the proxy server is returned as if it came directly from the original server, leaving the client with no knowledge of the original server.[5] Reverse proxies are installed in the neighborhood of one or more web servers. All traffic coming from the Internet and with a destination of one of the neighborhood's web servers goes through the proxy server. The use of reverse originates in its counterpart forward proxy since the reverse proxy sits closer to the web server and serves only a restricted set of websites. There are several reasons for installing reverse proxy servers:
> - Encryption/SSL acceleration,
> - Load balancing: the reverse proxy can distribute the load to several web servers, each web server serving its own application area. In such a case, the reverse proxy may need to rewrite the URLs in each web page (translation from externally known URLs to the internal locations), 
> - Serve/cache static content-
> -  Spoon feeding,
> - Security,
> -  Extranet publishing

#### Comment

##### F5 (reverse)

I consider F5 big LB LTM as a reverse proxy: https://support.f5.com/csp/article/K01713506

>  BIG-IP system is a full proxy that can be deployed to provide basic and advanced application network services, including load balancing, web performance optimization, application delivery firewall, and secure remote access. It can also act as a reverse proxy.
> - Configure the BIG-IP System as a reverse proxy server by performing the following steps:
> - Create a pool with pool members. (You can create members when you add them to the pool.)
> - Assign an appropriate service specific health monitor to the pool. (BIG-IP LTM comes with a number of popular pre-built monitors.) 
> - Create a virtual server, assign an IP address to the virtual server and attach the pool created in step one.

See also [TLS complements](../tls/tls-certificate.md#complements).

#### Ingress (reverse)

https://kubernetes.io/docs/concepts/services-networking/ingress/

> The Ingress spec has all the information needed to configure a load balancer or proxy server. Most importantly, it contains a list of rules matched against all incoming requests. Ingress resource only supports rules for directing HTTP(S) traffic.

But ingress can also be seen as a reverse proxy.

####  jwilder/nginx-proxy is a reverse proxy

#### Apache rewrite URL (reverse)

https://github.com/open-denon-heos/remote-control#deploy-apache-2-via-docker-recommended


<!--
We explain some compose network capbilities here
In simple config port forwarding for apache and use service name between Apache and other container, but in our case had host netowrk, note we can define network in compose file, https://docs.docker.com/compose/networking/#specify-custom-networks
--> 

<!-- TNZ proxy are reverse proxy -->

#### VPN (open)

VPN is more an open proxy: https://www.numerama.com/tech/645358-quelle-est-la-difference-entre-un-proxy-et-un-vpn.html

See:
- [NAS setup](../NAS-setup/README.md#nas-setup)
- https://github.com/scoulomb/myPublicCloud/blob/main/Azure/Networking/basic.md

<!-- when going in QNAP exposed ip is the one from home and can access local network -->

#### SQUID proxy, firewall (open)

> Squid is a caching and forwarding HTTP web proxy. It has a wide variety of uses, including speeding up a web server by caching repeated requests, caching web, DNS and other computer network lookups for a group of people sharing network resources, and aiding security by filtering traffic. Although primarily used for HTTP and FTP, Squid includes limited support for several other protocols including Internet Gopher, SSL,[7] TLS and HTTPS. Squid does not support the SOCKS protocol, unlike Privoxy, with which Squid can be used in order to provide SOCKS support. 

SQUID can do annymous + transparent proxy

SQUID can use WCCP: https://fr.wikipedia.org/wiki/Web_Cache_Communication_Protocol for caching: https://wiki.squid-cache.org/Features/Wccp2

An open proxy server can be used a firewall: https://waytolearnx.com/2018/09/difference-entre-proxy-et-firewall.html

A firewall can offer SNAT features.
https://docs.microsoft.com/en-us/azure/firewall/snat-private-range

<!-- private_script/tree/main/Links-mig-auto-cloud#outbound-links -->


### Equivalent to what we had seen in 

https://github.com/scoulomb/myDNS/blob/master/2-advanced-bind/5-real-own-dns-application/6-use-linux-nameserver-part-f.md

## 2- TLS Replicate Kubernetes Ingress locally with Docker Compose

[Article part 2](article-part-2.md)

### DNS and ingress

This is exactly what we did for ingress here: https://github.com/scoulomb/myDNS/blob/master/2-advanced-bind/5-real-own-dns-application/6-use-linux-nameserver-part-g.md

And we went further with real CA: https://github.com/scoulomb/myDNS/blob/master/2-advanced-bind/5-real-own-dns-application/6-use-linux-nameserver-part-h.md


### TLS complements and IN learning

And it is linked to [TLS complements](../tls/tls-certificate.md#complements).
In particular in [IN learning](../tls/in-learning-complement/learning-ssl-tld.md#acquire-a-webserver-certificate-using-openssl), we see we reference in Apache virtual server config:
- SSLCertificateFile /cert/www.fakesite.local.crt # certificate
- SSLCertificateKeyFile /cert/www.fakesitelocal.key  # private key of specific cert

Those are exactly equivalent to what we have in Docker [volume in article](./article-part-2.md#extend-docker-compose-config-for-https)

### Self-signed 

About self-signed [certificate generation in article](article-part-2.md#generate-self-signed-certificates), see comment here in [IN learning](../tls/in-learning-complement/learning-ssl-tld.md#acquire-a-webserver-certificate-using-openssl)
> Note we could also create a webserver certificate which is self-signed (creation would be similar to CA certificate)
and  here in [IN learning](../tls/in-learning-complement/learning-ssl-tld.md#link-with-cert-experiements-with-python-nodeport-and-k8s-ingresses).

<!-- view cert: openssl x509 -in appa.prd.coulombel.it.crt -text -->

### SNI

Note Kubernetes ingress controller can support SNI extension:
https://kubernetes.io/docs/concepts/services-networking/ingress/#tls

Ingress `spec.tls[i].hosts[j]` section must match a `spec.rules[m].host` section (convert yaml to json to see structure).
In [article ingress specific example](article-part-2.md#kubernetes-ingress-scenario) example we have the 2 hosts mapped to same certificate (so SNI feature is actually not used here), and no FQDN defined in cert but it could point to a wildcard or SAN certificate (where SAN could have wildcard). 

Quoting [article ingress specific example](article-part-2.md#kubernetes-ingress-scenario) 
> We assume here that the ssl certificate is a wildcard one for *.test.com, else you need to have multiple secrets.

As suggested in [multidomain appendix](../tls/multidomain-appendix.md#also-note-cn-is-deprecated).


We could also have used SNI on top.

However the jwilder nginx proxy uses SNI behind the scene to work, see https://dimuthukasunwp.github.io/Articles/Hosting-multiple-sites-or-applications-using-Docker-and-NGINX-reverse-proxy-with-Letsencrypt-SSL.html
> The ability to serve content from different domains using different certificates from one host is possible thanks to SNI

If we use a wildcard or SAN certificate (where SAN could have wildcard), we can duplicate the cert [with the correct naming convention](article-part-2.md#extend-docker-compose-config-for-https). This is what is suggested in the artcile.

> What the jwilder/nginx-proxy image needs it that the certificates are named like the `VIRTUAL_HOST` entries. 

But actually jwilder also support wildcard or san certificate (where SAN could have wildcard).
Quoting doc: https://github.com/nginx-proxy/nginx-proxy

> **Wildcard Certificates**:
> Wildcard certificates and keys should be named after the domain name with a .crt and .key extension. For example VIRTUAL_HOST=foo.bar.com would use cert name bar.com.crt and bar.com.key.
SNI

> **SNI**: If your certificate(s) supports multiple domain names, you can start a container with CERT_NAME=<name> to identify the certificate to be used. For example, a certificate for *.foo.com and *.bar.com could be named shared.crt and shared.key. A container running with VIRTUAL_HOST=foo.bar.com and CERT_NAME=shared will then use this shared cert.

SAN name can probaly also be a wildcard.

In doc they call it **SNI**, because it is a SNI feature to manage a **SAN** cert.

It would avoid cert duplication also with Compose.

Thus we can combine SNI and SAN.

<!-- so can exploit wildcard or SAN certificate (where SAN could have wildcard)  unlike what is shown in artcile OK CLEAR STOP YES -->

### OCSP

We had seen OCSP in [IN course](../tls/in-learning-complement/learning-ssl-tld.md#pki-components)
It is also supported here: https://github.com/nginx-proxy/nginx-proxy#ocsp-stapling
<!-- stop ocsp and do not explore more readme of proxy, not also https://github.com/nginx-proxy/nginx-proxy#virtual-ports but from proxy to docker app, why  -v /var/run/docker.sock:/tmp/docker.sock:ro, as to run docker in docker osef, network part osef OK -->

<!--
https://towardsdev.com/3-ways-to-add-a-caption-to-an-image-using-markdown-f2ca30562be6
-->
