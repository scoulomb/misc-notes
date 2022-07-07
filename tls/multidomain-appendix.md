# Multidomain appendix: SAN, SNI, wildcard

## Solution 1: Wildcard

Use wildcard certificate in CN [Data.Subject field](https://docs.oracle.com/cd/E19424-01/820-4811/6ng8i26ao/index.html).

## Solution 2: SAN certificate

SAN is `Data.subjectAltName` [field](https://fr.wikipedia.org/wiki/Subject_Alternative_Name).

### What are SAN certs

From  https://www.digicert.com/faq/subject-alternative-name.htm

> The Subject Alternative Name field lets you specify additional host names (sites, IP addresses, common names, etc.) to be protected by a single SSL Certificate, such as a Multi-Domain (SAN) or Extend Validation Multi-Domain Certificate.

> The Subject Alternative Name extension was a part of the X509 certificate standard before 1999, but it wasn't until the launch of Microsoft Exchange Server 2007 that it was commonly used; this change makes good use of Subject Alternative Names by simplifying server configurations. Now Subject Alternative Names are widely used for environments or platforms that need to secure multiple sites (names) across different domains/subdomains.

### Also note CN is deprecated

From https://en.wikipedia.org/wiki/Subject_Alternative_Name

> RFC 2818 (May 2000) specifies Subject Alternative Names as the preferred method of adding DNS names to certificates, deprecating the previous method of putting DNS names in the commonName field. Google Chrome version 58 (March 2017) removed support for checking the commonName field at all, instead only looking at the SANs.

From https://developer.chrome.com/blog/chrome-58-deprecations/#remove_support_for_commonname_matching_in_certificates

> RFC 2818 describes two methods to match a domain name against a certificate: using the available names within the subjectAlternativeName extension, or, in the absence of a SAN extension, falling back to the commonName. The fallback to the commonName was deprecated in RFC 2818 (published in 2000), but support remains in a number of TLS clients, often incorrectly.

From https://datatracker.ietf.org/doc/html/rfc2818

> If a **subjectAltName** extension of type dNSName is present, that MUST
be used as the identity. Otherwise, the (most specific) **Common Name
field in the Subject field of the certificate** MUST be used. Although
the use of the Common Name is existing practice, it is deprecated and
Certification Authorities are encouraged to use the dNSName instead.

We can have in SAN:
- also a wildcard 
- same top domain to be more specific 

We can also have different domain

See part 1, certificate at (3'16) https://www.linkedin.com/learning/learning-ssl-tls/certificates?autoplay=true&resume=false&u=75507506 

## Solution 3: SNI 

### SAN vs SNI

https://serverfault.com/questions/807959/what-is-the-difference-between-san-and-sni-ssl-certificates


> SAN (Subject Alternative Name) is part of the X509 certificate spec, where the certificate has a field with a list of alternative names that are also valid for the subject (in addition to the single Common Name / CN). This field and wildcard names are essentially the two ways of using one certificate for multiple names.

> SNI (Server Name Indication) is a TLS protocol extension that is sort of a TLS protocol equivalent of the HTTP Host-header. When a client sends this, it allows the server to pick the proper certificate to present to the client without having the limitation of using separate IP addresses on the server side (much like how the HTTP Host header is heavily used for plain HTTP).

> Do note that SNI is not something that is reflected in the certificate and it actually achieves kind of the opposite of what the question asks for; it simplifies having many certificates, not using one certificate for many things.

> On the other hand, it depends heavily on the situation which path is actually preferable. As an example, what the question asks for is almost assuredly not what you actually want if you need certificates for different entities.


Note when we perform the client hello we exchange SNI information.
See https://tls12.ulfheim.net/

> Extension - Server Name
The client has provided the name of the server it is contacting, also known as SNI (Server Name Indication).
Without this extension a HTTPS server would not be able to provide service for multiple hostnames on a single IP address (virtual hosts) because it couldn't know which hostname's certificate to send until after the TLS session was negotiated and the HTTP request was made.-  00 00 - assigned value for extension "server name"
>-    00 18 - 0x18 (24) bytes of "server name" extension data follows
> -    00 16 - 0x16 (22) bytes of first (and only) list entry follows
> -    00 - list entry is type 0x00 "DNS hostname"
> -    00 13 - 0x13 (19) bytes of hostname follows
> -    65 78 61 ... 6e 65 74 - "example.ulfheim.net" 

We can combine SNI and SAN: https://discussions.citrix.com/topic/320041-san-certificate-with-sni-should-it-work/
> The NetScaler appliance now supports SNI with a SAN extension certificate. During handshake initiation, the host name provided by the client is first compared to the common name and then to the subject alternative name. If the name matches, the corresponding certificate is presented to the client.

### F5 and SNI - TLS termination 

We can setup SNI certificate on F5.

From https://support.f5.com/csp/article/K13452

> With the introduction of TLS SNI, the client that supports TLS SNI can indicate the name of the server to which the client is attempting to connect, in the ClientHello packet, during the SSL handshake process. The server that supports TLS SNI can use this information to select the appropriate SSL certificate to return to the client in the ServerHello packet during the SSL handshake. As a result, the client can establish secure connections to the secure website from the list of multiple secure websites that are hosted on a single virtual server.

Also there is an interesting usage of F5 irule

> Note that there is no automatic mechanism which allows the BIG-IP system to select SSL profile based on Server Name value received in the client SSL Hello message. However, with additional help of an iRule you can force selection of proper ServerSSL profile based on the host-name header value received in initial HTTP request from the client.


<!-- related to links cloudif and add SAN field in ls-->