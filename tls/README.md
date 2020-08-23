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


    @startuml
    title Symetric cryptography
    
    Alice->Bob: exchange key in clear
    Alice -> Robber: steal the key
    Alice -> Alice : encode message with public key
    Alice -> Bob : send message to  bob
    Alice -> Robber : steal message 
    Robber -> Robber : decode the message
    Bob -> Bob : decode message with key wiht key
    @enduml


#### This why asymetric is used.

- Bob sends public key to Alice,
- Alice encrypts message with Bob's public key,
- Bob decrypts message with its private key.

Using asymmetric cryptography has an higer cost than symmetric.
This is the reason why, they generate a chiffer, Alice and Bob exchange this chiffer in asymmetric way,
and then continue to exchange and continue using this chiffer with symmetric cryptography.


    @startuml
    title ASymetric cryptography
    
    Alice -> Bob : initiate authent
    Bob -> Alice : send BOB public key
    Alice -> Alice : encode message with BOB public key
    Alice -> Bob : send message to  bob
    Bob -> Bob : decode message with key wiht BOB private key
    @enduml

### Man in the middle attach and need of a CA

**Problem**: The public key can be intercepted and substituted (man in the middle attach)
Robber intercepts Bob's  public key and replace by Robber key. Alice encrypts the message using robber key (believing it is the one of Bob).
She sends it back to robber (thinking it is bob).  Robber decodes Alice message. He reencodes it using Bob's public key (to not arouse suspicions) and send it to bob.

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


**Solution**: Certificate Authority (CA)

- Bob in exchange of money :) request CA to sign a certificate which contains Bob's public key,
- CA signs Bob's certificate and encode it with it CA private key (**signature**),
- When Alice to communicate with Bob: Bob sends his certificate to Alice which contains Bob's public key,
- Alice browser embeds CA public key, if she can decode the certificate sent by Bob (signature): 
Certificate is approved, and server identity is proved. Otherwise Alice refuses the connection.  She can get Bob's public key in certificate. Then communication happens as described in 
[Symetric and aysmetric section](#symetic-and-asymetric).


    @startuml
    title ASymetric cryptography and man in the middle attack, and certificate authority
        
    actor Alice 
    actor Robber 
    actor Bob
    actor CA
    
    == one time request ==
    
    Bob -> CA : send money to request CA to sign certificate which contains Bob public key
    CA -> CA : sign Bob certificate by encoding it using CA private key
    CA -> Bob : provide certicate to Bob
    
    == usual request ==
    
    Alice -> Bob : initiate authent
    
    alt original bob
        Bob -> Alice : send BOB certificate which contains BOB public key
        Alice -> Alice : decode BOB certificate with CA public key (signature)
        Alice-> Alice: Certificate is approved, and server identity is proved
        Alice -> Alice: get BOB public key in certificate
        Alice -> Alice : encode message with BOB public key
        Alice -> Bob : send message to  bob
        Bob -> Bob : decode message with key wiht BOB private key
    else robber
        Bob -> Robber: steal certificate and replace by its own 
        Robber -> Alice: send message with altered certificate 
        Alice -> Alice : decode Altered (Robber)certificate with CA public key (signature), failure
        Alice->Alice: Refuse to talk with Bob
    end
    @enduml

### More

- mutual authent is same think but in 2 ways.
In what we described Alice has the guarantee she talks to Bob, but not vice-vesa
Bob does not have the gurantee he is talking to Alice.

- From [Wikipedia](https://en.wikipedia.org/wiki/Self-signed_certificate): In cryptography and computer security, a self-signed certificate is a certificate that is not signed by a certificate authority (CA).

## Links

- https://medium.com/sitewards/the-magic-of-tls-x509-and-mutual-authentication-explained-b2162dec4401
- https://www.youtube.com/watch?v=7W7WPMX7arI
- https://www.youtube.com/watch?v=4nGrOpo0Cuc
- https://www.youtube.com/watch?v=T4Df5_cojAs

