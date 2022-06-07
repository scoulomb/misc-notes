# Leanring SSL TLS 

This complement on top of [TLS certificate](../tls-certificate.md).

## Part 1

### PKI hierarchy

https://www.linkedin.com/learning/learning-ssl-tls/pki-hierarchy?autoSkip=true&autoplay=true&resume=false&u=75507506

#### PKI components

- Certificate Authority (CA): issues, renews, revokes certificate, maintain CRL, offline
- Rgistration Authority (RA): subordinate CA, used to manage certificatees
- Certificate Revocation List (CRL) or Online Certificate Status Protocol (OCSP): verifiy certificate validity using s/n
- Ceritifcate template: Blueprint used when using certificate
- Certificate: contains template name, signature of CA, expiry info, pub/priv key

We can have Single-Tier PKI hierarchy (top is CA) or multi-tier PKI hierarchy (we have RA).
Multi-tier avoids to have CA online.

#### PKI cert

We should not say SSL or TLS vcert but PKI cert that can be used by SSL/TLS.
We can say PKI, digital cert, security or x509.

#### CRL 

Certificate revocation: 
- CRL (download full list maintained by CA) or OCSP (query soft component, query for a specific certificate).
- OCSP has OCSP stapling, certificate owner (webserver) check is own status.
Browser will receive OCSP stapling. Avoid client to query

#### Public Key Pinning (PKP)

There is also concept of public key pinning (similar to known host)

https://www.linkedin.com/learning/learning-ssl-tls/certificate-lifecycle-management?autoSkip=true&autoplay=true&resume=false&u=75507506


CSR is in PKCS#10 format

See also [multidomain](../multidomain.md).

<!-- Learning part 1 ok -->

## Part 2: PKI CA implementation

We will install a PKI Certificate Authority (CA) on Microsoft Windows Server, Linux.
This root CA relies on Root certificate.

