# Vault with kubernetes auth backend

Vault store secret and kubernetes manage service accounts. So why not merge those features and enabling access to secrets using kubernetes service accounts:

## Permissions w/ k8s

If configured, kubenernetes by default put a jwt token that is link to a particular serviceaccount in the `/var/run/secrets/kubernetes.io/serviceaccount/token`

examples:

base64 token
```
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkZWZhdWx0LXRva2VuLXM4OGNrIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRlZmF1bHQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJlMmRiMzk1ZC0yM2M1LTExZTktYmU3ZS0wMjQyMGUyY2E1YzMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06ZGVmYXVsdCJ9.GoCmth-vW4vtsaBHleKEQRj__WBn9CCnHg6WNLTu1itLdZGHMNyUZTR9FYHJfegUyzn2c0I3RLWeIPwHBGqdwn-cfWgv3EZRT-nM09Wy6giCuqQGR3wJZIRYCQgLWEsyR81hCDOY6xt6HYyknODZwfBBhpJrEtOzlQAJAZvhnMpJsxn8CAR8kLSlw84ocO0tIK-h94qYdRb59Ks7pZlOIEs8hGlVdC9CpR1fTaxdU__kSbQ1HQR0_HD2nNy1w881bqTRvrE5z_5t1IMKgtbKqUWgUApiDvzUbqKMaptkfMnAu6Y1nHCjzTjmboy0S7JpqW1Ugid6obZnVXXnpMLKJQ```
```

plain token without trailing binary signature
```
{"alg":"RS256","kid":""}{"iss":"kubernetes/serviceaccount","kubernetes.io/serviceaccount/namespace":"kube-system","kubernetes.io/serviceaccount/secret.name":"default-token-s88ck","kubernetes.io/serviceaccount/service-account.name":"default","kubernetes.io/serviceaccount/service-account.uid":"e2db395d-23c5-11e9-be7e-02420e2ca5c3","sub":"system:serviceaccount:kube-system:default"}
```

## Lets get started

### Vault instance

First we need to deploy the vault instance

>`setup/vault/deployment`

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: vault
  name: vault
  namespace: vault
spec:
  replicas: 1
  selector:
    matchLabels:
      run: vault
  template:
    metadata:
      labels:
        run: vault
    spec:
      containers:
      - image: vault
        imagePullPolicy: Always
        name: vault
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: vault
  name: vault
  namespace: vault
spec:
  ports:
  - port: 8200
    protocol: TCP
    targetPort: 8200
  selector:
    run: vault
  type: ClusterIP
```

### Configuring kube jwt review role

Then we need to create a role for vault that has the permissions to access the kubernetes token review api to validate JWT tokens.

>vault-k8s-auth/materials/1_vault_kube_role.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: vault
  namespace: kube-system
```

Then we will need the authentication materials:

#### Linux

```sh
export K8S_VAULT_SA_SECRET=$(kubectl get serviceaccount -n kube-system vault -o json  | jq -Mcr '.secrets[0].name')

kubectl get secret -n kube-system ${K8S_VAULT_SA_SECRET} -o json | jq -Mcr '.data["token"]' | base64 -d > vault_sa.token

kubectl get secret -n kube-system ${K8S_VAULT_SA_SECRET} -o json | jq -Mcr '.data["ca.crt"]' | base64 -d > k8s_ca.crt
```

#### Mac OS

```sh
export K8S_VAULT_SA_SECRET=$(kubectl get serviceaccount -n kube-system vault -o json  | jq -Mcr '.secrets[0].name')

kubectl get secret -n kube-system ${K8S_VAULT_SA_SECRET} -o json | jq -Mcr '.data["token"]' | base64 -D > vault_sa.token

kubectl get secret -n kube-system ${K8S_VAULT_SA_SECRET} -o json | jq -Mcr '.data["ca.crt"]' | base64 -D > k8s_ca.crt
```


### Configuring auth backend

Inside the kubernetes cluster, by default, the API can be reached using this endpoint `https://kubernetes.default.svc.cluster.local`. This endpoint will be used by vault to access the JWT verification API.

First we need to enable the kubernetes vault backend plugin.

```sh
vault auth enable kubernetes
```

Then we need to configure the backend to use security materials and setup the k8s apiserver endpoint

configure backend to use our role
```
vault write auth/kubernetes/config \
  token_reviewer_jwt=@vault_sa.token \
  kubernetes_host=https://kubernetes.default.svc.cluster.local \
  kubernetes_ca_cert=@k8s_ca.crt
```

### Configuring secret review

Then we need to create a role that allows pods that used a particular service account in a particular namespace to use a particular vault policy.

```sh
vault write auth/kubernetes/role/DefaultPolicyForPod \
  bound_service_account_names=auth-demo \
  bound_service_account_namespaces=default \
  policies=default \
  ttl=1h
```

Then we can deploy a pod using the above service account and in this particular namespace.

>vault-k8s-auth/materials/2_vault_pod_try.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: auth-demo
  namespace: default
---
apiVersion: v1
kind: Pod
metadata:
  name: try
  namespace: default
spec:
  serviceAccountName: auth-demo
  containers:
    - name: try
      image: jsenon/toolingcontainer
      command:
        - sh
        - -c
        - "while true; do sleep 30; done"
```

Then you can login to vault using API or cli

```sh
curl --request POST --data '{"jwt":"content of /var/run/secrets/kubernetes.io/serviceaccount/token", "role": "DefaultPolicyForPod"}' http://vault.vault:8200/v1/auth/kubernetes/login
```