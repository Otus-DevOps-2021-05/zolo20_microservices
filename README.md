# zolo20_microservices
zolo20 microservices repository

## Задание № 12

### Что сделано

1) В Yandex Cloud создал новый инстанс для docker из образа ubuntu-1804-lts:
```bash
yc compute instance create \
  --name docker-host \
  --zone ru-central1-a \
  --network-interface subnet-name=otus-ru-central1-a,nat-ip-version=ipv4 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-1804-lts,size=15 \
  --ssh-key ~/.ssh/appuser.pub
```

2) С помощью docker-machine проинициализировал на нем docker, указав публичный IP инстанса.
```bash
docker-machine create \
  --driver generic \
  --generic-ip-address=178.154.221.250 \
  --generic-ssh-user yc-user \
  --generic-ssh-key ~/.ssh/id_rsa \
  docker-host

eval $(docker-machine env docker-host) // инициализация docker на удаленном хосте
```
Проверка инициализации:
```bash
(venv) arsenkalasov@MacBook-Air-Arsen src % docker-machine ls
NAME           ACTIVE   DRIVER    STATE     URL                          SWARM   DOCKER    ERRORS
docker-host    -        generic   Running   tcp://178.154.221.250:2376           v20.10.8
```

3) Создал [Dockerfile](docker-monolith/Dockerfile)

4) Собрал образ и запустил контейнер в Yandex Cloud с помощью docker-machine(См. пункт 2):
```bash
docker build -t reddit:latest .
docker images -a
docker run --name reddit -d --network=host reddit:latest
```
Проверяем запуск приложения по ссылке: http://178.154.221.250:9292

5) Загрузил образ reddit:latest в docker-hub с названием zolo20/otus-reddit:1.0:
```bash
docker login  # авторизуемся в docker-hub
docker tag reddit:latest zolo20/otus-reddit:1.0
docker push zolo20/otus-reddit:1.0
```

### Первое задание со *

Отписать различие команд:

```bash
 docker inspect <u_container_id>
 docker inspect <u_image_id>
```
Различие описаны в файле [docker-1.log](docker-monolith/docker-1.log)

### Второе задание со *

Автоматизируем установку нескольких инстансов docker и запуск в них контейнера с нашим приложением из docker-образа с помощью Packer, Terraform и Ansible.

Требования:

- Нужно реализовать в виде прототипа в директории /docker-monolith/infra
- Поднятие инстансов с помощью Terraform, их количество задается переменной;
- Несколько плейбуков Ansible с использованием динамического инвентори для установки докера и запуска там образа приложения;
- Шаблон пакера, который делает образ с уже установленным Docker.

1) Создал шаблон packer, с уже установленным Docker

Создание шаблон packer находится в директории [packer](docker-monolith/infra/packer)

Предустановка docker в шаблон packer производится с помощью ansible-плейбука [docker_ubuntu1804.yml](docker-monolith/infra/ansible/playbooks/docker_ubuntu1804.yml)

2) Запуск сборку шаблона

```bash
packer validate -var-file=variables.json.example docker.json
packer build -var-file=variables.json.example docker.json
```

3) Создал шаблон terraform на основе packer шаблона

Создание шаблон packer находится в директории [terraform](docker-monolith/infra/terraform)

4) Создание инстансов с помощью terraform

```bash
terraform init
terraform plan
terraform apply
```

5) Создал ansible-плейбук, который запускает контейнер с нашим приложением,
   так как docker-image не находится на локальной машине, то он автоматически выкачает его из docker-hub

Плейбук находится [app.yml](docker-monolith/infra/ansible/playbooks/app.yml)

В файл [inventory](docker-monolith/infra/ansible/inventory) прописать публичные хосты поднятые с помощью terraform в
параметр ansible-host

6) Запуск ansible-плейбук

```bash
ansible-playbook playbooks/run_app_in_docker.yml
```

## Задание № 13

### Что сделано

1) Скопировал файлы приложения в папку src из скаченного [архива](https://github.com/express42/reddit/archive/microservices.zip)

2) Подключился к хосту с docker "docker-host" в Yandex Cloud:

```bash
docker-machine create \
  --driver generic \
  --generic-ip-address=178.154.221.250 \
  --generic-ssh-user yc-user \
  --generic-ssh-key ~/.ssh/id_rsa \
  docker-host

eval $(docker-machine env docker-host) // инициализация docker на удаленном хосте
```
3) Собираем образы и качаем образ MongoDB

```bash
docker build -t zolo20/post:1.0 ./post-py
docker build -t zolo20/comment:1.0 ./comment
docker build -t zolo20/ui:1.0 ./ui
docker pull mongo:latest
```

4) Создал bridge-сеть для контейнеров reddit

```bash
# создаем сеть
docker network create reddit

# запускаем контейнеры с алиасами
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
docker run -d --network=reddit --network-alias=post zolo20/post:1.0
docker run -d --network=reddit --network-alias=comment zolo20/comment:1.0
docker run -d --network=reddit -p 9292:9292 zolo20/ui:1.0
```
### Первое задание со *

- Запустите контейнеры с другими сетевыми алиасами
- Адреса для взаимодействия контейнеров задаются через ENV-переменные внутри Dockerfile'ов
- При запуске контейнеров (docker run) задайте им переменные окружения соответствующие новым сетевым алиасам, не пересоздавая образ
- Проверьте работоспособность сервиса

При изменении сетевых алиасов мы должны переопределить и ENV-переменные Dockerfile с помощью ключа --env,
поскольку они отвечают за сетевое взаимодействие контейнеров между собой.

```bash
docker run -d --network=reddit --network-alias=new_post_db --network-alias=new_comment_db mongo:latest
docker run -d --network=reddit --network-alias=new_post --env POST_DATABASE_HOST=new_post_db zolo20/post:1.0
docker run -d --network=reddit --network-alias=new_comment --env COMMENT_DATABASE_HOST=new_comment_db  zolo20/comment:1.0
docker run -d --network=reddit -p 9292:9292 --env POST_SERVICE_HOST=new_post --env COMMENT_SERVICE_HOST=new_comment zolo20/ui:1.0
```
### Второе задание со *

- Собрать образ на основе Alpine Linux
- Уменьшить размер образа

Собрал образ [Dockerfile.1](src/ui/Dockerfile.1) для сервиса UI. На основе Alpine Linux с оптимизацией размера образа
за счет опции установки пакетов --no-cache и удаления кэша rm -rf /var/cache/apk/* (если что-то осталось).