See [cert hierarchy](../tls-certificate.md#more-details-on-sections-and-certificate-hierarchy).


[For all cases below we setup our own CA (not a well know CA like Digicert or Comodo) so it means client needs to trust this private CA.](../tls-certificate.md#private-ca).

Whether we have a private or public CA. CA are self-signed.

See https://docs.oracle.com/cd/E19509-01/820-3503/ggbgc/index.html

We will also see how to setup a subordinate CA (in AWS) of a root CA (Windows).


### MS AD CS certificate authority (Step 1)

We install a PKI Certificate Authority (CA) on Microsoft Windows Server.
CA has common name (FakeDomain1 in video).

From there certifcate requestor can request certificate signed by this authority.

### MS AD CS certificate template (Step 2)

We can have template for certificate request.
And can customize it.
We need to issue the certificate template to be available for use.

From this template we can create certificate

### Linux OpenSSL PKI environment

Similalrly we can also have a CA authority in linux.
We start by genrating private key for the CA (on windows we had same step or could reuse exisiting one)

````
openssl genrsa -aes256 -out CAprivate.key 2048
````

Then we generate the root Certificate with that authority private key.

````
openssl req -new -x509 -key CAprivate.key -sha256 -day 365 -out FakeDomain2CA.pem 
````

This require CA private key passphrase

More details [here, create-root-ca-done-once](./self-signed-certificate-with-custom-ca.md#create-root-ca-done-once)

### Configure AWS subordinate CA

https://www.linkedin.com/learning/learning-ssl-tls/configure-an-aws-certificate-manager-subordinate-ca?autoSkip=true&autoplay=true&resume=false&u=75507506

We will define a subordinate CA of private CA we defined in [MS AD CS certificate authority (Step 1)](#ms-ad-cs-certificate-authority-step-1).

 - Go to `AWS certificate manager` > `private certificate CA` where we a single option to define a subordinate CA. 
- AWS provides a CSR for the CA. 
- In Windows Server certificate management (certsrv) UI, go to: > `Request a certificate` > `advanced` > `submit a certificate request`
Using subordinate CA template.
- Copy the CSR provided by AWS
- And we can download the certificate (`.cer` file) and the chain.
Certificate is digitally signed by the CA.
- Then in AWS we can import the certificate and we need to import the chain.
- The subordinate CA is then active.
- In windows CA, in issued certifcate we can see the certicate we issued for the subordinate CA.

==> This solution is hybrid with CA on premise and subordinate in the cloud.

This is compliant with 
See [cert hierarchy](../tls-certificate.md#more-details-on-sections-and-certificate-hierarchy) where here we did not sign a user certificate but a subordinate CA certificate.


## Part 3: PKI certticate acqusition

(user, webserver certificate)

https://www.linkedin.com/learning/learning-ssl-tls/ssl-vs-tls?autoplay=true&resume=false&u=75507506


### SSL vs TLS

- both use PKI certificates
- Used for
    - encryption 
    - digital signature
- SSL is deprecaed, even SSLv3, superseded by TLS (introduced in 1999)
- We can have security protocol downgrade attacks, so disable SSL (Poodle attacks)
- SSL known vulnerability is Heartbleed bug
- We also have SSL VPN
- TLS VPN are different as they use TCP/443 (firewall friendly)
- Security protocol can be configured on client side

#### Acquire a webserver certificate using Microsoft AD CS

**GUI certsrv**

we can see templates -> custom webserver
Certif template -> rigth click manage and see details.

**GUI certlm - `manage conmpute certificate` (also avail in win10)** to acquire PKI cert (can be done on any machine who jonied domain and not only)

We request new cert from here, we template `custom web server`
We provide subject name (common name) and SAN (important cf. domain mismatch). Some field can be mandatory and enforced template.
Then click enroll.
At the end FakeDomain CA has issued webserver certificate.

We can also acquire certificate via Web UI (`<cert authority url>/certsrv`) or automation.

It is also store in cert store (certlm).

#### Acquire a **client** certificate using Microsoft AD CS

We configure on server side a template for client certificate as we did [above](#ms-ad-cs-certificate-template-step-2).


**GUI cert srv**

We have a template for client/server authentification (computer template).
Certif template and rigth click manage to see details.
In example they delete the existing computer templat to reset ite and re-create one (new certificate template to issue, computer)

**MMC console (also avaialble in Windows 10)**

- `File > add a new snap in > cert > computer account cert`
- `Certificates > Personal > request new cert > we have computer cert > enroll`
We can see the issued certficate.

We can pay attention to intended purpose which is client/serv auth.

It is also available in Windows 10.
Here we acquire a cert for our machine's user(where MMC is run) from a CA.

<!-- can see well in corp laptop -->

#### Acquire a **webserver** certificate using OpenSSL

For instance to run Apache server.

From [Linux OpenSSL pki env](#linux-openssl-pki-environment) we have :
- CApprivate.key
- FakeDomain2CA.pem

**Step 1**: We will generate pub/priv key pair to be used for that specific

````
openssl genrsa -aes255 -out www.fakesitelcal.key 2048
````

This will create `www.fakesitelocal.key`.

**Step 2**: Then we will generate the CSR (Certificate Signing Request)

`````
openssl req -new -key www.faksesitelocal.key -out www.fakesite.local.csr
`````

This requuires passphrase of `www.fakesitelocal.key`.

Extra attributes are also requested.

This will create the CSR `www.fakesite.local.csr` and also `otherinfo.ext`.

In `otherinfo.ext` we have the alt name (filled in CN)

**Step 3**: Then we create the server certificate

````
openssl x509 -req -in wwww.fakesite.local.csr -CA FakeDomain2CA.pem -CAKey CAprivate.key -CAcreateserial -extfile otherinfo.ext -out www.fakesite.local.crt -day 365 -sha256
````

This requires passphrase of `CAPrivate.key`.

Output is `www.fakesite.local.crt`

**Step 4**: Finally we configure Apache we have 

````
<VirtualHost *:443>
    SSLEngine on
    SSLCertificateFile /cert/www.fakesite.local.crt # certificate
    SSLCertificateKeyFile /cert/www.fakesitelocal.key  # private key of specific cert
    SSLCACertificateFile /cert/FakeDomain2CA.pem # root certificate

````

We restart Apache, it requires passphrase (certificate private key)

In browser we can see it is trusted.
Because we import in browser SSLCACertificateFile as again it is our own private CA.

Here certificate is signed with our private CA (not a public CA or public CA delegation).
Note we could also create a webserver certificate which is self-signed (creation would be similar to CA certificate)

See 3 cases here: https://github.com/scoulomb/myDNS/blob/master/2-advanced-bind/5-real-own-dns-application/6-use-linux-nameserver-part-g.md#step-2-configure-the-server-to-use-https


What we described here is fully aligned with [create-root-ca-done-once](./self-signed-certificate-with-custom-ca.md#create-a-certificate-done-for-each-server)


Note that this directive `SSLCACertificateFile` is used for client auth.
Quoting: https://httpd.apache.org/docs/2.4/en/mod/mod_ssl.html
> This directive sets the all-in-one file where you can assemble the Certificates of Certification Authorities (CA) whose clients you deal with. These are used for Client Authentication. Such a file is simply the concatenation of the various PEM-encoded Certificate files, in order of preference. This can be used alternatively and/or additionally to SSLCACertificatePath.

So it means that if we acquire a client certificate like we did in this section with [windows](#acquire-a-client-certificate-using-microsoft-ad-cs),
We would trust client as the the CA is included in `SSLCACertificateFile`.

See also here an example of Apache config:
https://github.com/open-denon-heos/remote-control/blob/main/apache-setup/heos.conf


### Acquire a webserver certificate using AWS certificate manager


We will acquire a cert from subordinate C+
A (subordinate of Windows CA) in AWS certtificate manager.

> `certicate amanager` > can see FakeCOSubCA

> `certicate amanager` > `get started` > `Request private certificate` (we already imported) (public would mean it is signed by Amazon public CA) > `Select the sub CA` we created previously> we enter domnain name > `request`

We can see the issued certificate and can export the certificate.



### Link with cert experiements with Python (NodePort) and k8s ingresses

- Self-signed: https://github.com/scoulomb/myDNS/blob/master/2-advanced-bind/5-real-own-dns-application/6-use-linux-nameserver-part-g.md (not key generation `-newkey` in OpenSSL command)
- public (sub)CA:  https://github.com/scoulomb/myDNS/blob/master/2-advanced-bind/5-real-own-dns-application/6-use-linux-nameserver-part-h.md

We could also offload cert in loab balancer or ESB.

<!-- ok, links with links cloud/move tls term -->

### Acquire a code signing certificate

They are use to sign digitally software (Authority could be appstore or organization).


**GUI cert srv**

We have a template for client/server authentification. No code sign-in template is issued (not visible)
Certif template and rigth click manage to see details. We will modify existing code sign-in template, make change and call it `custom sign in`.
The rigth clik on cert template and issue the template.

**GUI certlm - `manage conmpute certificate` (also avail in win10)** to 

In `Personal > Certificates > request`, we can see custom code sign in  `X` status, because it can only be issue to a user and not an app.
It is not a computer type cert but user cert.

**MMC console (also avaialble in Windows 10)**

- `File > add a new snap in > cert > My user account cert`
We can see cert for this user
- `Certificates > Personal > request new cert > we have custom code sign in cert > add more info > enroll`
We can see the issued certficate.

We can pay attention to intended purpose which is code sign-in.

It is also available in Windows 10.
Here we acquire a cert for our machine's user (where MMC is run) from a CA.

We verify signature using public  key.

## Part 4: PKI certificate usage

<!-- all above is clear and OK -->