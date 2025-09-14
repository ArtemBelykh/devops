# Полное руководство по созданию и использованию своего Kubernetes кластера
Это пошаговое руководство проведет вас от базовых концепций до создания отказоустойчивого, масштабируемого Kubernetes кластера на ваших собственных серверах с использованием Proxmox и Fedora.

## Оглавление
Часть 1: Основы и Концепции
- Что такое Kubernetes?
- Ключевые концепции
- Реальные vs. Виртуальные серверы: Аналогия

Часть 2: Подготовка Среды на Proxmox
- Почему Proxmox + Fedora?
- Создание "Золотого" Шаблона ВМ

Часть 3: Создание Кластера
- Создание простого кластера (1 мастер, 2 воркера)
- Настройка сети кластера (Calico CNI)

Часть 4: Масштабирование и Отказоустойчивость
- Создание HA-кластера (3 мастера)
- Расширение на несколько физических серверов

Часть 5: Практическое Использование Кластера
- Деплой приложений и микросервисов
- CI/CD Пайплайн: Автоматическое обновление
- Работа со stateful-приложениями (RabbitMQ, Kafka)

Часть 6: Организация и Управление
- Пространства имен (Namespaces)
- Создание второго, изолированного кластера

## Часть 1: Основы и Концепции
### Что такое Kubernetes?
Kubernetes (K8s) — это система с открытым исходным кодом для автоматизации развертывания, масштабирования и управления контейнеризированными приложениями. Проще говоря, это "дирижер" для ваших контейнеров, который следит, чтобы они работали слаженно и надежно на группе серверов (кластере).

### Ключевые концепции
- Контейнер: Изолированная среда для вашего приложения (чаще всего Docker).
- Под (Pod): Наименьшая единица в Kubernetes. Обертка для одного или нескольких контейнеров.
- Узел (Node): Физический или виртуальный сервер, на котором работают Поды. Это "рабочая лошадка" кластера.
- Кластер (Cluster): Группа Узлов, управляемая как единое целое.
- Мастер-узел (Master Node / Control Plane): "Мозг" кластера. Принимает решения, управляет состоянием.
- Deployment: Декларативное описание того, как должно работать ваше приложение (например, "3 копии моего веб-сервера").
- Service: Предоставляет постоянный сетевой адрес для доступа к вашим Подам.

### Реальные vs. Виртуальные серверы: Аналогия
Для Kubernetes почти нет разницы, работает он на реальном "железе" или на виртуальных машинах (ВМ).
- Реальный сервер (Bare Metal) — это как отдельный частный дом. Все ресурсы ваши, максимальная производительность.
- Виртуальный сервер (ВМ) — это как квартира в многоквартирном доме. Один мощный физический сервер делится на несколько изолированных ВМ.

Использование ВМ для узлов Kubernetes — стандартная практика, так как она дает гибкость, эффективное использование ресурсов и управляемость (снэпшоты, миграция).

## Часть 2: Подготовка Среды на Proxmox
### Почему Proxmox + Fedora?
Proxmox: Мощная, бесплатная и удобная платформа для виртуализации. Позволяет легко создавать, клонировать и управлять ВМ.  
Fedora Server: Современный, стабильный дистрибутив Linux со свежими пакетами, который отлично подходит для работы с контейнерами.

### Создание "Золотого" Шаблона ВМ
Это самый важный шаг для экономии времени. Мы создаем одну ВМ, настраиваем ее и делаем из нее шаблон для быстрого клонирования.

1. Создание ВМ в Proxmox:
   - OS: Fedora Server Net-Install ISO.
   - System: Включить QEMU Guest Agent.
   - CPU: 2+ ядра, тип host.
   - Memory: 4GB+.
   - Network: Модель VirtIO.

2. Настройка ОС внутри ВМ (после установки Fedora):  
Обновление:
```bash
sudo dnf update -y
```
Отключение SWAP (Обязательно!):
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#/g' /etc/fstab
```
Настройка ядра для сети K8s:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```
Установка containerd:
```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install -y containerd.io
sudo mkdir -p /etc/containerd && containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd
```
Установка инструментов Kubernetes:
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repomd.xml.key
EOF
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```
Очистка и выключение:
```bash
sudo shutdown now
```
3. Преобразование в шаблон: В интерфейсе Proxmox ПКМ на ВМ -> Convert to template.

## Часть 3: Создание Кластера
### Создание простого кластера (1 мастер, 2 воркера)
1. Клонирование: Клонируем шаблон 3 раза: k8s-master, k8s-worker-1, k8s-worker-2.
2. Настройка: На каждой ВМ задаем уникальный hostname и статический IP-адрес.
3. Инициализация мастера (на k8s-master):
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --control-plane-endpoint=k8s-master
```
Сохраните команду kubeadm join из вывода!  
Настройте kubectl:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
4. Присоединение воркеров (на k8s-worker-1 и k8s-worker-2):
```bash
sudo kubeadm join ...
```

### Настройка сети кластера (Calico CNI)
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```
Через минуту узлы перейдут в статус Ready.

## Часть 4: Масштабирование и Отказоустойчивость
### Создание HA-кластера (3 мастера)
1. Компоненты: 3 ВМ для мастеров, 2+ ВМ для воркеров, 1 ВМ для балансировщика (HAProxy) и 1 "плавающий" IP (VIP).
2. Балансировщик: HAProxy принимает трафик на VIP:6443 и перенаправляет его на три мастер-ноды.
3. Инициализация:
- Первый мастер:
```bash
sudo kubeadm init --control-plane-endpoint "VIP:6443" --upload-certs ...
```
- Второй и третий мастер:
```bash
sudo kubeadm join VIP:6443 ... --control-plane ...
```
- Воркеры:
```bash
sudo kubeadm join VIP:6443 ...
```

### Расширение на несколько физических серверов
- Требование: Все серверы должны быть в одной локальной сети.
- Процесс: Создаете ВМ на другом сервере, присоединяете `kubeadm join`.
- Отказоустойчивость: Распределяйте мастера по разным серверам.

## Часть 5: Практическое Использование Кластера
### Деплой приложений и микросервисов
Один микросервис = Один Deployment. Обновления делаются через Rolling Update.

### CI/CD Пайплайн
1. Git Push  
2. Сборка контейнера  
3. Публикация в Registry  
4. kubectl set image  

### Stateful-приложения (RabbitMQ, Kafka)
- Используйте StatefulSet  
- RabbitMQ:
```bash
helm install my-rabbit bitnami/rabbitmq
```
- Kafka: Strimzi Operator

## Часть 6: Организация и Управление
### Namespaces
```bash
kubectl create namespace my-app
kubectl get pods -n my-app
```

### Второй кластер
- Новый диапазон IP и hostname
- kubeadm init
- Настройка контекстов:
```bash
kubectl config use-context ...
```
