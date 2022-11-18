# Kubernetes API Cert and istiod JWT token


Content of the kubernetes API certificates.

```console
openssl x509 -in k8sapi-cert1.pem -text
```

```
  Certificate:
      Data:
          Version: 3 (0x2)
          Serial Number: 0 (0x0)
      Signature Algorithm: ecdsa-with-SHA256
          Issuer: CN=k3s-server-ca@1668718694
          Validity
              Not Before: Nov 17 20:58:14 2022 GMT
              Not After : Nov 14 20:58:14 2032 GMT
          Subject: CN=k3s-server-ca@1668718694
          Subject Public Key Info:
              Public Key Algorithm: id-ecPublicKey
                  Public-Key: (256 bit)
                  pub: 
                      04:79:3a:e6:71:bb:3b:82:bb:4a:42:c2:55:30:56:
                      ba:42:cc:02:97:79:13:1d:50:3e:d5:d8:fd:e1:5d:
                      c8:23:3c:78:1b:40:11:98:56:5c:03:4a:1c:bc:be:
                      96:89:02:b1:9b:d4:72:c6:0b:ad:2a:02:44:34:3b:
                      96:a1:96:e5:ee
                  ASN1 OID: prime256v1
                  NIST CURVE: P-256
          X509v3 extensions:
              X509v3 Key Usage: critical
                  Digital Signature, Key Encipherment, Certificate Sign
              X509v3 Basic Constraints: critical
                  CA:TRUE
              X509v3 Subject Key Identifier: 
                  98:CC:58:4D:4B:A0:61:A3:73:4C:86:84:82:F9:5F:D3:95:5E:74:F3
      Signature Algorithm: ecdsa-with-SHA256
          30:46:02:21:00:b7:8a:49:a7:b9:e1:a0:1d:7b:ad:ec:37:ae:
          a6:e3:0f:b1:1f:7c:2d:60:02:52:db:32:ed:b0:48:ca:35:d1:
          36:02:21:00:f5:dc:1c:8c:11:1e:b1:3a:40:af:0e:80:be:ae:
          05:36:0b:03:1c:08:3d:be:24:7d:84:75:b1:7f:62:0d:d8:8e
  -----BEGIN CERTIFICATE-----
  MIIBeDCCAR2gAwIBAgIBADAKBggqhkjOPQQDAjAjMSEwHwYDVQQDDBhrM3Mtc2Vy
  dmVyLWNhQDE2Njg3MTg2OTQwHhcNMjIxMTE3MjA1ODE0WhcNMzIxMTE0MjA1ODE0
  WjAjMSEwHwYDVQQDDBhrM3Mtc2VydmVyLWNhQDE2Njg3MTg2OTQwWTATBgcqhkjO
  PQIBBggqhkjOPQMBBwNCAAR5OuZxuzuCu0pCwlUwVrpCzAKXeRMdUD7V2P3hXcgj
  PHgbQBGYVlwDShy8vpaJArGb1HLGC60qAkQ0O5ahluXuo0IwQDAOBgNVHQ8BAf8E
  BAMCAqQwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUmMxYTUugYaNzTIaEgvlf
  05VedPMwCgYIKoZIzj0EAwIDSQAwRgIhALeKSae54aAde63sN66m4w+xH3wtYAJS
  2zLtsEjKNdE2AiEA9dwcjBEesTpArw6Avq4FNgsDHAg9viR9hHWxf2IN2I4=
  -----END CERTIFICATE-----
```

Content of the Istiod ServiceAccount JWT token.

```console
jq -R 'split(".") | .[0],.[1] | @base64d | fromjson' <<< $(cat istiod1.jwt)
```

```json
  {
    "alg": "RS256",
    "kid": "Yf1kXn44WKQpqBoRlIY-_CD3187pbKgLRDgTGgr3Erc"
  }
  {
    "iss": "kubernetes/serviceaccount",
    "kubernetes.io/serviceaccount/namespace": "istio-system",
    "kubernetes.io/serviceaccount/secret.name": "istiod",
    "kubernetes.io/serviceaccount/service-account.name": "istiod",
    "kubernetes.io/serviceaccount/service-account.uid": "a7ed5518-73a1-4e35-bab5-a22e3d2e5008",
    "sub": "system:serviceaccount:istio-system:istiod"
  }
```

