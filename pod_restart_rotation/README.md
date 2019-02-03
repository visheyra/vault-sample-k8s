# liveness related to security materials

## TLDR

When using security materials, we want our system to be aware when those materials become revoked and to restart the pull of those materials process. The target here is to use kubernetes [liveness probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) feature to achieve such goal.

## Limitations

As stated [here](https://github.com/kubernetes/kubernetes/issues/37218) kubernetes does not support multiple liveness probes definitions so if you want to add another liveness probes to your pod you will have to add them to a custom script.

Due to this limitation, this is mandatory for your pod to run command based liveness probe because observed security materials are stored within the docker container (certificate, keys, ...)

But since we use command based liveness, multiple liveness check can be achieved using a bash (or other) script that will check the materials and also other user defined liveness probes.

## Materials

### Certificate

In that case we are only focused on the revoked / outdated security materials so no interractions with vault are shown. An expired certificate is avaialble for this test from `expired.badssl.com`. The certificate is shown below:

> Certificate

```
-----BEGIN CERTIFICATE-----
MIIE8DCCAtigAwIBAgIJAM28Wkrsl2exMA0GCSqGSIb3DQEBCwUAMH8xCzAJBgNV
BAYTAlVTMRMwEQYDVQQIDApDYWxpZm9ybmlhMRYwFAYDVQQHDA1TYW4gRnJhbmNp
c2NvMQ8wDQYDVQQKDAZCYWRTU0wxMjAwBgNVBAMMKUJhZFNTTCBJbnRlcm1lZGlh
dGUgQ2VydGlmaWNhdGUgQXV0aG9yaXR5MB4XDTE2MDgwODIxMTcwNVoXDTE4MDgw
ODIxMTcwNVowgagxCzAJBgNVBAYTAlVTMRMwEQYDVQQIDApDYWxpZm9ybmlhMRYw
FAYDVQQHDA1TYW4gRnJhbmNpc2NvMTYwNAYDVQQKDC1CYWRTU0wgRmFsbGJhY2su
IFVua25vd24gc3ViZG9tYWluIG9yIG5vIFNOSS4xNDAyBgNVBAMMK2JhZHNzbC1m
YWxsYmFjay11bmtub3duLXN1YmRvbWFpbi1vci1uby1zbmkwggEiMA0GCSqGSIb3
DQEBAQUAA4IBDwAwggEKAoIBAQDCBOz4jO4EwrPYUNVwWMyTGOtcqGhJsCK1+ZWe
sSssdj5swEtgTEzqsrTAD4C2sPlyyYYC+VxBXRMrf3HES7zplC5QN6ZnHGGM9kFC
xUbTFocnn3TrCp0RUiYhc2yETHlV5NFr6AY9SBVSrbMo26r/bv9glUp3aznxJNEx
tt1NwMT8U7ltQq21fP6u9RXSM0jnInHHwhR6bCjqN0rf6my1crR+WqIW3GmxV0Tb
ChKr3sMPR3RcQSLhmvkbk+atIgYpLrG6SRwMJ56j+4v3QHIArJII2YxXhFOBBcvm
/mtUmEAnhccQu3Nw72kYQQdFVXz5ZD89LMOpfOuTGkyG0cqFAgMBAAGjRTBDMAkG
A1UdEwQCMAAwNgYDVR0RBC8wLYIrYmFkc3NsLWZhbGxiYWNrLXVua25vd24tc3Vi
ZG9tYWluLW9yLW5vLXNuaTANBgkqhkiG9w0BAQsFAAOCAgEAsuFs0K86D2IB20nB
QNb+4vs2Z6kECmVUuD0vEUBR/dovFE4PfzTr6uUwRoRdjToewx9VCwvTL7toq3dd
oOwHakRjoxvq+lKvPq+0FMTlKYRjOL6Cq3wZNcsyiTYr7odyKbZs383rEBbcNu0N
c666/ozs4y4W7ufeMFrKak9UenrrPlUe0nrEHV3IMSF32iV85nXm95f7aLFvM6Lm
EzAGgWopuRqD+J0QEt3WNODWqBSZ9EYyx9l2l+KI1QcMalG20QXuxDNHmTEzMaCj
4Zl8k0szexR8rbcQEgJ9J+izxsecLRVp70siGEYDkhq0DgIDOjmmu8ath4yznX6A
pYEGtYTDUxIvsWxwkraBBJAfVxkp2OSg7DiZEVlMM8QxbSeLCz+63kE/d5iJfqde
cGqX7rKEsVW4VLfHPF8sfCyXVi5sWrXrDvJm3zx2b3XToU7EbNONO1C85NsUOWy4
JccoiguV8V6C723IgzkSgJMlpblJ6FVxC6ZX5XJ0ZsMI9TIjibM2L1Z9DkWRCT6D
QjuKbYUeURhScofQBiIx73V7VXnFoc1qHAUd/pGhfkCUnUcuBV1SzCEhjiwjnVKx
HJKvc9OYjJD0ZuvZw9gBrY7qKyBX8g+sglEGFNhruH8/OhqrV8pBXX/EWY0fUZTh
iywmc6GTT7X94Ze2F7iB45jh7WQ=
-----END CERTIFICATE-----
```

> Digest Certificate

```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 14824823351240255409 (0xcdbc5a4aec9767b1)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, ST=California, L=San Francisco, O=BadSSL, CN=BadSSL Intermediate Certificate Authority
        Validity
            Not Before: Aug  8 21:17:05 2016 GMT
            Not After : Aug  8 21:17:05 2018 GMT
        Subject: C=US, ST=California, L=San Francisco, O=BadSSL Fallback. Unknown subdomain or no SNI., CN=badssl-fallback-unknown-subdomain-or-no-sni
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c2:04:ec:f8:8c:ee:04:c2:b3:d8:50:d5:70:58:
                    cc:93:18:eb:5c:a8:68:49:b0:22:b5:f9:95:9e:b1:
                    2b:2c:76:3e:6c:c0:4b:60:4c:4c:ea:b2:b4:c0:0f:
                    80:b6:b0:f9:72:c9:86:02:f9:5c:41:5d:13:2b:7f:
                    71:c4:4b:bc:e9:94:2e:50:37:a6:67:1c:61:8c:f6:
                    41:42:c5:46:d3:16:87:27:9f:74:eb:0a:9d:11:52:
                    26:21:73:6c:84:4c:79:55:e4:d1:6b:e8:06:3d:48:
                    15:52:ad:b3:28:db:aa:ff:6e:ff:60:95:4a:77:6b:
                    39:f1:24:d1:31:b6:dd:4d:c0:c4:fc:53:b9:6d:42:
                    ad:b5:7c:fe:ae:f5:15:d2:33:48:e7:22:71:c7:c2:
                    14:7a:6c:28:ea:37:4a:df:ea:6c:b5:72:b4:7e:5a:
                    a2:16:dc:69:b1:57:44:db:0a:12:ab:de:c3:0f:47:
                    74:5c:41:22:e1:9a:f9:1b:93:e6:ad:22:06:29:2e:
                    b1:ba:49:1c:0c:27:9e:a3:fb:8b:f7:40:72:00:ac:
                    92:08:d9:8c:57:84:53:81:05:cb:e6:fe:6b:54:98:
                    40:27:85:c7:10:bb:73:70:ef:69:18:41:07:45:55:
                    7c:f9:64:3f:3d:2c:c3:a9:7c:eb:93:1a:4c:86:d1:
                    ca:85
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints: 
                CA:FALSE
            X509v3 Subject Alternative Name: 
                DNS:badssl-fallback-unknown-subdomain-or-no-sni
    Signature Algorithm: sha256WithRSAEncryption
         b2:e1:6c:d0:af:3a:0f:62:01:db:49:c1:40:d6:fe:e2:fb:36:
         67:a9:04:0a:65:54:b8:3d:2f:11:40:51:fd:da:2f:14:4e:0f:
         7f:34:eb:ea:e5:30:46:84:5d:8d:3a:1e:c3:1f:55:0b:0b:d3:
         2f:bb:68:ab:77:5d:a0:ec:07:6a:44:63:a3:1b:ea:fa:52:af:
         3e:af:b4:14:c4:e5:29:84:63:38:be:82:ab:7c:19:35:cb:32:
         89:36:2b:ee:87:72:29:b6:6c:df:cd:eb:10:16:dc:36:ed:0d:
         73:ae:ba:fe:8c:ec:e3:2e:16:ee:e7:de:30:5a:ca:6a:4f:54:
         7a:7a:eb:3e:55:1e:d2:7a:c4:1d:5d:c8:31:21:77:da:25:7c:
         e6:75:e6:f7:97:fb:68:b1:6f:33:a2:e6:13:30:06:81:6a:29:
         b9:1a:83:f8:9d:10:12:dd:d6:34:e0:d6:a8:14:99:f4:46:32:
         c7:d9:76:97:e2:88:d5:07:0c:6a:51:b6:d1:05:ee:c4:33:47:
         99:31:33:31:a0:a3:e1:99:7c:93:4b:33:7b:14:7c:ad:b7:10:
         12:02:7d:27:e8:b3:c6:c7:9c:2d:15:69:ef:4b:22:18:46:03:
         92:1a:b4:0e:02:03:3a:39:a6:bb:c6:ad:87:8c:b3:9d:7e:80:
         a5:81:06:b5:84:c3:53:12:2f:b1:6c:70:92:b6:81:04:90:1f:
         57:19:29:d8:e4:a0:ec:38:99:11:59:4c:33:c4:31:6d:27:8b:
         0b:3f:ba:de:41:3f:77:98:89:7e:a7:5e:70:6a:97:ee:b2:84:
         b1:55:b8:54:b7:c7:3c:5f:2c:7c:2c:97:56:2e:6c:5a:b5:eb:
         0e:f2:66:df:3c:76:6f:75:d3:a1:4e:c4:6c:d3:8d:3b:50:bc:
         e4:db:14:39:6c:b8:25:c7:28:8a:0b:95:f1:5e:82:ef:6d:c8:
         83:39:12:80:93:25:a5:b9:49:e8:55:71:0b:a6:57:e5:72:74:
         66:c3:08:f5:32:23:89:b3:36:2f:56:7d:0e:45:91:09:3e:83:
         42:3b:8a:6d:85:1e:51:18:52:72:87:d0:06:22:31:ef:75:7b:
         55:79:c5:a1:cd:6a:1c:05:1d:fe:91:a1:7e:40:94:9d:47:2e:
         05:5d:52:cc:21:21:8e:2c:23:9d:52:b1:1c:92:af:73:d3:98:
         8c:90:f4:66:eb:d9:c3:d8:01:ad:8e:ea:2b:20:57:f2:0f:ac:
         82:51:06:14:d8:6b:b8:7f:3f:3a:1a:ab:57:ca:41:5d:7f:c4:
         59:8d:1f:51:94:e1:8b:2c:26:73:a1:93:4f:b5:fd:e1:97:b6:
         17:b8:81:e3:98:e1:ed:64
```

## Liveness definition

### Certificate expired

We will here write a bash script that will check if the certificate is expired. Since we do not want a service disruption to occur (when the security materials will become outdated) we will have to check if the certificate will be outdated in the **next minute**.

> outdated.sh

```bash
openssl x509 -checkend 60 -nnout -in /tmp/cert.pem
```

This snippet check if the certificate has expired or will expire in the next minute.

We then need to make this script available to the container. A configmap will allows us to do so.

```yaml
kind: ConfigMap
metadata:
  name: liveness-script
data:
  livenessScript.sh: |
    openssl x509 -checkend 60 -noout -in /tmp/cert.pem
```

To provide the cert to our container (for demo purpose) we will make it available using another configmap.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: outdated-certificate
data:
  cert.pem: |
    -----BEGIN CERTIFICATE-----
    MIIE8DCCAtigAwIBAgIJAM28Wkrsl2exMA0GCSqGSIb3DQEBCwUAMH8xCzAJBgNV
    BAYTAlVTMRMwEQYDVQQIDApDYWxpZm9ybmlhMRYwFAYDVQQHDA1TYW4gRnJhbmNp
    c2NvMQ8wDQYDVQQKDAZCYWRTU0wxMjAwBgNVBAMMKUJhZFNTTCBJbnRlcm1lZGlh
    dGUgQ2VydGlmaWNhdGUgQXV0aG9yaXR5MB4XDTE2MDgwODIxMTcwNVoXDTE4MDgw
    ODIxMTcwNVowgagxCzAJBgNVBAYTAlVTMRMwEQYDVQQIDApDYWxpZm9ybmlhMRYw
    FAYDVQQHDA1TYW4gRnJhbmNpc2NvMTYwNAYDVQQKDC1CYWRTU0wgRmFsbGJhY2su
    IFVua25vd24gc3ViZG9tYWluIG9yIG5vIFNOSS4xNDAyBgNVBAMMK2JhZHNzbC1m
    YWxsYmFjay11bmtub3duLXN1YmRvbWFpbi1vci1uby1zbmkwggEiMA0GCSqGSIb3
    DQEBAQUAA4IBDwAwggEKAoIBAQDCBOz4jO4EwrPYUNVwWMyTGOtcqGhJsCK1+ZWe
    sSssdj5swEtgTEzqsrTAD4C2sPlyyYYC+VxBXRMrf3HES7zplC5QN6ZnHGGM9kFC
    xUbTFocnn3TrCp0RUiYhc2yETHlV5NFr6AY9SBVSrbMo26r/bv9glUp3aznxJNEx
    tt1NwMT8U7ltQq21fP6u9RXSM0jnInHHwhR6bCjqN0rf6my1crR+WqIW3GmxV0Tb
    ChKr3sMPR3RcQSLhmvkbk+atIgYpLrG6SRwMJ56j+4v3QHIArJII2YxXhFOBBcvm
    /mtUmEAnhccQu3Nw72kYQQdFVXz5ZD89LMOpfOuTGkyG0cqFAgMBAAGjRTBDMAkG
    A1UdEwQCMAAwNgYDVR0RBC8wLYIrYmFkc3NsLWZhbGxiYWNrLXVua25vd24tc3Vi
    ZG9tYWluLW9yLW5vLXNuaTANBgkqhkiG9w0BAQsFAAOCAgEAsuFs0K86D2IB20nB
    QNb+4vs2Z6kECmVUuD0vEUBR/dovFE4PfzTr6uUwRoRdjToewx9VCwvTL7toq3dd
    oOwHakRjoxvq+lKvPq+0FMTlKYRjOL6Cq3wZNcsyiTYr7odyKbZs383rEBbcNu0N
    c666/ozs4y4W7ufeMFrKak9UenrrPlUe0nrEHV3IMSF32iV85nXm95f7aLFvM6Lm
    EzAGgWopuRqD+J0QEt3WNODWqBSZ9EYyx9l2l+KI1QcMalG20QXuxDNHmTEzMaCj
    4Zl8k0szexR8rbcQEgJ9J+izxsecLRVp70siGEYDkhq0DgIDOjmmu8ath4yznX6A
    pYEGtYTDUxIvsWxwkraBBJAfVxkp2OSg7DiZEVlMM8QxbSeLCz+63kE/d5iJfqde
    cGqX7rKEsVW4VLfHPF8sfCyXVi5sWrXrDvJm3zx2b3XToU7EbNONO1C85NsUOWy4
    JccoiguV8V6C723IgzkSgJMlpblJ6FVxC6ZX5XJ0ZsMI9TIjibM2L1Z9DkWRCT6D
    QjuKbYUeURhScofQBiIx73V7VXnFoc1qHAUd/pGhfkCUnUcuBV1SzCEhjiwjnVKx
    HJKvc9OYjJD0ZuvZw9gBrY7qKyBX8g+sglEGFNhruH8/OhqrV8pBXX/EWY0fUZTh
    iywmc6GTT7X94Ze2F7iB45jh7WQ=
    -----END CERTIFICATE-----
```

We will then build a pod that:
- will mounts those configmaps as volumes
- has a configured liveness probe that uses the script from the first configmap to check the certificate from the second configmap

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: docker.io/visheyra/ubuntu-with-openssl
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 36000']
    volumeMounts:
    - name: liveness
      mountPath: /live
    - name: cert
      mountPath: /tmp
    livenessProbe:
      exec:
        command:
        - sh
        - -c
        - /live/livenessScript.sh
      initialDelaySeconds: 5
      periodSeconds: 60 # Here we are telling kube to run the liveness probe once every 60 seconds
  volumes:
  - name: liveness # mounting liveness script
    configMap:
      name: liveness-script
  - name: cert # mounting certificate for demo
    configmap:
      name: outdated-certificate
```

And that's all folks, your pod should crash after the first minute.