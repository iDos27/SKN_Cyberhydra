# 1. Start klastra
```bash
minikube start
```
# 2. Tworzenie folderu i pliku z flagą na hoście
```bash
minikube ssh "sudo mkdir -p /var/tmp/sekretny_folder && echo 'ECSC{v0lum3_m0unt_m4st3r}'
```
# 3. Utworzenie środowiska (wklej cały blok)
```bash
cat <<EOF | kubectl apply -f -
---
apiVersion: v1
kind: Namespace
metadata:
  name: volume-flag
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: player04
  namespace: volume-flag
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: volume-flag
  name: volume-hunter-role
rules:
- apiGroups: [""]
  resources: ["pods", "pods/exec"]
  verbs: ["get", "list", "create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: volume-hunter-binding
  namespace: volume-flag
subjects:
- kind: ServiceAccount
  name: player04
  namespace: volume-flag
roleRef:
  kind: Role
  name: volume-hunter-role
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: v1
kind: Namespace
metadata:
  name: secret-agent
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: player02
  namespace: secret-agent
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: secret-agent
  name: secret-hunter-role
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-hunter-binding
  namespace: secret-agent
subjects:
- kind: ServiceAccount
  name: player02
  namespace: secret-agent
roleRef:
  kind: Role
  name: secret-hunter-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

# 4. Ukrycie flagi do Secret
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key -out /tmp/tls.crt \
  -subj "/CN=ecsc{tls_trick_L36_K0KS13^}"
```
```bash
kubectl create secret generic flag --from-literal=flag=ZWNzY3tUUllfSEFSREVSfQ== -n secret-agent
kubectl create secret generic ok --from-literal=flag=ZWNzY3t0aGlzX2lzX25vdF90aGlzX2ZsYWd9 -n secret-agent
kubectl create secret tls newtls --cert=/tmp/tls.crt --key=/tmp/tls.key -n secret-agent
```
# 5. Wygenerowanie plików kubeconfig
**Secret Agent**
```bash
export TOKEN2=$(kubectl create token player02 -n secret-agent --duration=24h)
kubectl config view --raw > config2
kubectl --kubeconfig=config2 config set-credentials player02 --token=$TOKEN2
kubectl --kubeconfig=config2 config set-context secret-context --cluster=minikube --user=player02 --namespace=secret-agent
kubectl --kubeconfig=config2 config use-context secret-context
```
**Volume hunter**
```bash
export TOKEN2=$(kubectl create token player02 -n secret-agent --duration=24h)
kubectl config view --raw > config2
kubectl --kubeconfig=config2 config set-credentials player02 --token=$TOKEN2
kubectl --kubeconfig=config2 config set-context secret-context --cluster=minikube --user=player02 --namespace=secret-agent
kubectl --kubeconfig=config2 config use-context secret-context
```