After you have set-up the vault configuration, you can test the kubernetes auth method.

```console
AUTH_RESPONSE=$(curl --request POST --data "{\"jwt\": \"`cat istiod1.jwt`\", \"role\": \"istiod\"}" http://localhost:8200/v1/auth/kubernetes-cluster1/login)
echo $AUTH_RESPONSE | jq
```

```json
  {
    "request_id": "9ccea87c-064d-7178-bb44-c44cd89aa2ab",
    "lease_id": "",
    "renewable": false,
    "lease_duration": 0,
    "data": null,
    "wrap_info": null,
    "warnings": null,
    "auth": {
      "client_token": "hvs.CAESILOCPwIGzFX7EdX18x0rwJ3LNO8c8Q_Gp32npcWmvrwGGh4KHGh2cy5jUktUeW9WVWswTFJUN3dLQ2hzNVZZRU4",
      "accessor": "B1sP45E3Jf5wglL0r31LJT5k",
      "policies": [
        "default",
        "istiod-certs-cluster1"
      ],
      "token_policies": [
        "default",
        "istiod-certs-cluster1"
      ],
      "metadata": {
        "role": "istiod",
        "service_account_name": "istiod",
        "service_account_namespace": "istio-system",
        "service_account_secret_name": "istiod",
        "service_account_uid": "1edcbdc5-b5a2-44fa-a86d-3d4d384e7ae2"
      },
      "lease_duration": 86400,
      "renewable": true,
      "entity_id": "9bf66db0-e625-b2f3-af45-751d4bac894e",
      "token_type": "service",
      "orphan": true,
      "mfa_requirement": null,
      "num_uses": 0
    }
  }
```

We can now use this token to fetch our istio certificates.

```console
VAULT_TOKEN=$(echo $AUTH_RESPONSE | jq .auth.client_token --raw-output)
curl --request GET --header "X-Vault-Token: $VAULT_TOKEN" http://localhost:8200/v1/kubernetes-cluster1-secrets/istiod-service/certs | jq
```

