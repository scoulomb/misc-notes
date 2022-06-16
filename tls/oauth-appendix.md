# OAuth appendix

## Reference

Links with [oauth section](../oauth/README.md).

## Signing algorithm

From https://auth0.com/docs/get-started/applications/signing-algorithms

> Signing algorithms are algorithms used to sign tokens issued for your application or API. A signature is part of a JSON Web Token (JWT) and is used to verify that the sender of the token is who it says it is and to ensure that the message wasn't changed along the way.

> You can select from the following signing algorithms:

- > RS256 (RSA Signature with SHA-256): An asymmetric algorithm, which means that there are two keys: one public key and one private key that must be kept secret. Auth0 has the private key used to generate the signature, and the consumer of the JWT retrieves a public key from the metadata endpoints provided by Auth0 and uses it to validate the JWT signature.
- > HS256 (HMAC with SHA-256): A symmetric algorithm, which means that there is only one private key that must be kept secret, and it is shared between the two parties. Since the same key is used both to generate the signature and to validate it, care must be taken to ensure that the key is not compromised. This private key (or secret) is created when you register your application (client secret) or API (signing secret) and choose the HS256 signing algorithm.

> The most secure practice, and our recommendation, is to use RS256 because:

- > With RS256, you are sure that only the holder of the private key (Auth0) can sign tokens, while anyone can check if the token is valid using the public key.
- > With RS256, if the private key is compromised, you can implement key rotation without having to re-deploy your application or API with the new secret (which you would have to do if using HS256).

## And this key can be in a certificate

Example of google Authorization code:  https://developers.google.com/identity/protocols/oauth2/openid-connect

We had seen [auth code flow](../oauth/1-oauth2.0-with-authorization-code.puml) where we have exactly same step as here: https://developers.google.com/identity/protocols/oauth2/openid-connect#authenticatingtheuser

> Validating an ID token

> You need to validate all ID tokens on your server unless you know that they came directly from Google. For example, your server must verify as authentic any ID tokens it receives from your client apps.

[...]

> ID token or rely on it as an assertion that the user has authenticated, you must validate it.

> Validation of an ID token requires several steps:

    1. Verify that the ID token is properly signed by the issuer. Google-issued tokens are signed using one of the certificates found at the URI specified in the jwks_uri metadata value of the Discovery document.
    2. Verify that the value of the iss claim in the ID token is equal to https://accounts.google.com or accounts.google.com.
    3. Verify that the value of the aud claim in the ID token is equal to your app's client ID.
    4. Verify that the expiry time (exp claim) of the ID token has not passed.
    5. If you specified a hd parameter value in the request, verify that the ID token has a hd claim that matches an accepted domain associated with a Google Cloud organization.

Steps 2 to 5 involve only string and date comparisons which are quite straightforward, so we won't detail them here.

> The first step is more complex, and involves cryptographic signature checking. For debugging purposes, you can use Google's tokeninfo endpoint to compare against local processing implemented on your server or device. Suppose your ID token's value is XYZ123. Then you would dereference the URI https://oauth2.googleapis.com/tokeninfo?id_token=XYZ123. If the token signature is valid, the response would be the JWT payload in its decoded JSON object form.

> **The tokeninfo endpoint is useful for debugging but for production purposes, retrieve Google's public keys from the keys endpoint and perform the validation locally. You should retrieve the keys URI from the Discovery document using the jwks_uri metadata value. Requests to the debugging endpoint may be throttled or otherwise subject to intermittent errors.**

> Since Google changes its public keys only infrequently, you can cache them using the cache directives of the HTTP response and, in the vast majority of cases, perform local validation much more efficiently than by using the tokeninfo endpoint. This validation requires retrieving and parsing certificates, and making the appropriate cryptographic calls to check the signature. Fortunately, there are well-debugged libraries available in a wide variety of languages to accomplish this (see jwt.io).


Google discovery document can be found here: https://accounts.google.com/.well-known/openid-configuration


