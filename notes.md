 Kubernetes Развертывание - Полное Руководство

На основе практического опыта развертывания приложения в production-like Kubernetes кластере.

## 🔧 Основные Проблемы и Решения

### 1. Flannel в CrashLoopBackOff

**Причины:**
- Не загружен модуль ядра `br_netfilter`
- Несовпадение Pod CIDR между кластером и конфигом Flannel
- Нет прав на запись в `/run/flannel/`

**Решение:**
```bash
# На ВСЕХ нодах
sudo modprobe br_netfilter
echo "br_netfilter" | sudo tee /etc/modules-load.d/br_netfilter.conf

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```
# Проверить CIDR
kubectl cluster-info dump | grep -i cidr
kubectl -n kube-flannel get cm kube-flannel-cfg -o jsonpath='{.data.cni-conf\.json}' | jq .
2. ImagePullBackOff / ErrImagePull
Причины:

Нет imagePullSecrets для приватного registry

Неверный адрес образа

Registry требует аутентификацию

Containerd пытается использовать HTTPS вместо HTTP

Решение:

bash
# Создать секрет
kubectl create secret docker-registry harbor-secret \
  --docker-server=192.168.1.10:90 \
  --docker-username=gitlab-ci \
  --docker-password='P@ssw0rd' \
  --namespace=your-namespace

# Добавить в deployment.yaml
spec:
  template:
    spec:
      imagePullSecrets:
      - name: harbor-secret
3. Настройка Containerd для HTTP Registry
bash
# На ВСЕХ нодах
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Добавить в конец config.toml
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."192.168.1.10:90"]
      endpoint = ["http://192.168.1.10:90"]
  [plugins."io.containerd.grpc.v1.cri".registry.configs]
    [plugins."io.containerd.grpc.v1.cri".registry.configs."192.168.1.10:90".tls]
      insecure_skip_verify = true

sudo systemctl restart containerd
🚀 Deployment Best Practices
Образец рабочего deployment.yaml
yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  namespace: hello
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      # ⚠️ ОБЯЗАТЕЛЬНО для приватного registry
      imagePullSecrets:
      - name: harbor-secret
      
      containers:
      - name: app
        # ⚠️ Используйте конкретные теги, не latest
        image: 192.168.1.10:90/hello-world/hello-app:$CI_COMMIT_SHORT_SHA
        ports:
        - containerPort: 8080
        
        # ⚠️ Пробы здоровья - увеличивайте задержки для медленных приложений
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30    # Для Flask development server
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
          
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
🌐 Service Configuration
NodePort Service (для доступа извне)
yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
  namespace: hello
spec:
  type: NodePort
  selector:
    app: hello-app    # ⚠️ Должен совпадать с labels в deployment
  ports:
    - protocol: TCP
      port: 80           # Порт сервиса внутри кластера
      targetPort: 8080   # Порт контейнера
      nodePort: 30080    # Порт на нодах (30000-32767)
Доступ по: http://<node-ip>:30080

🔑 Access & Security
Настройка kubeconfig на worker нодах
bash
# На control-plane
scp ~/.kube/config user@worker-ip:/tmp/kubeconfig

# На worker
mkdir -p ~/.kube
sudo cp /tmp/kubeconfig ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
ServiceAccount для CI/CD
bash
# Создать ServiceAccount с ограниченными правами
kubectl create serviceaccount gitlab-ci -n hello

# Создать Role и RoleBinding
kubectl create role gitlab-ci-role -n hello \
  --verb=get,list,watch,create,update,patch,delete \
  --resource=pods,deployments,services

kubectl create rolebinding gitlab-ci-binding -n hello \
  --serviceaccount=hello:gitlab-ci \
  --role=gitlab-ci-role

# Получить токен
kubectl create token gitlab-ci -n hello --duration=8760h
🔄 GitLab CI/CD Pipeline
Рабочий .gitlab-ci.yml
yaml
stages:
  - build
  - deploy

variables:
  HARBOR_HOST: 192.168.1.10:90
  IMAGE: $HARBOR_HOST/hello-world/hello-app:$CI_COMMIT_SHORT_SHA

build:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - mkdir -p /kaniko/.docker
    - |
      cat > /kaniko/.docker/config.json <<EOF
      {
        "auths": {
          "192.168.1.10:90": {
            "username": "gitlab-ci",
            "password": "P@ssw0rd"
          }
        },
        "insecure-registries": ["$HARBOR_HOST"]
      }
      EOF
    - /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      --destination "${IMAGE}"
      --insecure
      --skip-tls-verify

deploy:
  stage: deploy
  image:
    name: bitnami/kubectl:latest
    entrypoint: [""]
  script:
    # Подстановка переменных в манифест
    - export CI_COMMIT_SHORT_SHA="$CI_COMMIT_SHORT_SHA"
    - envsubst < k8s/deployment.yaml | kubectl apply -f -
    - kubectl apply -f k8s/service.yaml
🐛 Диагностика Проблем
Команды для отладки
bash
# Логи пода
kubectl logs -n hello <pod-name>
kubectl logs -n hello <pod-name> --previous

# Детали пода
kubectl describe pod -n hello <pod-name>

# События в namespace
kubectl get events -n hello --sort-by=.lastTimestamp

# Проверка сети
kubectl get pods -n kube-system -o wide | grep flannel
kubectl get pods -n kube-system -o wide | grep proxy

# Проверка сервисов
kubectl get svc -n hello -o wide
kubectl get endpoints -n hello

# Проверка нод
kubectl get nodes -o wide
kubectl describe node <node-name>
Последовательность диагностики
kubectl get pods - общий статус

kubectl describe pod - события и причины падения

kubectl logs - логи приложения

kubectl get events - события кластера

🛠 Quick Fixes
Быстрые решения частых проблем
Flannel падает:

bash
kubectl delete pod -n kube-flannel -l app=flannel
Kube-proxy падает:

bash
kubectl delete pod -n kube-system -l k8s-app=kube-proxy
Поды в ImagePullBackOff:

bash
# Проверить секрет
kubectl get secrets -n hello
# Пересоздать поды
kubectl delete pods -n hello --all
Приложение падает после запуска:

Увеличить initialDelaySeconds в пробах здоровья

Временно отключить пробы для теста

📋 Checklist при Развертывании
Flannel Running на всех нодах

Kube-proxy Running на всех нодах

Секрет harbor-secret создан

imagePullSecrets добавлен в deployment

Containerd настроен для HTTP registry

Kubeconfig доступен на worker нодах

Пробы здоровья с адекватными таймаутами

Service создан и селекторы совпадают

Образ существует в registry с правильным тегом

💡 Важные Моменты
Всегда используйте конкретные теги образов - не latest

Настраивайте пробы здоровья - без них поды будут перезапускаться

Проверяйте селекторы в Service и Deployment - они должны совпадать

Worker нодам нужен kubeconfig для доступа к API

Flannel требует br_netfilter на всех нодах


Настройка RBAC для CI/CD
yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-ci
  namespace: hello
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: hello
  name: gitlab-ci-role
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "deployments", "services", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-ci-binding
  namespace: hello
subjects:
- kind: ServiceAccount
  name: gitlab-ci
  namespace: hello
roleRef:
  kind: Role
  name: gitlab-ci-role
  apiGroup: rbac.authorization.k8s.io
Минимальный kubeconfig для CI
yaml
apiVersion: v1
kind: Config
clusters:
- name: k8s-cluster
  cluster:
    server: https://192.168.1.10:6443
    insecure-skip-tls-verify: true
contexts:
- name: gitlab-ci@hello
  context:
    cluster: k8s-cluster
    user: gitlab-ci
    namespace: hello
current-context: gitlab-ci@hello
users:
- name: gitlab-ci
  user:
    token: YOUR_TOKEN_HERE
🚀 Команды для Быстрого Старта
Инициализация кластера
bash
# На control-plane
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# На worker нодах
sudo kubeadm join 192.168.1.10:6443 --token ... --discovery-token-ca-cert-hash ...
Установка Flannel
bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
Создание namespace и секретов
bash
kubectl create namespace hello
kubectl create secret docker-registry harbor-secret \
  --docker-server=192.168.1.10:90 \
  --docker-username=gitlab-ci \
  --docker-password='P@ssw0rd' \
  --namespace=hello

  

Приватные registry требуют imagePullSecrets

HTTP registry нужно явно настраивать в containerd
