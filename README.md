# Домашнее задание к занятию «`Настройка приложений и управление доступом в Kubernetes`» - `Рыбянцев Павел`

------

Репозиторий содержит конфигурационные манифесты, команды генерации сертификатов и отчёт о выполнении домашнего задания. Локальный кластер развернут с помощью MicroK8s.

---

## 📂 Структура репозитория

```text
k8s-config-rbac-homework/
├── .gitignore
├── README.md
├── task1/
│   ├── configmap-web.yaml
│   └── deployment.yaml
├── task2/
│   ├── secret-tls.yaml
│   └── ingress-tls.yaml
└── task3/
    ├── role-pod-reader.yaml
    └── rolebinding-developer.yaml
```

---

## 🛠 Задание 1: Работа с ConfigMaps

### Исходные файлы:
* Манифест ConfigMap: [task1/configmap-web.yaml](task1/configmap-web.yaml)
* Манифест деплоймента: [task1/deployment.yaml](task1/deployment.yaml)

### Описание решения:
1. Создан ConfigMap `web-content`, содержащий кастомную веб-страницу `index.html`.
2. Развернут Deployment `web-app` с двумя контейнерами в одном поде: `nginx:latest` (веб-сервер) и `wbitt/network-multitool:latest` (технический контейнер отладки). Конфликт портов устранен пробросом переменной `HTTP_PORT: "8080"` для multitool.
3. Раздел с веб-страницей из ConfigMap успешно смонтирован в контейнер `nginx` по пути `/usr/share/nginx/html`.

### Проверка работоспособности:
Запрос к веб-серверу `nginx` выполнен изнутри соседнего контейнера `multitool`:
```bash
kubectl exec -it deployment/web-app -c multitool -- curl http://localhost:80
```
*Вставьте сюда скриншот вывода команды curl с кодом HTML-страницы приветствия*

---

## 🔐 Задание 2: Настройка HTTPS с Secrets

### Исходные файлы:
* Манифест секретов: [task2/secret-tls.yaml](task2/secret-tls.yaml)
* Манифест Ingress: [task2/ingress-tls.yaml](task2/ingress-tls.yaml)

### Команды генерации сертификатов:
```bash
# 1. Генерация самоподписанного SSL-сертификата и ключа для домена (в одну строчку для Bash)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj '/CN=://example.com'

# 2. Автоматическая сборка K8s Secret манифеста из полученных файлов сертификата
kubectl create secret tls myapp-tls-secret --key tls.key --cert tls.crt --dry-run=client -o yaml > task2/secret-tls.yaml
```

### Описание решения:
1. Сгенерирован самоподписанный SSL-сертификат для домена `://example.com`. На его основе создан объект `Secret` типа `kubernetes.io/tls`.
2. Деплоймент `web-app` опубликован наружу кластера с помощью сервиса `web-app-svc` на порту 80.
3. Настроен Ingress-маршрутизатор `myapp-ingress-tls` (класс `nginx`), использующий созданный секрет для терминации SSL-трафика.
4. Доступ к защищенному порту на локальной хост-машине обеспечен через проброс портов: `sudo microk8s kubectl port-forward -n ingress-nginx deployment/ingress-nginx-controller 443:443`.

### Проверка работоспособности:
Запрос выполнен с локальной машины по протоколу HTTPS с игнорированием самоподписанного статуса сертификата (`-k`) и подменой заголовка `Host`:
```bash
curl -k -H "Host: ://example.com" https://localhost:443
```
*Вставьте сюда скриншот успешного curl -k ответа через 443 порт*

---

## 🎛 Задание 3: Настройка RBAC

### Исходные файлы:
* Манифест роли: [task3/role-pod-reader.yaml](task3/role-pod-reader.yaml)
* Манифест привязки роли: [task3/rolebinding-developer.yaml](task3/rolebinding-developer.yaml)

### Команды генерации сертификатов пользователя:
```bash
# 1. Генерация приватного ключа пользователя developer
openssl genrsa -out developer.key 2048

# 2. Создание запроса на подпись сертификата (CSR)
openssl req -new -key developer.key -out developer.csr -subj '/CN=developer'

# 3. Подпись сертификата корневым центром сертификации (CA) локального кластера MicroK8s
sudo openssl x509 -req -in developer.csr -CA /var/snap/microk8s/current/certs/ca.crt -CAkey /var/snap/microk8s/current/certs/ca.key -CAcreateserial -out developer.crt -days 365
```

### Описание решения:
1. В кластере активирован системный плагин контроля доступа на основе ролей: `microk8s enable rbac`.
2. Сгенерирован и подписан клиентский SSL-сертификат для пользователя `developer`.
3. Создана роль `Role` с именем `pod-viewer`, предоставляющая права исключительно на просмотр подов (`get`, `list`, `watch`) и чтение их системных логов (`pods/log`).
4. Объект `RoleBinding` с именем `rolebinding-developer` привязал созданную роль к субъекту `developer`.

### Проверка ограничения прав (Имитация запросов через флаг --as):

**1. Проверка разрешенной операции (Просмотр списка подов):**
```bash
kubectl get pods --as=developer
```
*Вставьте сюда скриншот успешного вывода списка подов от имени разработчика*

**2. Проверка запрещенной операции (Просмотр списка сетевых сервисов):**
```bash
kubectl get svc --as=developer
```
*Вставьте сюда скриншот с ошибкой доступа Error from server (Forbidden): services is forbidden...*