````
$ curl https://accounts.google.com/.well-known/openid-configuration
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1280  100  1280    0     0  10847      0 --:--:-- --:--:-- --:--:-- 10940{
 "issuer": "https://accounts.google.com",
 "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
 "device_authorization_endpoint": "https://oauth2.googleapis.com/device/code",
 "token_endpoint": "https://oauth2.googleapis.com/token",
 "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
 "revocation_endpoint": "https://oauth2.googleapis.com/revoke",
 "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
 "response_types_supported": [

````

And then 

```

$ curl https://www.googleapis.com/oauth2/v1/certs
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3675    0  3675    0     0  30882      0 --:--:-- --:--:-- --:--:-- 30882{
  "2b09e744d58c9955d4f240b6a92f7b37fead2ff8": "-----BEGIN CERTIFICATE-----\nMIIDJzCCAg+gAwIBAgIJALrt0FL5utANMA0GCSqGSIb3DQEBBQUAMDYxNDAyBgNV\nBAMMK2ZlZGVyYXRlZC1zaWdub24uc3lzdGVtLmdzZXJ2aWNlYWNjb3VudC5jb20w\nHhcNMjIwNjE2MTUyMTU4WhcNMjIwNzAzMDMzNjU4WjA2MTQwMgYDVQQDDCtmZWRl\ncmF0ZWQtc2lnbm9uLnN5c3RlbS5nc2VydmljZWFjY291bnQuY29tMIIBIjANBgkq\nhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAts0Gom7Zc7IE+jgtd6ggN7sjEhw/Qp8l\nlljq3J1a748CMF3gNo+Uq3yZCOpeuWG418li24SKyKvjq4FRQCiPNzOLlczgHBza\nUNqeOZ4nH1mCjG7TA0asZRJP/9KcFCefzy5mGL1OJCR/c8vJTjjfU3KgoBZfdL2K\ndDcuO89jhuCuSst/ybME3LWAn8swO1NBZhsZqUgU2r+Pzh1j/6+jAnb/O35CoK1N\nCpvZTgbhHbjH874L+ZMIVAQ9v+bkhl0usFFc+Tvhchh+6RErttHi15AapJ8UoNFA\ndsxv1+OYw58c0thtkWdoK3vmZWxYRWgfl0XuDFvoazR8KaW8a9EPXwIDAQABozgw\nNjAMBgNVHRMBAf8EAjAAMA4GA1UdDwEB/wQEAwIHgDAWBgNVHSUBAf8EDDAKBggr\nBgEFBQcDAjANBgkqhkiG9w0BAQUFAAOCAQEAXFUnwzpD+jaHfYkUblMD00MCy8aQ\nNDRM4xFGd1rcjLBwfPDBRAFSxEnueVwrInQ55u/utAryCzfx9Bv3DHywMz1rGAKo\nffO9iX39Xqa4Ms+Xm2UiZwFdA4DpeMcB8KozrJqByvHlZAqI8QlwIIrImm4eBRTQ\nRrEeDKc0d625t/QPX0JMb/1J9HmKmvGK1DlxLUOyH274g/g1fCtsnfp4Xc3+ptom\nNKfH1O0b8EI6LmUb9Zxcyow60OueAi4ZTRDjFLORHmxRgrYFkwe/dVt02MHkiWpS\ndoaHmNIIogZZjK6lSfwvqsuMIAfK4eYfyJTTSZeUu7+F4OYYo4BulQmYSw==\n-----END CERTIFICATE-----\n",
  "580adb0c32a175d950a1c1901c182fc17341ddc4": "-----BEGIN CERTIFICATE-----\nMIIDJzCCAg+gAwIBAgIJANZUApjSByJ+MA0GCSqGSIb3DQEBBQUAMDYxNDAyBgNV\nBAMMK2ZlZGVyYXRlZC1zaWdub24uc3lzdGVtLmdzZXJ2aWNlYWNjb3VudC5jb20w\nHhcNMjIwNjA4MTUyMTU3WhcNMjIwNjI1MDMzNjU3WjA2MTQwMgYDVQQDDCtmZWRl\ncmF0ZWQtc2lnbm9uLnN5c3RlbS5nc2VydmljZWFjY291bnQuY29tMIIBIjANBgkq\nhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA9rWgi2qEYWWMfkF7U947MGGZBx6pNVCU\ni9tgTKqmo+8J4MaMSnu0I2+YsyLeXh5r3ztzuk65w8egHtudxE1Q47VquJIcBs4Z\nn95607VCJeZxYBG3CYk1OnUWGuTjPbpXIeAlnhc+MpgUvjphE6nMEsQsxd/PTh9r\nj/gUdahA0xRQBj+8kOhnOoFQAO0ZV5mP5gyhTBZubQOIeKVvjZJvxZqSoSWqocnd\nVGrVEAYMdjXbxdw8Z9Gf0NiWk3rImEjzXu65a62cJ6Fn1PjQEhohvwM0cCvKS6kq\n7bZ3ABJ//9VQtC6SyN0jo/jxAIVfWyVmZG+o+JIClrxjeMlRkqbbZQIDAQABozgw\nNjAMBgNVHRMBAf8EAjAAMA4GA1UdDwEB/wQEAwIHgDAWBgNVHSUBAf8EDDAKBggr\nBgEFBQcDAjANBgkqhkiG9w0BAQUFAAOCAQEAgqp79QVuGmFkXJ1MFrP0DF5D6xRN\ng783l0haNwAxjUy0tjSsPrjI3lzSrCdAFG8yLTgo1ueMujS8LTvbTdtLQJlMYxYM\nfkAYRg4ak4P15YbBCARnU4gH/sBXaxTITbSEz/UfXvQwa+kE1CjwUbUpDmhyQThv\nZC6aFt2b5a9oGnJkfthaa0KouLBccGli3hsJTMeai6YmpmYuy50HKtWtb1CqjOGp\nb8rrmZUaIwO60iC155D1o8UZZlIlh/NT5EPbUozFOz+ZVsklz2W2ZDjZQKlXSvr6\n2vvIn1lcJY+eVPsHy171goUFl3PUO51OHGWDTF0mNxVElQ8xrMzFfxq42Q==\n-----END CERTIFICATE-----\n",
  "7483a088d4ffc00609f0e22f3c22da5fe3906ccc": "-----BEGIN CERTIFICATE-----\nMIIDJzCCAg+gAwIBAgIJAO4cJm6sqZ2qMA0GCSqGSIb3DQEBBQUAMDYxNDAyBgNV\nBAMMK2ZlZGVyYXRlZC1zaWdub24uc3lzdGVtLmdzZXJ2aWNlYWNjb3VudC5jb20w\nHhcNMjIwNTMxMTUyMTU2WhcNMjIwNjE3MDMzNjU2WjA2MTQwMgYDVQQDDCtmZWRl\ncmF0ZWQtc2lnbm9uLnN5c3RlbS5nc2VydmljZWFjY291bnQuY29tMIIBIjANBgkq\nhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAyF2BwlYB66C/+FKSk4riKizNKGjTyi2L\nYz/MImk95lPcW3SDpheJRghc/5iaZxpf1xAMDwHphdkEPal/kNnqLu49B7XGv9iP\nRUgzAEBD+w9E1CjrxzCRhM8MRVgUo7Msnydd6wOqNDKFCD1TlKpkGrUS32PB1jsu\n+OuwG6HCL1cxSNzP7SCDgCKecPvnpwcNarc3FzzJDraUaPX639cvvBf5PDeltBzk\nAIbSY0YS6RVBmZpKEDO/C1GtbADqvpz95uSdYO6H4tjUQtFwooNsAL8tv1TmLSeh\nClFyYzaESOBfSCpxudNmPgBEzfAZM/WoZ2JukAo73Ynu3YxJOJtMTQIDAQABozgw\nNjAMBgNVHRMBAf8EAjAAMA4GA1UdDwEB/wQEAwIHgDAWBgNVHSUBAf8EDDAKBggr\nBgEFBQcDAjANBgkqhkiG9w0BAQUFAAOCAQEAMsdMZkbjkpvdQtEb0OheGTibWGJ0\nHA23VQrslcNlntmH3ACH3JVoX2kr6RS1Om+CKXu99JUYVLINlocJxSWb2PGMd6Qs\n/veBIaFCME3Rpj/fHWIAQAA2/ye/LOUEUj1bPXwnCi4dEMp/HXFHxixLUHo+W1a0\nAmhgSw/v45YCzXyYqgkfBWQas7+gd4/Bppf2f2hwRZynjGK2vGVMpHpQCI5jXqCw\nz/hpuJ3luAO3dDF4reWqR24EV0RMjOq8a5CgmkqL3SAfHqt8akfkgzBD7sh9HohY\nMg6SIzxy9EkcZReAQazQwIjPFEcKGYmsC4WRrwUVw9kQJjzSu1MB02TJjA==\n-----END CERTIFICATE-----\n"
}



$ curl https://www.googleapis.com/oauth2/v3/certs
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1540    0  1540    0     0  13050      0 --:--:-- --:--:-- --:--:-- 13050{
  "keys": [
    {
      "alg": "RS256",
      "n": "9rWgi2qEYWWMfkF7U947MGGZBx6pNVCUi9tgTKqmo-8J4MaMSnu0I2-YsyLeXh5r3ztzuk65w8egHtudxE1Q47VquJIcBs4Zn95607VCJeZxYBG3CYk1OnUWGuTjPbpXIeAlnhc-MpgUvjphE6nMEsQsxd_PTh9rj_gUdahA0xRQBj-8kOhnOoFQAO0ZV5mP5gyhTBZubQOIeKVvjZJvxZqSoSWqocndVGrVEAYMdjXbxdw8Z9Gf0NiWk3rImEjzXu65a62cJ6Fn1PjQEhohvwM0cCvKS6kq7bZ3ABJ__9VQtC6SyN0jo_jxAIVfWyVmZG-o-JIClrxjeMlRkqbbZQ",
      "kty": "RSA",
      "e": "AQAB",
      "use": "sig",
      "kid": "580adb0c32a175d950a1c1901c182fc17341ddc4"
    },
    {
      "alg": "RS256",
      "use": "sig",
      "e": "AQAB",
      "kid": "2b09e744d58c9955d4f240b6a92f7b37fead2ff8",
      "kty": "RSA",
      "n": "ts0Gom7Zc7IE-jgtd6ggN7sjEhw_Qp8llljq3J1a748CMF3gNo-Uq3yZCOpeuWG418li24SKyKvjq4FRQCiPNzOLlczgHBzaUNqeOZ4nH1mCjG7TA0asZRJP_9KcFCefzy5mGL1OJCR_c8vJTjjfU3KgoBZfdL2KdDcuO89jhuCuSst_ybME3LWAn8swO1NBZhsZqUgU2r-Pzh1j_6-jAnb_O35CoK1NCpvZTgbhHbjH874L-ZMIVAQ9v-bkhl0usFFc-Tvhchh-6RErttHi15AapJ8UoNFAdsxv1-OYw58c0thtkWdoK3vmZWxYRWgfl0XuDFvoazR8KaW8a9EPXw"
    },
    {
      "kty": "RSA",
      "use": "sig",
      "e": "AQAB",
      "alg": "RS256",
      "kid": "7483a088d4ffc00609f0e22f3c22da5fe3906ccc",
      "n": "yF2BwlYB66C_-FKSk4riKizNKGjTyi2LYz_MImk95lPcW3SDpheJRghc_5iaZxpf1xAMDwHphdkEPal_kNnqLu49B7XGv9iPRUgzAEBD-w9E1CjrxzCRhM8MRVgUo7Msnydd6wOqNDKFCD1TlKpkGrUS32PB1jsu-OuwG6HCL1cxSNzP7SCDgCKecPvnpwcNarc3FzzJDraUaPX639cvvBf5PDeltBzkAIbSY0YS6RVBmZpKEDO_C1GtbADqvpz95uSdYO6H4tjUQtFwooNsAL8tv1TmLSehClFyYzaESOBfSCpxudNmPgBEzfAZM_WoZ2JukAo73Ynu3YxJOJtMTQ"
    }
  ]
}

````

Where we can see JSON web key:
- https://stackoverflow.com/questions/31277898/difference-between-v1-v2-and-v3-in-https-www-googleapis-com-oauth2-v3-certs
- https://self-issued.info/docs/draft-ietf-jose-json-web-key-12.html#x5cDef
- https://auth0.com/docs/secure/tokens/json-web-tokens/json-web-key-set-properties
- https://datatracker.ietf.org/doc/html/rfc7517#section-4

Also here
https://www.ssi.gouv.fr/uploads/2020/09/anssi-guide-recommandations_pour_la_securisation_de_la_mise_en_oeuvre_du_protocole_openid_connect-v1.0.pdf

They recommend to store authoization code and access token as [fingerprint](./bitcoin-appendix.md).

<!-- no need to implement google full auth code flow -->

<!-- oauth concluded -->
