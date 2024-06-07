step 1

vault-auth-service-account.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth
  namespace: default
---

role-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
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
  name: vault-auth
  namespace: default

  ----

vault-auth-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: vault-auth-secret
  annotations:
    kubernetes.io/service-account.name: vault-auth
type: kubernetes.io/service-account-token



step 2 

vault secrets enable -path=secret kv-v2


step 3

vault policy write myapp-kv-ro - <<EOF
  path "secret/data/myapp/*" {
    capabilities = ["create", "read", "update", "delete", "list"]
 }
EOF


step 4

vault kv put secret/myapp/config \
      username='appuser' \
      password='suP3rsec(et!' \
      ttl='30s'


step 5 run on k8 cluster and export output in external vault machine

export SA_SECRET_NAME=$(kubectl get secrets --output=json \
    | jq -r '.items[].metadata | select(.name|startswith("vault-auth-")).name')


export SA_JWT_TOKEN=$(kubectl get secret $SA_SECRET_NAME \
    --output 'go-template={{ .data.token }}' | base64 --decode)


export SA_CA_CRT=$(kubectl config view --raw --minify --flatten \
    --output 'jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)


export K8S_HOST=$(kubectl config view --raw --minify --flatten \
    --output 'jsonpath={.clusters[].cluster.server}')


    
setp 6 on vault machine

vault auth enable kubernetes

step 7 on vault machine

vault write auth/kubernetes/config \
     token_reviewer_jwt="$SA_JWT_TOKEN" \
     kubernetes_host="$K8S_HOST" \
     kubernetes_ca_cert="$SA_CA_CRT" \
     issuer="https://192.168.80.71:6443"


  step 9 manual testing on k8 using pod connectivity with vault


cat > devwebapp.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: devwebapp
  labels:
    app: devwebapp
spec:
  serviceAccountName: vault-auth
  containers:
    - name: devwebapp
      image: burtlo/devwebapp-ruby:k8s
      env:
        - name: VAULT_ADDR
          value: "https://vault.cloudsmartz.com:8200"
EOF

kubectl exec --stdin=true --tty=true devwebapp -- /bin/sh

KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)


curl --request POST \
       --data '{"jwt": "'"$KUBE_TOKEN"'", "role": "example"}' \
       https://vault.cloudsmartz.com:8200/v1/auth/kubernetes/login | python3 -m json.tool
