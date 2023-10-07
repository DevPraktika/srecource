**Предупреждение. Хранить пароли в репозитории не безопасно. Пренебрегаем в учебных целях.**

## Предварительные настройки

В https://hub.cloud.mts.ru созданы шесть виртуальных машин на которых будет разворачиваться кластер. Публичный IP должна иметь только одна ВМ, на остальные будем ходить через нее.
На управляющем хосту, это отельная машина с Linux, приватный SSH ключ для доступа к этим ВМ `~/.ssh/id_rsa`. Установлены python3, pip, необходимые бинарники, kubectl, helm, git, psql.
Имеется доступ в Kubernetes, конфиг лежит в `~/.kube/config`.
 

## Развертывание роли PostgreSQL High-Availability Cluster

Клонируем репозиторий с ролью кластера БД:

    git clone https://github.com/DevPraktika/postgresql_cluster.git

Устанавливаем в виртуальное окружение ansible и другие требуемые компоненты:

    python3 -m venv venv
    source venv/bin/activate
    pip install -r requirements.txt
    ansible --version
    eval "$(ssh-agent -s)"
    ssh-add

В файле инвентаря ansible - inventory, указываем IP адреса ВМ, ProxyCommand для подключения к ВМ без публичного IP.
При необходимости изменяем файл `vars/main.yml`, указываем нужные нам параметры.

Проверяем подключение к ВМ:

    ansible -m ping all

И запускаем развертывание роли:

    ansible-playbook deploy_pgcluster.yml

Далее команды для значений по умолчанию в `vars/main.yml`, текущие IP адреса, при необходимости нужно скорректировать.

Проверяем подключение к кластеру БД:

    PGPASSWORD=hide-postgres-pass psql -h 77.105.185.88 -p 5000 -d postgres -U postgres -c "SELECT version();"

В репозитории имеется скрипт инициализации БД `db_init.sh`, добавляет пользователя, базу данных, таблицы, тестовые данные.

## Установка Helm чарта приложения srecource

Клонируем репозиторий с чартом:

    git clone https://github.com/DevPraktika/srecource.git

Для целей разработки и тестирования в чарте реализована возможность запуска сервера Postgresql в Kubernetes, это режим по умолчанию.
При использовании внешней базы нужно изменить *postgresql.enabled* на *false* и задать параметры подключения к внешнему Postgresql.
Настроить чарт можно изменив параметры в файле `srecource/values.yaml` или его копии, указав ее при запуске helm.
А можно переопределить параметры в командной строке, например, так:

    helm upgrade --install srecource srecource --wait --timeout 300s --atomic --debug \
    	--set dotnetEnv=Production \
    	--set replicaCount=3 \
            --set externalPostgresql.host=77.105.185.88 \
            --set externalPostgresql.user=sre_user \
            --set externalPostgresql.password=sre_password \
            --set externalPostgresql.database=sredatabase \
            --set postgresql.enabled=false

Здесь используем *upgrade --install* для универсальности, одна команда установки и обновления. Еще для удобства добавлено ожидание успешной установки или обновления, и откат на предыдущее состояние при неудаче.

После успешной установки проверяем, обратившись к api приложения:

    curl -D- -N --silent --resolve 'api.student65.sre:80:91.185.85.213' http://api.student65.sre/Cities/2 | head -n 10
    curl -D- -N --silent --resolve 'api.student65.sre:80:91.185.85.213' http://api.student65.sre/Cities | head -n 10
    curl -X POST -D- -N --silent --resolve 'api.student65.sre:80:91.185.85.213' http://api.student65.sre/Cities -H 'Content-Type: application/json' -d '{"id":10,"name":"Omsk"}'

#### Основные параметры чарта
| Параметр | Описание | По умолчанию |
|--|--|--|
| dotnetEnv | Значение DOTNET_ENVIRONMENT приложения | Development |
| replicaCount | Число реплик приложения | 1 |
| livenessProbe | Проверка живо ли приложение | path: /healthz/live  <br> initialDelaySeconds: 10  <br> periodSeconds: 20 |
| readinessProbe | Проба готовности принимать траффик | path: /healthz/ready <br> initialDelaySeconds: 10  <br> periodSeconds: 20 |
| service.port | Порт приложения | 80 |
| ingress.enabled | Маршрутизации внешнего трафика | true |
| ingress.hosts.host | Внешнее имя приложения | api.student65.sre |
| resources | Ограничение ресурсов | limits.cpu: 100m <br> limits.memory: 128Mi <br> requests.cpu: 50m <br> requests.memory: 64Mi |
| externalPostgresql.host  | Хост внешнего сервера БД <br> Используется когда `postgresql.enabled=false` | srecource-postgresql |
| externalPostgresql.port | Порт внешнего сервера БД | 5000 |
| externalPostgresql.user | Имя пользователя внешнего сервера БД | sre_user |
| externalPostgresql.password | Пароль внешнего сервера БД | "" |
| externalPostgresql.database | Имя базы внешнего сервера БД | sredatabase |
| postgresql.enabled | Разворачивать внутренний сервер БД <br> Если false используется внешний сервер БД | true | 
