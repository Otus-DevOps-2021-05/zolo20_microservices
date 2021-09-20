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
