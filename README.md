# Implementación de Network Policies con Calico en Minikube (GitHub Codespaces)

Este documento describe paso a paso cómo desplegar **Calico** como plugin de red en **Minikube** dentro de un entorno de **GitHub Codespaces**, y cómo crear y probar **Network Policies** en Kubernetes.

---

## 1. Requisitos previos

Asegúrate de que tu Codespace tenga permisos de superusuario y Docker habilitado.

Instala las herramientas necesarias:

```bash
sudo apt-get update

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# calicoctl (opcional)
curl -LO https://github.com/projectcalico/calico/releases/latest/download/calicoctl-linux-amd64
sudo install calicoctl-linux-amd64 /usr/local/bin/calicoctl
```

---

## 2. Iniciar Minikube con Calico

Ejecuta Minikube usando el driver `docker` (ideal para Codespaces):

```bash
minikube start --driver=docker --cni=calico --memory=4096 --cpus=2
```

> Nota: el flag `--cni=calico` indica a Minikube que instale y configure Calico automáticamente como plugin de red.

Verifica que Calico esté corriendo correctamente:

```bash
kubectl get pods -A | grep calico
```

Debes ver pods como `calico-node` y `calico-kube-controllers` en estado `Running`.

---

## 3. Crear namespaces y aplicación de prueba

Crear dos namespaces (`app` y `test`):

```bash
kubectl create ns app
kubectl create ns test
kubectl label ns test name=test
```

Desplegar una app sencilla (nginx) en el namespace `app`:

```bash
kubectl -n app create deployment web --image=nginx
kubectl -n app expose deployment web --port=80 --target-port=80 --name web-svc
```

---

## 4. Aplicar Network Policies

### a) Política “deny all ingress”

Primero, negamos todo el tráfico entrante al namespace `app`:

```yaml
# deny-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-ingress
  namespace: app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

Aplica la política:

```bash
kubectl apply -f deny-ingress.yaml
```

---

### b) Política “allow from test namespace”

Permitimos tráfico HTTP solo desde el namespace `test`:

```yaml
# allow-from-test.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-test
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: test
    ports:
    - protocol: TCP
      port: 80
```

Aplica la política:

```bash
kubectl apply -f allow-from-test.yaml
```

---

## 5. Probar conectividad

Desde el namespace `test`, ejecuta un pod temporal con `curl`:

```bash
kubectl -n test run curlpod --rm -it --image=curlimages/curl --restart=Never --   curl -sS http://web-svc.app.svc.cluster.local:80 -m 5
```

**Resultado esperado:** el pod puede conectarse.

Ahora prueba desde el namespace `default`:

```bash
kubectl run curlpod --rm -it --image=curlimages/curl --restart=Never --   curl -sS http://web-svc.app.svc.cluster.local:80 -m 5
```

 **Resultado esperado:** la conexión debe fallar (bloqueada por la política `deny-ingress`).

---

## 6. Comandos útiles de verificación

```bash
# Estado de minikube y componentes
minikube status
kubectl get nodes
kubectl get pods -A

# Ver logs de Calico
kubectl logs -n kube-system -l k8s-app=calico-node

# Listar policies aplicadas
kubectl get networkpolicies -A
```
