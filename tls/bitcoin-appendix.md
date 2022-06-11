# Bitcoin appendix

## Book

[Mastering bitcoin, chapter 4 (Key, Addresses), p55-80](https://www.oreilly.com/library/view/mastering-bitcoin-2nd/9781491954379/).

## Key take-away
- page 55
    > - Cryptography encompasses more that secret wirting (encryption)
    > - It can be used to prove knowledge of secret without revealing secret (digital signature)
    > - Or prove authenticity of data (digital fingerprint)
    > - Bitcoin communication and transaction data are not encrypted
    - Bitcoin use digital signature and fingerprint
    - We can can make parallel with signature explained [for PKI certificate](./in-learning-complement/learning-ssl-tld.md#digital-signature) where we sign data using certificate.
    - In bitcoin we sign transactions and verify it using public key (p141) 

- page 57
    - **`Private key [k] `** ==> `Elliptic curve multiplication [K=k*G] (one-way)` ==> **`Public key [K]`** == `(Double) Hashing function + Base58Check Encode  (one-way)` ==> **`Bitcoin Address [A] (Base58Check Encoded public Key Hash)`**
    <!-- in fig 4-6 payload is the hash of public key, proof: in code snippet p70, we append checksum to unencoded address -->
    - Base58 check encoding includes a checksum to protect against error and used to correctly transcribe a number (ensure we use good Bitcoin adress)
    - Quoting https://en.wikipedia.org/wiki/Checksum
    - > A checksum is a small-sized block of data derived from another block of digital data for the purpose of detecting errors that may have been introduced during its transmission or storage. By themselves, checksums are often used to verify data integrity but are not relied upon to verify data authenticity.
    -  Similar to [digital signature](./in-learning-complement/learning-ssl-tld.md#digital-signature) without autenticity part 

- What is a fingerprint?
    - A fingerprint is a checkum 
- Quoting: http://www.dg77.net/tekno/securite/md5.htm
    > **Signature et empreinte**

    > **Signature numérique**

    > Une signature numérique (ou digitale) repose sur les principes de l'empreinte et des paires de clefs asymétriques (voir pages précédentes).

    > La signature est une empreinte chiffrée par la clef privée. Si on peut la déchiffrer à l'aide de la clef publique de l'auteur, et si son résultat est conforme au fichier correspondant à l'empreinte, alors on est sûr qu'il provient bien de son expéditeur supposé.

    > **Empreinte, fingerprint, checksum, md5.**

    > Notion d'empreinte ("fingerprint") : somme de contrôle ("checksum") résultant d'un calcul à partir d'un ensemble de données ou fichiers, telle que la probabilité d'une "collision" avec un autre ensemble (i.e. deux fichiers donnant exactement le même résultat) est très faible. On utilise couramment les algorythme MD5 (du Pr Ronald L. Rivest) ou SHA-1. Exemple : un fichier gaston.txt sera accompagné d'un fichier de controle gaston.txt.md5. Si, en relançant le calcul sur gaston.txt on obtient un résultat différent du .md5 reçu, c'est qu'un des deux fichiers est altéré. Sous UNIX, Linux et cie, md5sum est pour ainsi dire une commande standard. Les utilisateurs de MS Windows devront chercher un programme, mais les solutions ne manquent pas.

    > La fonction MD5 n’est plus admise comme suffisante en tant que composant cryptographique, mais elle suffit toujours à s’assurer avec une probabilité confortable qu’un ensemble de données ne se trouve pas altéré.

    > Exemple minimal d'utilisation de md5sum sous Linux
        > - Création de la somme de contrôle d'un fichier "monfichier.txt" :  `md5sum  monfichier.txt  >  monfichier.md5` 
        >  - Vérification :  `md5sum  -c=monfichier.md5  monfichier.txt` 

    > Exemple avec Fastsum pour Windows :
        > - Création d'un "digest" MD5 pour un fichier "progbis.exe" dans le répertoire "report" :  `fsum  progbis.exe  progbis.md5` 
        > - Vérification :  `fsum  progbis.exe  progbis.md5  /V`
        >  Si le diagnostic signale "file changed" c'est qu'il y a une anomalie (le résultat n'est pas conforme au fichier md5 présenté). 
- We also have public key fingerprint
    - Quoting https://en.wikipedia.org/wiki/Public_key_fingerprint
    - > In public-key cryptography, a public key fingerprint is a short sequence of bytes used to identify a longer public key. Fingerprints are created by applying a cryptographic hash function to a public key. Since fingerprints are shorter than the keys they refer to, they can be used to simplify certain key management tasks. In Microsoft software, "thumbprint" is used instead of "fingerprint." 
    - > For example, if Alice wishes to authenticate a public key as belonging to Bob, she can contact Bob over the phone or in person and ask him to read his fingerprint to her, or give her a scrap of paper with the fingerprint written down. Alice can then check that this trusted fingerprint matches the fingerprint of the public key. Exchanging and comparing values like this is much easier if the values are short fingerprints instead of long public keys. 

- We also have certificate fingerprint
    - Quoting https://knowledge.digicert.com/solution/SO9840.html
    > - A certificate's fingerprint is the unique identifier of the certificate. Microsoft Internet Explorer calls it Thumbprint. Browsers tend to display it as if it were a part of the certificate. It is not a part of the certificate, but it is computed from it.
    > - A fingerprint is the MD5 digest of the der-encoded Certificate Info, which is an ASN.1 type specified as part of the X.509 specification.
    > - The Certificate Fingerprint is a digest (hash function) of a certificate in x509 binary format. It can be calculated by different algorithms, such as SHA1 for Microsoft Internet Explorer.

## Note

Note tanenmbaum p914, MAC uses MD5 for data integrity. `Datablock + MD5` is encrypted with derived key from nonce and preliminary key (MAC key).
See link with [Exchange key and their usage](./tls-certificate.md#exchange-key-and-their-usage).
and [SSL in short](./tls-certificate.md#ssl-in-short).