```json
  {
    "request_id": "a42c5847-bb29-0c78-8298-b6628d7595cb",
    "lease_id": "",
    "renewable": false,
    "lease_duration": 2764800,
    "data": {
      "ca_cert": "-----BEGIN CERTIFICATE-----\nMIIFUjCCAzqgAwIBAgIJAN7fKYoUdoHWMA0GCSqGSIb3DQEBCwUAMCIxDjAMBgNV\nBAoMBUlzdGlvMRAwDgYDVQQDDAdSb290IENBMB4XDTIyMTExNzIyMDE1NFoXDTI0\nMTExNjIyMDE1NFowRDEOMAwGA1UECgwFSXN0aW8xGDAWBgNVBAMMD0ludGVybWVk\naWF0ZSBDQTEYMBYGA1UEBwwPaXN0aW9kLWNsdXN0ZXIxMIICIjANBgkqhkiG9w0B\nAQEFAAOCAg8AMIICCgKCAgEA0de2Lr+DhI0HlEcl6uDrJonpUttReh57ntNNLA4A\nH+7lb6LexQtw+byDQwlv4zId8yJ3nN5VntX5RLAlCAyOR1EPIkCYt2vnsK2lrp2P\nzJdETwjisDrBFmQHL3pl9iEU9fNru5+3ViPQEtCjyQsWEiuJHO5+ZWsRz7AeuN4I\nh4k41hahDRw9kNJTHngxxRoGAffsYQbuj6e8GLH0sBWp+D7SN7UBcoVFQr/Ui0fa\n66V+4ASGPVvijgTw0jRL1t7e0VguGX491M0gUUXf1TWfPqezct2bQTAb8+gwe2zf\nYpXVrcGEMSZmk7oBs+AJQlsq61eorKSX8FeOp+/Rz6/FN77bV51fqZ/tQiF4jJ7p\nh2lTz6upX/nO47N+QRsMRapEHsXReY0W3VS/WthbKhkNXvQw5MrZ4xg6QSVgRA4a\nsU23ZJ1KuUKgT2XsA/hL5L13kg8aa64y9azKSc2VHQe/N87MIhvzE1UD+Vn4928p\n9Oebq6+EiKBoAiUUG4OgssGOrL1YxU6X03yVwCFBAJdCfx+wjOE55MZonXxafbL2\n8yYNPEjWYiQZNrWuDnePCb5kS1XIoG7DZ7ta+WNutE+59pIyG6ECPZ7xJLYBquTa\n86hyKz59VdYVqAUYj+TYF81U3MWJCf1LSWEbe2Gg2Giy3dgaOCwifwXsFHn9WcSG\nPQkCAwEAAaNpMGcwHQYDVR0OBBYEFIUXcQuj9KKILV23iNGtPOTUmphDMBIGA1Ud\nEwEB/wQIMAYBAf8CAQAwDgYDVR0PAQH/BAQDAgLkMCIGA1UdEQQbMBmCF2lzdGlv\nZC5pc3Rpby1zeXN0ZW0uc3ZjMA0GCSqGSIb3DQEBCwUAA4ICAQC9Va6PpBpYji3A\nkscmFUjjR8Yi8HgCwLZgs6toy8RGMejM4ANsB22Kl/cYQx9YNODTTxd3GOqAPglB\nL2iqYP0+qJWU+h8u4n2Bgaz77DKmiIhKBhozeSUGltzFFK93zFwMhVEvlTOfwhgb\n2xS1iAAAGFvPYeJSRNwfTz59mFmIYErbjWIl+3pxjen0YD5AOntW+SkpJBfz7jqf\npvDEja1uP60kjSdqy4ppj6Dlo6/AwQpM2hbn1riD0MRcE56c0SNfuygfCpj/o2iu\nmmYHGPgoiN8MXM99GyJnQ3CZhl3MHlxZ2Uy7zln6h1OR8abLtyJjueu/7qbQi/+t\nJg8B4jg3ofZ4+Te+b+nmiJ06FQ2VpQSigpGTQQbsfkEM9Nio5+TaULLXyaazizD6\nYG1uIgxT14zRLkAcc+asT941qobHbshcqabqJQ3jeIeMAENBSTwtaQaY4HupCqGz\nUkca0gimyNa4U84CRzB6qkRA2Qu8mK4HggbzmzMLCIuCg2hALNw/HCZoN5lnT6ja\nbiTqljc00xswAlxKfmNtyUFd/Obsm5kdMG1Fc/gDwqeQoauzs2lO/ZJa8+AulJ3f\nPy9b7HnuhVI513gfjC/rueZiLWiz9SHToCZ1OEBhQ0+Gn1X1fCcb2/npDHrQ3WUJ\n1mNnbjvxR/Wi2cn30cQeb7BXEUCeWA==\n-----END CERTIFICATE-----\n",
      "ca_key": "-----BEGIN RSA PRIVATE KEY-----\nMIIJKwIBAAKCAgEA0de2Lr+DhI0HlEcl6uDrJonpUttReh57ntNNLA4AH+7lb6Le\nxQtw+byDQwlv4zId8yJ3nN5VntX5RLAlCAyOR1EPIkCYt2vnsK2lrp2PzJdETwji\nsDrBFmQHL3pl9iEU9fNru5+3ViPQEtCjyQsWEiuJHO5+ZWsRz7AeuN4Ih4k41hah\nDRw9kNJTHngxxRoGAffsYQbuj6e8GLH0sBWp+D7SN7UBcoVFQr/Ui0fa66V+4ASG\nPVvijgTw0jRL1t7e0VguGX491M0gUUXf1TWfPqezct2bQTAb8+gwe2zfYpXVrcGE\nMSZmk7oBs+AJQlsq61eorKSX8FeOp+/Rz6/FN77bV51fqZ/tQiF4jJ7ph2lTz6up\nX/nO47N+QRsMRapEHsXReY0W3VS/WthbKhkNXvQw5MrZ4xg6QSVgRA4asU23ZJ1K\nuUKgT2XsA/hL5L13kg8aa64y9azKSc2VHQe/N87MIhvzE1UD+Vn4928p9Oebq6+E\niKBoAiUUG4OgssGOrL1YxU6X03yVwCFBAJdCfx+wjOE55MZonXxafbL28yYNPEjW\nYiQZNrWuDnePCb5kS1XIoG7DZ7ta+WNutE+59pIyG6ECPZ7xJLYBquTa86hyKz59\nVdYVqAUYj+TYF81U3MWJCf1LSWEbe2Gg2Giy3dgaOCwifwXsFHn9WcSGPQkCAwEA\nAQKCAgEAwDdYKnpDfqewyaJimURuIl8x2zQK7lH96v6jMjeg5Z9vi1MlvFk+o4SK\nuF1soDDIPm7UIl2HEHfwXXr8cOMPcURPGJETUvEEylJF8i1iC4aEi+EXxVYMiPYX\nnuX/f/XNvX28saEbz0v+zT1QylfdX8eBUX8lSMFLD3PEsJKyPXT1GyafX+L+giom\n+UIgVOwBlMwFOtueqvh61CQufx1ZFIx3A5BKQxzQ1NPjXbH0VubB0XJThOEmJfFg\npyxATBLbB+g+UhvRh5xefhQDdMoplLsJJa7ZCF2JPWLzBhw0g5m8oe0hqeQDEk7Q\nQHR4BtB8ABfL6lja1M1fX3XOOvBHNaB3Si5NVWVpYbuvni26NJ2YhytdINUDdfmd\nvsMB6PE3LJI2R66tWqlbH25bTf906FBJHAS19QDYqI3IW1j13ePrUZJy+pJL/olr\nacfm9LvnO/ToLebnhJllJzqD/Tt+j+LPwMd2/j5ZvTlJuU/OPDc2kU2yEMuCWzza\npMeXb/8yiy8Hn7triw/t303pZDyP0DyKPn2i1bH1kyi4191emiamyHKgPc++Cxt8\nerSJC7tnwQqdsbz8DOhmsDLdLLbQuHVwN2oZDGuRoov70UTI97oHE8FIWSHy3m43\n9VRffnBrtn3uQFyzpa2emhYzHLckAu1d5iyM5mIENX9J386yKgECggEBAOqvqsFc\n7Gslv2PIqu5DzqEZZXYyHwAv9XOwrAEos9orxlfjE4ZA3yc9Txy03OgP8L96CTUG\nA6EF8Out6vQbg80oPvnXbZiTZFNf9v9DDvNn/UrnMGF8xtWlB4P2pZlKhS2dlApB\nSper1LgxW2jYJbBJaCyvHtCRLYuXfr+xlzqwjgx4C8W79LjJ5bxnuu+dEvCMEQz3\nRxM5Eozwd7PghuMexM6SFC7Bqezjszfk1j6LlKppbEC3OqrKJgCXZbOh5R818Ga7\nnxo1iWkwB4+N1CEbB0SsWTeRaKsRIslvZ23gBcH0/9cf3fDPk51HV8vWqLhRB4Ua\noI0xk5dSphH1bZMCggEBAOTmcb+G0S+ZW0hByLqkNfORDAjTLRxmY54mUHVQjPSH\nkfnKiRNv4WyYqSsGHMUseetcVPnpu8CP1pyO90Dzeyy7rZANtqB4YJwxftXBnfb5\nPH4tn/F28+Kz/nfU0CIOTceHYB6OjY+qCc32m+pzDfb34/hywj+nuQmzU3n9D/e1\nruxLX7sHPKtDdsuz8OMxpkN/qRm2hmm4zNpZGY1R9LMZHU4nKxe/axsuJqOJuPBX\n6PHtaUkk+DxJNyqP8KO/Ks33nGCwAdeSm7g4Z4+YMPucIXvhP9mwdYWZ1U9ytPHV\n1XjHFUrKy6NQgxta7DyEFEytiqv/mcSxH7XpEXpEbHMCggEBALdrxmRMEQcJQJVn\nX5jK7DLi23bOY4ZM9WSPH0/klPSeI+3KrxbNmttbQnqoLMM+uiWc5pdHdQyjzREW\nI7zXyGJO4zF3mtOV1uKG7U/CBGxeyQuCt0BqOij+S2prGjA9mur07qA5OWhjRuUS\nxmOiE4q9RKsvz0CpRtSD+e8uiIi5NrwuEt1fMjw+p8xhsivWMthIUIc2uJkgkQwQ\nYS33/NSD1sOwTg/hEsLvj8HOm1fU1cN+k7ncuwCC78KkkTsc/CsxiAty9j2QvC22\n+SHMco/RRRP6M9yHTCvvP6X56PdqEHXv2wkygc7VHYTeHpNU2Rb9VYhFMFhJ+BVb\n5inBDPsCggEBAKbghn8OZ8Ve9ZipNRE1FIw869wnMRUqZGfxIOlWT10a1UaZ7PN5\ntou4hGR0cVcihMQdLWqBh7rsYpcC96mnmN5U+UUzajh1amGVCBYIsQRUUlDfLGMa\nyNU3SkbMpOyfJv9XZ7D/Vp8tZTZ+Gs+DD+REdzQzXgCQY6t5zFr8Lr71+tAUZ3dv\n4EAv0BTUW8MW+FLvaDXxxu6epuJs4N8Rp+dGYQIQNi97AzfunobNqkG2pYJzBjYo\nOL2i1xA1nkeS4D8GzUAEMWObY+GbZYzfdJ6LBjJNVoJ7TkKXk1b3lolUzuvdoF1F\nmc63rM2trNq1pCL+xkF89/rY8vhpMa/E4JcCggEBAMxLBkgUOAb007szuFyl6r5S\nmSdLUUiqJZJILjzmVL6WZARAgPGZQiAUd5a95wnBTxgnm5b4rTD8q8l0cSotUS8W\nYd8PewxAH9p+9cjEwq4ddHpxns68h+6vftaTb00ZF645l1EGng3Tkmd9sS1Z5XqK\nXMMrSkeIzEuI9BUR8hPTxBXlVKMjSe4iwC9VcS8Pxl454nmDfGiJmH6xIGupmntI\n3RERF+EsxAyVy902ij0whq66wAHNWPMbgpPKDvOINTN91vIah/QNxPhXhBpbfXDe\nhBns6fArR5xeuwR5bvuSgkdvUc5P6N2DvSK8dpFHcdWNleaZbv4q+V0dwoWSANM=\n-----END RSA PRIVATE KEY-----\n",
      "cert_chain": "-----BEGIN CERTIFICATE-----\nMIIFUjCCAzqgAwIBAgIJAN7fKYoUdoHWMA0GCSqGSIb3DQEBCwUAMCIxDjAMBgNV\nBAoMBUlzdGlvMRAwDgYDVQQDDAdSb290IENBMB4XDTIyMTExNzIyMDE1NFoXDTI0\nMTExNjIyMDE1NFowRDEOMAwGA1UECgwFSXN0aW8xGDAWBgNVBAMMD0ludGVybWVk\naWF0ZSBDQTEYMBYGA1UEBwwPaXN0aW9kLWNsdXN0ZXIxMIICIjANBgkqhkiG9w0B\nAQEFAAOCAg8AMIICCgKCAgEA0de2Lr+DhI0HlEcl6uDrJonpUttReh57ntNNLA4A\nH+7lb6LexQtw+byDQwlv4zId8yJ3nN5VntX5RLAlCAyOR1EPIkCYt2vnsK2lrp2P\nzJdETwjisDrBFmQHL3pl9iEU9fNru5+3ViPQEtCjyQsWEiuJHO5+ZWsRz7AeuN4I\nh4k41hahDRw9kNJTHngxxRoGAffsYQbuj6e8GLH0sBWp+D7SN7UBcoVFQr/Ui0fa\n66V+4ASGPVvijgTw0jRL1t7e0VguGX491M0gUUXf1TWfPqezct2bQTAb8+gwe2zf\nYpXVrcGEMSZmk7oBs+AJQlsq61eorKSX8FeOp+/Rz6/FN77bV51fqZ/tQiF4jJ7p\nh2lTz6upX/nO47N+QRsMRapEHsXReY0W3VS/WthbKhkNXvQw5MrZ4xg6QSVgRA4a\nsU23ZJ1KuUKgT2XsA/hL5L13kg8aa64y9azKSc2VHQe/N87MIhvzE1UD+Vn4928p\n9Oebq6+EiKBoAiUUG4OgssGOrL1YxU6X03yVwCFBAJdCfx+wjOE55MZonXxafbL2\n8yYNPEjWYiQZNrWuDnePCb5kS1XIoG7DZ7ta+WNutE+59pIyG6ECPZ7xJLYBquTa\n86hyKz59VdYVqAUYj+TYF81U3MWJCf1LSWEbe2Gg2Giy3dgaOCwifwXsFHn9WcSG\nPQkCAwEAAaNpMGcwHQYDVR0OBBYEFIUXcQuj9KKILV23iNGtPOTUmphDMBIGA1Ud\nEwEB/wQIMAYBAf8CAQAwDgYDVR0PAQH/BAQDAgLkMCIGA1UdEQQbMBmCF2lzdGlv\nZC5pc3Rpby1zeXN0ZW0uc3ZjMA0GCSqGSIb3DQEBCwUAA4ICAQC9Va6PpBpYji3A\nkscmFUjjR8Yi8HgCwLZgs6toy8RGMejM4ANsB22Kl/cYQx9YNODTTxd3GOqAPglB\nL2iqYP0+qJWU+h8u4n2Bgaz77DKmiIhKBhozeSUGltzFFK93zFwMhVEvlTOfwhgb\n2xS1iAAAGFvPYeJSRNwfTz59mFmIYErbjWIl+3pxjen0YD5AOntW+SkpJBfz7jqf\npvDEja1uP60kjSdqy4ppj6Dlo6/AwQpM2hbn1riD0MRcE56c0SNfuygfCpj/o2iu\nmmYHGPgoiN8MXM99GyJnQ3CZhl3MHlxZ2Uy7zln6h1OR8abLtyJjueu/7qbQi/+t\nJg8B4jg3ofZ4+Te+b+nmiJ06FQ2VpQSigpGTQQbsfkEM9Nio5+TaULLXyaazizD6\nYG1uIgxT14zRLkAcc+asT941qobHbshcqabqJQ3jeIeMAENBSTwtaQaY4HupCqGz\nUkca0gimyNa4U84CRzB6qkRA2Qu8mK4HggbzmzMLCIuCg2hALNw/HCZoN5lnT6ja\nbiTqljc00xswAlxKfmNtyUFd/Obsm5kdMG1Fc/gDwqeQoauzs2lO/ZJa8+AulJ3f\nPy9b7HnuhVI513gfjC/rueZiLWiz9SHToCZ1OEBhQ0+Gn1X1fCcb2/npDHrQ3WUJ\n1mNnbjvxR/Wi2cn30cQeb7BXEUCeWA==\n-----END CERTIFICATE-----\n-----BEGIN CERTIFICATE-----\nMIIFCTCCAvGgAwIBAgIJAPW81+ib22JIMA0GCSqGSIb3DQEBCwUAMCIxDjAMBgNV\nBAoMBUlzdGlvMRAwDgYDVQQDDAdSb290IENBMB4XDTIyMTExNzIyMDE1MloXDTMy\nMTExNDIyMDE1MlowIjEOMAwGA1UECgwFSXN0aW8xEDAOBgNVBAMMB1Jvb3QgQ0Ew\nggIiMA0GCSqGSIb3DQEBAQUAA4ICDwAwggIKAoICAQDSzeKUVYad9rExNis/ufhw\ntQHd9P8SEuq9DBNJBxfo2wRt6QyCsLIekCjH0GIPLp3UWj03i9JDauPFXWvMrQTV\nyMG9sjEmhvUlIGAe6w+4fLvljLSmLQeWTVgQUxiDk7bM2OC3+e+sxOGoKDME5dp+\n9od7YuEOmc5WnNF3sFqEkf2E+KB1FQg/PpyilBJSnYLXGasb3OWJE03EOj+mC8va\ninK0u6xw81fcrlDuBHeh3meB8ud7ovY+ZAPqhRWUY4dz3CuGXN5PWCHaUSDNrzKM\nfsHnem8XnG+5Ws4Og9bavlKXf7SvJlpLrn6Y1XC3kdFiWoG4Kf1rAYACiBTH24HI\nIGrLlGXMBtEkMO/RjsV4kSJqkdSkeryvVnZaI5nGXyxvNdUzmmM3qqbS62aQA062\nB1GuIM49DB7xma5Lue1qRxBopOJVGmzcNKDpZ9+HiikReS/fl0A6Z+Sk3sbl2VP6\n/WbpZ7LuuSZkkzAbs+PgsmkQu2hGIk//Aw8xBSqJN5K9lQBrBfKfkyVdPK7WkUNG\n/KinEMmWgQ9LHzXTdeJzvD3aV6Zz1BBVWgaFny7xEbIg8+46Y58oIPfrqtsmd9tS\nv+OkyJjDvArVi4ErG72AriiY1zK1MemLScnGKWQmwCT6Y1foH3nFqJcnSfUzgCh5\nOC6Y7eWo3D1UQWGNlvkkwQIDAQABo0IwQDAdBgNVHQ4EFgQUf19JeG8G9jDhTa4l\nWDm/zyjeyc8wDwYDVR0TAQH/BAUwAwEB/zAOBgNVHQ8BAf8EBAMCAuQwDQYJKoZI\nhvcNAQELBQADggIBAJX1TUFtaS39OorE3oOKazttc30kcaMaBeva8DVVucG6OZKx\nqGOjEblmTsdqGYbJ7aoG1bz3LrdkZoViE953sVvq4Yf4ZJx7rORGtNKsWi7FAOEJ\nZIqLDLQk8LUZzNjaFxxa4urqC6tw4x5jcG086nUmIjCN0/z2dyIGozWIKUSL8URs\nYDlXqDQTEZg8qrhWisIdyO5qpkCLslTcGVl3Kq3wzEZxkky3Asc70vob9ine7O7K\nVsgb3Wkcp2FzgoSH7zvhELwePnBfAgY3UlWkqA8MSJbS9C3QdRf+nzrVNfANj/Gd\npFBdJ/blnYyqyba38oHgMFuEFSX2tXOm9LD0PB6qnVyWC2lMFZCjjQzohirgTenP\n4c5q0M4t7ZOtA+USK0v+pMzivgLdaYdjViUKcud62E19gIDQwUxWkkDySAYmrakR\nE49Ai0bx7uuLyf+5RHSWw3B9RAGNg1KBYe2ysLIh8tC1ov5xlBLnmssmK/HP619U\nYpINph/MXXHVe/VXbTTNVdhd6qM8wb48dvd3U2MXVqLSlfi51AJsugKAF0AThLCJ\nlUOgIOyMX2+UTc1Ci3t46xBbEzGWssMmyzwXkBoMJ5v/+FnNABqyE//JPNZt1/DU\ncO1F6NLHd5wXO9f2DjTrgjEqN9DgE3Kg0G5i5k2CqXvbImL6ZpZhu3A4MRs6\n-----END CERTIFICATE-----\n",
      "root_cert": "-----BEGIN CERTIFICATE-----\nMIIFCTCCAvGgAwIBAgIJAPW81+ib22JIMA0GCSqGSIb3DQEBCwUAMCIxDjAMBgNV\nBAoMBUlzdGlvMRAwDgYDVQQDDAdSb290IENBMB4XDTIyMTExNzIyMDE1MloXDTMy\nMTExNDIyMDE1MlowIjEOMAwGA1UECgwFSXN0aW8xEDAOBgNVBAMMB1Jvb3QgQ0Ew\nggIiMA0GCSqGSIb3DQEBAQUAA4ICDwAwggIKAoICAQDSzeKUVYad9rExNis/ufhw\ntQHd9P8SEuq9DBNJBxfo2wRt6QyCsLIekCjH0GIPLp3UWj03i9JDauPFXWvMrQTV\nyMG9sjEmhvUlIGAe6w+4fLvljLSmLQeWTVgQUxiDk7bM2OC3+e+sxOGoKDME5dp+\n9od7YuEOmc5WnNF3sFqEkf2E+KB1FQg/PpyilBJSnYLXGasb3OWJE03EOj+mC8va\ninK0u6xw81fcrlDuBHeh3meB8ud7ovY+ZAPqhRWUY4dz3CuGXN5PWCHaUSDNrzKM\nfsHnem8XnG+5Ws4Og9bavlKXf7SvJlpLrn6Y1XC3kdFiWoG4Kf1rAYACiBTH24HI\nIGrLlGXMBtEkMO/RjsV4kSJqkdSkeryvVnZaI5nGXyxvNdUzmmM3qqbS62aQA062\nB1GuIM49DB7xma5Lue1qRxBopOJVGmzcNKDpZ9+HiikReS/fl0A6Z+Sk3sbl2VP6\n/WbpZ7LuuSZkkzAbs+PgsmkQu2hGIk//Aw8xBSqJN5K9lQBrBfKfkyVdPK7WkUNG\n/KinEMmWgQ9LHzXTdeJzvD3aV6Zz1BBVWgaFny7xEbIg8+46Y58oIPfrqtsmd9tS\nv+OkyJjDvArVi4ErG72AriiY1zK1MemLScnGKWQmwCT6Y1foH3nFqJcnSfUzgCh5\nOC6Y7eWo3D1UQWGNlvkkwQIDAQABo0IwQDAdBgNVHQ4EFgQUf19JeG8G9jDhTa4l\nWDm/zyjeyc8wDwYDVR0TAQH/BAUwAwEB/zAOBgNVHQ8BAf8EBAMCAuQwDQYJKoZI\nhvcNAQELBQADggIBAJX1TUFtaS39OorE3oOKazttc30kcaMaBeva8DVVucG6OZKx\nqGOjEblmTsdqGYbJ7aoG1bz3LrdkZoViE953sVvq4Yf4ZJx7rORGtNKsWi7FAOEJ\nZIqLDLQk8LUZzNjaFxxa4urqC6tw4x5jcG086nUmIjCN0/z2dyIGozWIKUSL8URs\nYDlXqDQTEZg8qrhWisIdyO5qpkCLslTcGVl3Kq3wzEZxkky3Asc70vob9ine7O7K\nVsgb3Wkcp2FzgoSH7zvhELwePnBfAgY3UlWkqA8MSJbS9C3QdRf+nzrVNfANj/Gd\npFBdJ/blnYyqyba38oHgMFuEFSX2tXOm9LD0PB6qnVyWC2lMFZCjjQzohirgTenP\n4c5q0M4t7ZOtA+USK0v+pMzivgLdaYdjViUKcud62E19gIDQwUxWkkDySAYmrakR\nE49Ai0bx7uuLyf+5RHSWw3B9RAGNg1KBYe2ysLIh8tC1ov5xlBLnmssmK/HP619U\nYpINph/MXXHVe/VXbTTNVdhd6qM8wb48dvd3U2MXVqLSlfi51AJsugKAF0AThLCJ\nlUOgIOyMX2+UTc1Ci3t46xBbEzGWssMmyzwXkBoMJ5v/+FnNABqyE//JPNZt1/DU\ncO1F6NLHd5wXO9f2DjTrgjEqN9DgE3Kg0G5i5k2CqXvbImL6ZpZhu3A4MRs6\n-----END CERTIFICATE-----\n"
    },
    "wrap_info": null,
    "warnings": null,
    "auth": null
  }
```
