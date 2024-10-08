# CI-CD

Спринт 2

ЗАДАЧА

```
1. Клонируем репозиторий, собираем его на сервере srv.
Исходники простого приложения можно взять здесь. Это простое приложение на Django с уже написанным Dockerfile. 
Приложение работает с PostgreSQL, в самом репозитории уже есть реализация docker-compose — её можно брать за 
референс при написании Helm-чарта. Необходимо склонировать репозиторий выше к себе в Git и настроить пайплайн 
с этапом сборки образа и отправки его в любой docker registry. Для пайплайнов можно использовать GitLab, 
Jenkins или GitHub Actions — кому что нравится. Рекомендуем GitLab.

2. Описываем приложение в Helm-чарт.
Описываем приложение в виде конфигов в Helm-чарте. По сути, там только два контейнера — с базой и приложением, 
так что ничего сложного в этом нет. Стоит хранить данные в БД с помощью PVC в Kubernetes.

3. Описываем стадию деплоя в Helm.
Настраиваем деплой стадию пайплайна. Применяем Helm-чарт в наш кластер. Нужно сделать так, чтобы наше приложение 
разворачивалось после сборки в Kubernetes и было доступно по бесплатному домену или на IP-адресе с выбранным портом.
Для деплоя должен использоваться свежесобранный образ. По возможности нужно реализовать сборку из тегов в Git, где 
тег репозитория в Git будет равен тегу собираемого образа. Чтобы создание такого тега запускало пайплайн на сборку 
образа c таким именем hub.docker.com/skillfactory/testapp:2.0.3.
```

РЕШЕНИЕ

Подзадача 1: Клонируем репозиторий, собираем его на сервере srv.
  - Клонируем указанный репозиторий тестового приложения.
  git clone https://github.com/vinhlee95/django-pg-docker-tutorial
  cd django-pg-docker-tutorial/
  - Дорабатываем приложение - выносим секретные данные по подключению к БД, вносим файл 
  в гитигнор. 
  - Тестово разворачиваем приложение на ноде srv в docker для дебагинга:
  ```
  sudo docker compose up -d
  ```
  Видим удачное тестовое разворачивание:

