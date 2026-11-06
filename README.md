# DevOps Test Task

Статическая страница на nginx, собираемая в Docker-образ и запускаемая в Kubernetes.

## Состав проекта

- `Dockerfile` и `nginx.conf` — собирают и настраивают nginx на порту `8080`; процесс работает от UID/GID `101`.
- `index.html` — содержимое веб-страницы.
- `deployment.yaml` — Deployment с двумя репликами, probes, ограничениями ресурсов, Service и PodDisruptionBudget.
- `.gitlab-ci.yml` — проверяет конфигурацию образа, публикует его в GitLab Container Registry и деплоит из основной ветки при наличии `KUBE_CONFIG`.

## Локальный запуск

```sh
docker build -t static-web:local .
docker run --rm -p 8080:8080 static-web:local
curl http://localhost:8080
```

## GitLab CI/CD

Runner с тегом `shell-runner` должен иметь Docker CLI с доступом к Docker daemon и `kubectl`. В настройках проекта должен быть включён Container Registry.

Pipeline собирает образ с тегом commit SHA, проверяет конфигурацию командой `nginx -t` и отправляет образ в GitLab Container Registry. Deploy выполняется только для основной ветки, если задана защищённая переменная `KUBE_CONFIG` типа **File** с kubeconfig, имеющим права на обновление ресурсов в нужном namespace.

Перед деплоем создайте в целевом namespace secret `gitlab-registry`, чтобы кластер мог скачать приватный образ. Используйте GitLab Deploy Token с правом `read_registry`; не сохраняйте пароль в репозитории:

```sh
REGISTRY_HOST=registry.gitlab.com
kubectl create secret docker-registry gitlab-registry \
	--docker-server="$REGISTRY_HOST" \
	--docker-username="$REGISTRY_USER" \
	--docker-password="$REGISTRY_PASSWORD"
```

Для self-managed GitLab укажите в `REGISTRY_HOST` хост вашего Container Registry.

Манифест можно применить вручную, заменив базовый образ на опубликованный образ проекта:

```sh
IMAGE=registry.gitlab.com/<group>/<project>:<tag>
sed "s|image: nginx:stable|image: $IMAGE|" deployment.yaml | kubectl apply -f -
kubectl rollout status deployment/nginx-deployment
```

Service типа `ClusterIP` доступен внутри кластера по адресу `nginx-service:80`. Для внешнего доступа добавьте Ingress с TLS и настройками своего ingress-контроллера.

Проверка и диагностика:

```sh
kubectl get deployments,pods,services,pdb
kubectl get endpoints nginx-service
kubectl describe pods -l app=nginx
kubectl logs deployment/nginx-deployment
kubectl get events --sort-by=.metadata.creationTimestamp
```

## Справочник по Linux и сетевой диагностике

### Обновление пакетов в Ubuntu/Debian

`apt update` обновляет локальный индекс доступных пакетов из настроенных репозиториев. `apt upgrade` устанавливает доступные обновления для уже установленных пакетов, используя этот индекс. Обычно сначала выполняют `apt update`, затем `apt upgrade`.

### Проверка прослушиваемых портов

Чтобы проверить, слушает ли процесс нужный порт, используйте одну из команд:

```sh
ss -tuln | grep '<порт>'
netstat -tuln | grep '<порт>'
lsof -i :<порт>
```

`ss` и `netstat` показывают сетевые сокеты, а `lsof` помогает определить процесс, использующий порт.

### Диагностика сети

- `ping <адрес>` проверяет доступность узла и задержку по ICMP, если ICMP не блокируется.
- `traceroute <адрес>` показывает маршрут пакетов до узла.
- `curl <URL>` проверяет HTTP(S)-соединение и ответ сервера.
- `nc -zv <адрес> <порт>` проверяет доступность TCP-порта.
- `telnet <адрес> <порт>` также может проверить TCP-соединение, если клиент установлен.
