# TLS explained to myself

## Basic concept

[For the reamaining of this article. Alice is message sender, Bob is the receiver](https://fr.wikipedia.org/wiki/Alice_et_Bob).

- Symetric cryptography: We use same key to encode and decode.
- Asymetric cryptography: we generate two keys, key1 and key2.
When we encode with key 1 we can decode with key 2  and vice versa.
One key is in general private, and other one is public.

Thus definition given [here](https://youtu.be/T4Df5_cojAs?t=128)

1. Any message encrypted with Bob's public key can only be decrypted with Bob's private key.
In that case Recipient (Bob) give public key to sender (Alice). 
Sender (Alice) use recipient (Bob) public key to encode message, recipient (Bob) use its private key to decode the message.

2. Anyone with access to Alice's public key can verify that a message (**signature**)
could only have been created by someone with access to Alice's private key. (it was encrypted using private key here)

In that case Alice encrypt message using her private key. Bob check tries to decode (digital signature) using Alice public key. 
If he can decode, it proves Alice had encrypted this message.
This is what a CA does.

I had read:
It makes no sense to encrypt anything with your private key, 
because the message could then be decrypted by the public key that everybody has access to. 
When somebody wants to send you a secret message, they encrypt it using your public key, and then only you can decrypt it, 
because only you know your private key
-> not the case for signature

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

````
@startuml
title ASymetric cryptography and man in the middle attack, and certificate authority
    
actor Alice 
actor Robber 
actor Bob
actor CA

== one time request ==

Bob -> CA : send money to request CA to sign certificate which contains Bob public key
CA -> CA : sign Bob certificate by encoding it using CA private key (signature mode)
CA -> Bob : provide certicate (fullchain.pem) to Bob which contains pub key and a private key (privkey.pem)

== usual request ==

Alice -> Bob : send 'client hello' message. It includes which TLS version the client supports, the cipher suites supported, and a string of random bytes known as the "client random."
Bob -> Alice : send 'server hello' with BOB certificate which contains BOB public key, cipher and "server random"

alt original bob
    Alice -> Alice : Authentification - Alice decode BOB certificate with CA public key  embedded in browser (signature)
    Alice-> Alice: Authentification - Certificate is approved, and server identity is proved (client is interaction with domain owner)
    Alice -> Alice: get BOB public key from certificate
    Alice -> Alice : encode premaster secret with BOB public key
    Alice -> Bob : send premaster secret
    Bob -> Bob : decode premaster secret with BOB private key
    Bob -> Bob: Generate session keys from the client random, the server random, and the premaster secret.
    Alice -> Alice:  Also generate session keys and should reach same result as Bob
    Alice -> Bob: send client is ready with session key (symetric)
    Bob -> Alice: send server is ready with session key (symetric)
else robber
    Bob -> Robber: steal certificate and replace by its own 
    Robber -> Alice: send message with altered certificate 
    Alice -> Alice : decode Altered (Robber)certificate with CA public key (signature)
    Alice->Alice: It failed and refuse to talk with Bob
end
@enduml
````

Then they continue with the session key.

This explain why when we generate a certificate we have:
- a cert file with public key
- a key file with private key 

See actual application here: https://github.com/scoulomb/myDNS/blob/master/2-advanced-bind/5-real-own-dns-application/6-use-linux-nameserver-part-h.md


This is known as TLS handshake diagram is aligned with explanation given here:
https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/ (not the Diffie-Hellman)

I mirrored the page [cloudfare.md](cloudfare.md)

Some CA like let's encrypt are free.

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
