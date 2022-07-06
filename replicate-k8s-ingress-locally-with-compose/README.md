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

This is exactly what we did for ingress here: https://github.com/scoulomb/myDNS/blob/master/2-advanced-bind/5-real-own-dns-application/6-use-linux-nameserver-part-g.md

And we went further with real CA: https://github.com/scoulomb/myDNS/blob/master/2-advanced-bind/5-real-own-dns-application/6-use-linux-nameserver-part-h.md


And it is linked to [TLS complements](../tls/tls-certificate.md#complements).

<!--
https://towardsdev.com/3-ways-to-add-a-caption-to-an-image-using-markdown-f2ca30562be6
-->
