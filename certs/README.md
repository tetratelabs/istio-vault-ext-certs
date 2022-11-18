# Istio Certificate details

Root Certificate.

```console
openssl x509 -in root-cert.pem -text
```

```
  Certificate:
      Data:
          Version: 3 (0x2)
          Serial Number:
              42:67:4a:6d:22:54:a7:28:31:64:ca:58:2e:99:cd:66:3e:6d:e4:de
          Signature Algorithm: sha256WithRSAEncryption
          Issuer: O = Istio, CN = Root CA
          Validity
              Not Before: Nov 17 13:00:39 2022 GMT
              Not After : Nov 14 13:00:39 2032 GMT
          Subject: O = Istio, CN = Root CA
          Subject Public Key Info:
              Public Key Algorithm: rsaEncryption
                  Public-Key: (4096 bit)
                  Modulus:
                      00:c4:80:6c:d7:04:0d:c0:4e:22:6c:be:4a:a2:45:
                      ...
                      df:57:95
                  Exponent: 65537 (0x10001)
          X509v3 extensions:
              X509v3 Subject Key Identifier: 
                  5C:0F:9A:96:63:65:D6:3D:8E:FE:04:87:8C:16:0B:66:84:6C:71:B3
              X509v3 Basic Constraints: critical
                  CA:TRUE
              X509v3 Key Usage: critical
                  Digital Signature, Non Repudiation, Key Encipherment, Certificate Sign
      Signature Algorithm: sha256WithRSAEncryption
      Signature Value:
          70:0b:b7:98:3b:66:b6:6a:4a:a2:ac:4c:ce:49:df:bb:cc:c4:
          ...
          b9:36:3e:f8:44:94:52:60
  -----BEGIN CERTIFICATE-----
  MIIFFDCCAvygAwIBAgIUQmdKbSJUpygxZMpYLpnNZj5t5N4wDQYJKoZIhvcNAQEL
  ...
  uTY++ESUUmA=
  -----END CERTIFICATE-----
```


Intermediate Certificate.

```console
openssl x509 -in k3s-cluster1/ca-cert.pem -text
```

```
  Certificate:
      Data:
          Version: 3 (0x2)
          Serial Number:
              5d:d1:d1:46:08:35:f5:91:e6:03:c0:aa:56:4d:80:28:4e:4e:79:b9
          Signature Algorithm: sha256WithRSAEncryption
          Issuer: O = Istio, CN = Root CA
          Validity
              Not Before: Nov 17 13:00:40 2022 GMT
              Not After : Nov 16 13:00:40 2024 GMT
          Subject: O = Istio, CN = Intermediate CA, L = k3s-cluster1
          Subject Public Key Info:
              Public Key Algorithm: rsaEncryption
                  Public-Key: (4096 bit)
                  Modulus:
                      00:d3:6f:bf:41:0e:69:94:65:d3:6c:66:c1:39:a9:
                      ...
                      28:a7:d5
                  Exponent: 65537 (0x10001)
          X509v3 extensions:
              X509v3 Subject Key Identifier: 
                  01:CC:15:FE:F3:39:A9:C4:C7:A2:A4:E9:EF:F5:27:00:B4:C6:A6:0F
              X509v3 Basic Constraints: critical
                  CA:TRUE, pathlen:0
              X509v3 Key Usage: critical
                  Digital Signature, Non Repudiation, Key Encipherment, Certificate Sign
              X509v3 Subject Alternative Name: 
                  DNS:istiod.istio-system.svc
              X509v3 Authority Key Identifier: 
                  5C:0F:9A:96:63:65:D6:3D:8E:FE:04:87:8C:16:0B:66:84:6C:71:B3
      Signature Algorithm: sha256WithRSAEncryption
      Signature Value:
          36:a5:5d:a5:33:48:e6:00:ce:74:e7:0f:df:98:b3:38:67:09:
          ...
          82:46:65:ce:ea:13:dd:f0
  -----BEGIN CERTIFICATE-----
  MIIFfTCCA2WgAwIBAgIUXdHRRgg19ZHmA8CqVk2AKE5OebkwDQYJKoZIhvcNAQEL
  ...
  EBFrT5QK4o5kgkZlzuoT3fA=
  -----END CERTIFICATE-----
```

> **REMARK:** Note the X509v3 Basic Constraint `CA:TRUE`, which is necessary so that `istiod` actually can generate workload certificates from this intermediate certificate.