![test deploy1](https://github.com/dilitrium/screendiplom/blob/5ab6d4adde84a7dca1ea5d9ead123458d47b3734/ci/test%20deploy1.png)


![test deploy1_web](https://github.com/dilitrium/screendiplom/blob/97e6234e40e0e0be8cd56cd4b037f3302a2df83f/ci/test%20deploy-web.png)



  - Удаляем тестовое разворачивание приложения в docker:

  ![delete test deploy](https://github.com/dilitrium/screendiplom/blob/97e6234e40e0e0be8cd56cd4b037f3302a2df83f/ci/delete%20test%20deploy.png)

  

  - В рамках подзадачи в роли Docker registry будем использовать Dockerhub. Логинимся в Dockerhub и создаем репозиторий.
  - В нашем случае это: dilitrium/diplom
![dockerhub_repo](https://github.com/dilitrium/screendiplom/blob/97e6234e40e0e0be8cd56cd4b037f3302a2df83f/ci/docker-hub.png)

  
  - В качестве CI/CD будем использовать Gitlab-CI
  - Создаём репозиторий в gitlab.com под проект: https://gitlab.com/dilitrium/diplom.git
  - Создаём и настраиваем в нём гитлаб-раннер по пути Settings-CI/CD-Runners и отключаем шаред-раннеры
  - Если видим раннер зелёным цветом, значит все нормально
![create_runner](https://github.com/dilitrium/screendiplom/blob/97e6234e40e0e0be8cd56cd4b037f3302a2df83f/ci/runner.png)


  - Вносим пользователей в разрешение на docker
  ```
  usermod -aG docker ubuntu
  usermod -aG docker root
  usermod -aG docker gitlab-runner
  ```
  - Создаём нужные нам переменные для хранения приватных данных и другой информации по пути Settings-CI/CD-Variables:
![variables](https://github.com/dilitrium/screendiplom/blob/97e6234e40e0e0be8cd56cd4b037f3302a2df83f/ci/variables.png)


  - Описываем pipeline в стандартном .gitlab-ci.yml файле состоящий из двух стадий - сборки приложения из докерфайла и выкатка - деплой приложения. Запускаем пробно pipeline, который собирает
    проект и пушит его как артефакт в докер хаб. Результат тестовой работы pipeline:
![build_push_CI](https://github.com/dilitrium/screendiplom/blob/97e6234e40e0e0be8cd56cd4b037f3302a2df83f/ci/build-ci.png)


![dockerhub_repo_push_CI](https://github.com/dilitrium/screendiplom/blob/97e6234e40e0e0be8cd56cd4b037f3302a2df83f/ci/docker-hub.png)



  - Для разворачивания приложения в  кластере kubernetes пишем манифесты в каталоге "kube-manifests".
  - Секретные данные кодируем и помещаем в манифест "credentials.yaml":
  ```
  echo -n 'info_to_encode' | base64
  ```
  - Для приложения создадим отдельный нэймспейс, куда будем его деплоить:
  ```
  kubectl create namespace diplom
  ```
  - Переходим в каталог с манифестами и деплоим приложение в ранее развёрнутый K8S кластер:
  ```
  kubectl apply -f . -n diplom 
  ```
![kube_apply](https://github.com/dilitrium/screendiplom/blob/22e4735829adfe31fdf808d351c7cfa485390dbe/ci/kubectl-deploy.png)


  - Проверяем
  ```
  kubectl get -n diplom services 
  ```

  ```
  kubectl get pod -A
  ```


![kubectl_apply](https://github.com/dilitrium/screendiplom/blob/22e4735829adfe31fdf808d351c7cfa485390dbe/infr/kuber_cluster_status.png)


![app_web](https://github.com/dilitrium/screendiplom/blob/master/ci/app-kuber.png)



  - Для удаления приложения с кластера переходим в каталог с манифестами:
  ```
  kubectl delete -f . -n diplom 
  ```
Подзадача 2: Описываем приложение в Helm-чарт.
  - На базе нашего приложения м манифестов создаём Helm chart. Переходим в папку с манифестами и формируем свой чарт:
  ```
  helm create app-diplom
  tree app-diplom
  ```
![tree](https://github.com/dilitrium/screendiplom/blob/1b67269b462b57fd8c5f628a0d7872651691753b/ci/helm-tree.png)

  - Редактируем созданную helm-заглушку. В первую очередь манифесты, которые берём с предыдущих шагов
  - Развернём наш Helm chart в кластере k8s. Переходим в папку c чартом и запускаем установку с учетом данных в credentials.yaml:
  ```
  helm upgrade --install -n diplom --values templates/credentials.yaml --set service.type=NodePort app-diplom .
  ```
![helm](https://github.com/dilitrium/screendiplom/blob/c840a25eb7a38c058ddfe5f41e879717063b71eb/ci/helm-2.png)

Подзадача 3: Описываем стадию деплоя в Helm
  - Архивируем созданный helm-chart из папки выше чарта:
  ```
   helm package ./app-diplom
  ```
  - Копируем и пушим созданный архив и сам чарт в репозиторий GitLab c CI/CD
  - Разрабатываем второй стейдж ci для развёртывания приложения в кластере k8s новой версии на основании изменения тэга
  То есть у нас тригер будет - тэг.

![build-CI](https://github.com/dilitrium/screendiplom/blob/c840a25eb7a38c058ddfe5f41e879717063b71eb/ci/build-ci.png)

![deploy-CI](https://github.com/dilitrium/screendiplom/blob/c840a25eb7a38c058ddfe5f41e879717063b71eb/ci/app-deploy-ci.png)

![deploy-CI-po-A](https://github.com/dilitrium/screendiplom/blob/c840a25eb7a38c058ddfe5f41e879717063b71eb/ci/app-kuber.png)

![app](https://github.com/dilitrium/screendiplom/blob/fd691f8efc27d6097fcd304639ca3b60fe4a4a10/ci/app-kuber-2.png)


Приложение доступно по внешним IP-адреса (worker или master):
  ```
 http://84.201.147.109:30036/
  ```
или
  ```
  http://84.201.151.205:30036/
  ```
