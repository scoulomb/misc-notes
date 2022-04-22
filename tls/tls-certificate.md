# TLS explained to myself

## Basic crypto concept

[For the reamaining of this article. Alice is message sender, Bob is the receiver](https://fr.wikipedia.org/wiki/Alice_et_Bob).

### Symetric cryptography

We use same key to encode and decode.

### Asymetric cryptography: we generate two keys, key1 and key2.

When we encode with key 1 we can decode with key 2  and vice versa.
One key is in general private, and other one is public.

Thus definition given [here](https://youtu.be/T4Df5_cojAs?t=128)

2 usages:

1. **Key encryption**: Any message encrypted with Bob's public key can only be decrypted with Bob's private key.
In that case Recipient (Bob) give public key to sender (Alice). 
Sender (Alice) use recipient (Bob) public key to encode message, recipient (Bob) use its private key to decode the message.

2. **Digital Signatures**: Anyone with access to Alice's public key can verify that a message 
could only have been created by someone with access to Alice's private key. (it was encrypted using private key here)

In that case Alice encrypt message using her private key. Bob check tries to decode (digital signature) using Alice public key. 
If he can decode, it proves Alice had encrypted this message.
We will see that this is what a CA does to digitally sign a certificate.

I had read:
It makes no sense to encrypt anything with your private key, 
because the message could then be decrypted by the public key that everybody has access to. 
When somebody wants to send you a secret message, they encrypt it using your public key, and then only you can decrypt it, 
because only you know your private key
-> not the case for signature

Key encryption and digital signature are well in 2 related section of this doc:
https://docs.oracle.com/cd/E19424-01/820-4811/6ng8i26ba/index.html

The pub/priv keys are related :

Quoting : https://docs.neo.org/docs/en-us/basic/concept/cryptography/encryption_algorithm.html

> Elliptic Curve Cryptography (ECC) algorithm is a kind of asymmetric encryption algorithm. With the irreversible feature of K=k*G process (K: public key, G: base point (constant)), it can prevent solving private key from public key by brutal force.


See also Mastering bitcoin book.

## Certificates

### See intro to certificate

Oracle doc: https://docs.oracle.com/cd/E19424-01/820-4811/6ng8i26ao/index.html

### Note cert can be used for 

- client authentification: signature
- Mail encryption (S/MIME certificate): encryption
- SSL/TLS (client and server SSL certificate): certifcates are signed and enables encryption keys exchange
- Signature of software delivery (object signing certifcate): signature
- Identify CA (CA certificates): signature


### Every X.509 certificate consists of the following sections.

- A data section, including the following information.
    - d1: The version number of the X.509 standard supported by the certificate.
    - d2: The certificate’s serial number. Every certificate issued by a CA has a serial number that is unique among the certificates issued by that CA.
    - d3: **Information about the user’s public key, including the algorithm used and a representation of the key itself.**
    - d4: **The DN of the CA that issued the certificate.**
    - d5: **The period during which the certificate is valid** (for example, between 1:00 p.m. on November 15, 2003 and 1:00 p.m. November 15, 2004).
    - d6: **The DN of the certificate subject** (for example, in a client SSL certificate this would be the user’s DN), also called the subject name.
    - d7: Optional certificate extensions, which may provide additional data used by the client or server. For example, the certificate type extension indicates the type of certificate—that is, whether it is a client SSL certificate, a server SSL certificate, a certificate for signing email, and so on. Certificate extensions can also be used for a variety of other purposes.

- A signature section, includes the following information.
    - s1: The cryptographic algorithm, or cipher, used by the issuing CA to create its own digital signature.
    - s2: **The CA’s digital signature, obtained by hashing all of the data in the certificate together and encrypting it with the CA's private key.**



### chain certificate and certificate verification (Server Authentication During SSL Handshake)

Quoting Oracle doc on certificate:
https://docs.oracle.com/cd/E19424-01/820-4811/6ng8i26ao/index.html
Here is how certificate and chain of certificate are verified

![cert chain](media/chn.gif)

> A certificate chain traces a path of certificates from a branch in the hierarchy to the root of the hierarchy. In a certificate chain, the following occur:
> Each certificate is followed by the certificate of its issuer.
>  In Figure 5–4, the Engineering CA certificate contains the DN of the CA (that is, USA CA), that issued that certificate. USA CA’s DN is also the subject name of the next certificate in the chain.
>  Each certificate is signed with the private key of its issuer. The signature can be verified with the public key in the issuer’s certificate, which is the next certificate in the chain.

![cert chain verification](media/chver.gif)

> Verifying a Certificate Chain

> Certificate chain verification is the process of making sure a given certificate chain is well-formed, valid, properly signed, and trustworthy. Directory Server software uses the following steps to form and verify a certificate chain, starting with the certificate being presented for authentication:

> 1. The certificate validity period is checked against the current time provided by the verifier’s system clock. (d5)
> 2. The issuer’s certificate is located (d4). The source can be either the verifier’s local certificate database (on that client or server) or the certificate chain provided by the subject (for example, over an SSL connection).
> 3. The certificate signature is verified using the public key in the issuer certificate.(verify s2 by using d3 in the next certificate in the chain )
> If the issuer’s certificate is trusted by the verifier in the verifier’s certificate database, verification stops successfully here. Otherwise, the issuer’s certificate is checked to make sure it contains the appropriate subordinate CA indication in the Directory Server certificate type extension, and chain verification returns to step 1 to start again, but with this new certificate.

Quoting Oracle Doc on SSL those steps are added for SSL certificate.
As depicted here: https://docs.oracle.com/cd/E19424-01/820-4811/6ng8i26ba/index.html

> 4. Does the domain name in the server’s certificate match the domain name of the server itself?
> This step confirms that the server is actually located at the same network address specified by the domain name in the server certificate. Although step 4 is not technically part of the SSL protocol, it provides the only protection against a form of security attack known as man-in-the-middle. Clients must perform this step and must refuse to authenticate the server or establish a connection if the domain names don’t match. If the server’s actual domain name matches the domain name in the server certificate, the client goes on to the next step.
> 5. The server is authenticated.
The client proceeds with the SSL handshake. If the client doesn’t get to step 5 for any reason, the server identified by the certificate cannot be authenticated, and the user is warned of the problem and informed that an encrypted and authenticated connection cannot be established. 

> If the server requires client authentication, the server performs the steps described in Client Authentication During SSL Handshake.



### Client Authentication During SSL Handshake

This process is similar to server authentification.
We can add 2 steps for that cases between step 4 and 5.

> 5'. Is the user’s certificate listed in the LDAP entry for the user?

> This optional step provides one way for a system administrator to revoke a user’s certificate even if it passes the tests in all the other steps. The Certificate Management System can automatically remove a revoked certificate from the user’s entry in the LDAP directory. All servers that are set up to perform this step then refuses to authenticate that certificate or establish a connection. If the user’s certificate in the directory is identical to the user’s certificate presented in the SSL handshake, the server goes on to the next step.

> 5''. Is the authenticated client authorized to access the requested resources?

> The server checks what resources the client is permitted to access according to the server’s access control lists (ACLs) and establishes a connection with appropriate access. If the server doesn’t get to step 6 for any reason, the user identified by the certificate cannot be authenticated, and the user is not allowed to access any server resources that require authentication.



## Application to TLS

Alice (client) -> Bob (server)

### Symetic and asymetric

#### Symetric cryptography is not sufficient bcecause an attacker could steal chiffer key. This key is exchanged between Alice and Bob.

````
@startuml
title Symetric cryptography

Alice->Bob: exchange key in clear
Alice -> Robber: steal the key
Alice -> Alice : encode message with the key
Alice -> Bob : send message to  bob
Alice -> Robber : steal message 
Robber -> Robber : decode the message
Bob -> Bob : decode message with key
@enduml
````


#### This why asymetric is used.

- Bob sends public key to Alice,
- Alice encrypts message with Bob's public key,
- Bob decrypts message with its private key.

````
@startuml
title ASymetric cryptography

Alice -> Bob : initiate authent
Bob -> Alice : send BOB public key
Alice -> Alice : encode message with BOB public key
Alice -> Bob : send message to  bob
Bob -> Bob : decode message with BOB private key
@enduml
````

Using asymmetric cryptography (public key crypto) has an higher cost than symmetric.
This is the reason why, they use a session key, Alice and Bob exchange this session key in asymmetric way,
and then continue to exchange and continue using this session key with symmetric cryptography.

### Man in the middle attach and need of a CA

**Problem**: The public key can be intercepted and substituted (man in the middle attach)
- Robber intercepts Bob's  public key and replace by Robber key.
- Alice encrypts the message using robber key (believing it is the one of Bob).
- She sends it back to robber (thinking it is bob). 
- Robber decodes Alice message. 
- He reencodes it using Bob's public key (to not arouse suspicions) 
- and send it to bob.

````
@startuml
title ASymetric cryptography and man in the middle attack

actor Alice 
actor Robber 
actor Bob

Alice -> Bob : initiate authent
Bob -> Robber : send BOB public key to Alice but stolen by Robber
Robber -> Robber : save BOB pulic key
Robber -> Alice : Send ROBBER public key instead of bob public key
Alice -> Alice : encode message with ROBBER public key (believing it is the one of Bob)
Alice -> Robber : send message to Robber (believing it is  Bob)
Robber -> Robber: decode message using ROOBER private key. He is happy
Robber -> Robber: encode message using BOB public key (to not make Bob suspicious)
Robber -> Bob: send message to bob
Bob -> Bob : decode message with key wiht BOB private key
@enduml
````

**Solution**: Certificate Authority (CA)

This matches that documentation (and many others): https://docs.oracle.com/cd/E19424-01/820-4811/6ng8i26ba/index.html (Messages Exchanged During SSL Handshake)


````
@startuml
title ASymetric cryptography and man in the middle attack, and certificate authority
    
actor Alice 
actor Robber 
actor Bob
actor CA

== one time request: certificate creation ==

Bob -> Bob: Generate a private key / public key pair (pub determined from priv) 
Bob -> Bob: Fill Certificate Signing Request (CSR). CSR contains public key, CN.
' https://fr.wikipedia.org/wiki/Demande_de_signature_de_certificat
Bob -> Bob: Sign CSR with private key (signature mode)
Bob -> CA : send money + CSR reqest to request CA 
CA -> CA: Validate CSR (CSR signature with public key in cert, DNS owner...)
CA -> CA : sign Bob certificate by encoding it using CA private key (signature mode)
CA -> Bob : provide certificate (fullchain.pem) to Bob which embeds provided pub key and a private key of the certificate (privkey.pem)
' this privkey is not used in SSL handshake, we use gnerated private key, this key could be used to sign with this certificate 

==  SSL handshake exchange ==

Alice -> Bob : (1) send 'client hello' message. It includes which TLS version the client supports, the cipher suites supported, and a string of random bytes known as the "client random."

alt original bob
    Bob -> Alice : (2) send 'server hello' with BOB certificate which contains BOB public key, cipher and "server random". \n If the client is requesting a server resource that requires client authentication, requests the client’s certificate.
    Alice -> Alice : (3) Authentification - Alice (client) can use some of the information sent by the server (Bob) to authenticate the server.(signature) \n We describe this in section "chain certificate and certificate verification (Server Authentication During SSL Handshake)"
    Alice -> Alice: get BOB public key from certificate
    Alice -> Alice : (4) Alice depending on the cipher being used (chosen base on step 1+2),\ncreates the pre-master secret for the session, encrypts it with the (Bob) server’s public key, obtained from (Bob) the server’s certificate, sent in Step 2, and 
    Alice -> Bob : (4) sends pre-master secret to the server.
    Alice -> Bob : (5) If the server has requested client authentication, the client also signs another piece of data that is unique to this handshake and known by both the client and server. In this case the client sends both the signed data and the client’s own certificate to the server along with the encrypted pre-master secret.
    Bob -> Bob: (6) If the server has requested client authentication, the server attempts to authenticate the client. See "Client Authentication During SSL Handshake"
    ' Error in Oracle doc as point to server
    Bob -> Bob: (6) server uses its private key (Bob private key) to decrypt the pre-master secret, then performs a series of steps (which the client also performs, starting from the same pre-master secret) to generate the master secret.
    Alice -> Alice: (6) Generates master secret from pre-master, client and server random
    ' https://www.acunetix.com/blog/articles/establishing-tls-ssl-connection-part-5/' master_secret = PRF(pre_master_secret, "master secret", ClientHello.random + ServerHello.random) [0..47];
    Bob -> Bob: (7) Generate session keys from the client random, the server random, and the premaster secret/master secret
    Alice -> Alice:  (7) Also generate session keys and should reach same result as Bob
    Alice -> Bob: (8) send client is ready with session key (symetric)
    Bob -> Alice: (9) send server is ready with session key (symetric)
    Alice -> Bob: (10) The SSL handshake is now complete, and the SSL session has begun. The client and the server use the session keys to encrypt and decrypt the data they send to each other and to validate its integrity.
else robber
    Bob -> Robber: steal certificate and replace by its own 
    Robber -> Alice: send message with altered certificate 
    Alice -> Alice : Certificate failure
    Alice->Alice: It failed and refuse to talk with Bob
end
@enduml
````


See actual application here: https://github.com/scoulomb/myDNS/blob/master/2-advanced-bind/5-real-own-dns-application/6-use-linux-nameserver-part-h.md


This is known as TLS handshake diagram is aligned with explanation given here:
https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/ (not the Diffie-Hellman)

I mirrored the page [cloudfare.md](cloudfare.md)

Some CA like let's encrypt are free.

[OK here clear, confirm this privkey is not used in SSL handshake, we use generated p
rivate key, this key could be used to sign with this certificate+readme]
Add CRL
windows emplae
can add to trustore cert
CSR for CA itself

## Link with http over socket

When we implemented our own http client here: https://github.com/scoulomb/http-over-socket/blob/main/main.py
This part (tls handshake):

````python
https_context.wrap_socket(raw_socket, server_hostname=connection.hostname)
````

matches this part of the flow explained here!
while the initial connect is shown in [cloudfare.md](cloudfare.md)

### More

- mutual authent is same thing but in 2 ways.
In what we described Alice has the guarantee she talks to Bob, but not vice-vesa (known as mtls)
Bob does not have the gurantee he is talking to Alice.

- From [Wikipedia](https://en.wikipedia.org/wiki/Self-signed_certificate): In cryptography and computer security, a self-signed certificate is a certificate that is not signed by a certificate authority (CA).

## Links

- https://medium.com/sitewards/the-magic-of-tls-x509-and-mutual-authentication-explained-b2162dec4401
- https://www.youtube.com/watch?v=7W7WPMX7arI
- https://www.youtube.com/watch?v=4nGrOpo0Cuc
- https://www.youtube.com/watch?v=T4Df5_cojAs


[see certificate](../lab-env/README.md#ssh-summary)