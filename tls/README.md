# README - TLS certificates


## TLS Basic

- [TLS certificate](./tls-certificate.md)

## TLS deep-dive

TLS in details

### Cloudfare blogpost

- [Cloudfare blogpost](./cloudfare.md) as mentioned [TLS certificate](./tls-certificate.md#man-in-the-middle-attach-and-need-of-a-ca)


### tls13.ulfheim.net

#### Post

- https://tls.ulfheim.net/ and https://github.com/scoulomb/illustrated-tls

#### Note 1: Ephemeral Diffie-Helman


In  https://tls.ulfheim.net/, the pre-master key is not sent to the client (unlike nominal case in Cloudfare blogpost and [TLS certificate](./tls-certificate.md))

This the second case in cloud fare blogpost

From [Cloudfare blogpost](./cloudfare.md)

> All TLS handshakes make use of asymmetric encryption (the public and private key), but not all will use the private key in the process of generating session keys. For instance, an ephemeral Diffie-Hellman handshake proceeds as follows:

In diffie-hellman Shared encryption key is computed by using

- Client (PreMasterSecret encryption key calaculation) via
    - server random (from Server Hello)
    - client random (from Client Hello)
    - server public key (from Server Key Exchange)
    - client private key (from Client Key Generation) 
- Server  (MasterSecret encryption key calaculation which is equal to PreMasterSecret) via
    - server random (from Server Hello)
    - client random (from Client Hello)
    - client public key (from Client Key Exchange)
    - server private key (from Server Key Generation) 

We use: https://en.wikipedia.org/wiki/Elliptic-curve_Diffie%E2%80%93Hellman to reach same secret (PreMasterSecret == MasterSecret)


From master and premaster key we compute on both side the same key data

- client MAC key
- server MAC key
- client write key
- server write key
- client write IV
- server write IV

#### Note 2: SNI data

Note when we perform the client hello we exchange SNI information 

> Extension - Server Name
The client has provided the name of the server it is contacting, also known as SNI (Server Name Indication).
Without this extension a HTTPS server would not be able to provide service for multiple hostnames on a single IP address (virtual hosts) because it couldn't know which hostname's certificate to send until after the TLS session was negotiated and the HTTP request was made.-  00 00 - assigned value for extension "server name"
>-    00 18 - 0x18 (24) bytes of "server name" extension data follows
> -    00 16 - 0x16 (22) bytes of first (and only) list entry follows
> -    00 - list entry is type 0x00 "DNS hostname"
> -    00 13 - 0x13 (19) bytes of hostname follows
> -    65 78 61 ... 6e 65 74 - "example.ulfheim.net" 


## SAN vs SNI

### SAN and SNI

https://serverfault.com/questions/807959/what-is-the-difference-between-san-and-sni-ssl-certificates


> SAN (Subject Alternative Name) is part of the X509 certificate spec, where the certificate has a field with a list of alternative names that are also valid for the subject (in addition to the single Common Name / CN). This field and wildcard names are essentially the two ways of using one certificate for multiple names.

> SNI (Server Name Indication) is a TLS protocol extension that is sort of a TLS protocol equivalent of the HTTP Host-header. When a client sends this, it allows the server to pick the proper certificate to present to the client without having the limitation of using separate IP addresses on the server side (much like how the HTTP Host header is heavily used for plain HTTP).

> Do note that SNI is not something that is reflected in the certificate and it actually achieves kind of the opposite of what the question asks for; it simplifies having many certificates, not using one certificate for many things.

> On the other hand, it depends heavily on the situation which path is actually preferable. As an example, what the question asks for is almost assuredly not what you actually want if you need certificates for different entities.

### F5 - TLS termination 

We can setup SNI certificate on F5.

From https://support.f5.com/csp/article/K13452

> With the introduction of TLS SNI, the client that supports TLS SNI can indicate the name of the server to which the client is attempting to connect, in the ClientHello packet, during the SSL handshake process. The server that supports TLS SNI can use this information to select the appropriate SSL certificate to return to the client in the ServerHello packet during the SSL handshake. As a result, the client can establish secure connections to the secure website from the list of multiple secure websites that are hosted on a single virtual server.

Also there is an interesting usage of F5 irule

> Note that there is no automatic mechanism which allows the BIG-IP system to select SSL profile based on Server Name value received in the client SSL Hello message. However, with additional help of an iRule you can force selection of proper ServerSSL profile based on the host-name header value received in initial HTTP request from the client.


