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

![test deploy1](https://github.com/vajierik/CI-CD/assets/150177457/7421912b-54fd-4667-98ed-99fd1d6b2cb8)


![test deploy1_web](https://github.com/vajierik/CI-CD/assets/150177457/db4480d8-2ea4-443d-84c8-1237fc84d5c0)



  - Удаляем тестовое разворачивание приложения в docker:

  ![delete test deploy](https://github.com/vajierik/CI-CD/assets/150177457/2f98bd8e-2d0b-45ac-9ab9-9dcf6a1806e6)

  

  - В рамках подзадачи в роли Docker registry будем использовать Dockerhub. Логинимся в Dockerhub и создаем репозиторий.
  - В нашем случае это: otr580/diplom
![dockerhub_repo](https://github.com/vajierik/CI-CD/assets/150177457/97ceae9b-8fa2-4e57-b8dc-e2e829901027)

  
  - В качестве CI/CD будем использовать Gitlab-CI
  - Создаём репозиторий в gitlab.com под проект: https://gitlab.com/vajieriy1/diplom.git
  - Создаём и настраиваем в нём гитлаб-раннер по пути Settings-CI/CD-Runners и отключаем шарэд раннеры
  - Если видим раннер зелёным цветом, значит все нормально
![create_runner](https://github.com/vajierik/CI-CD/assets/150177457/61c6f44d-1dca-4730-93f2-effa25eba359)


  - Вносим пользователей в разрешение на docker
  ```
  usermod -aG docker ubuntu
  usermod -aG docker root
  usermod -aG docker gitlab-runner
  ```
  - Создаём нужные нам переменные для хранения приватных данных и другой информации по пути Settings-CI/CD-Variables:
![variables](https://github.com/vajierik/CI-CD/assets/150177457/1e6bf93d-2e55-48f7-92cb-9609624a0c17)


  - Описываем pipeline в стандартном .gitlab-ci.yml файле состоящий из двух стадий - сборки приложения из докерфайла и выкатка - деплой приложения. Запускаем пробно pipeline, который собирает
    проект и пушит его как артефакт в докер хаб. Результат тестовой работы pipeline:
![build_push_CI](https://github.com/vajierik/CI-CD/assets/150177457/29a20680-2ed4-4738-8c78-a0bcf1c20923)


![dockerhub_repo_push_CI](https://github.com/vajierik/CI-CD/assets/150177457/b5a0600f-a77e-4388-a4fb-bf6047f542e5)



  - Для разворачивания приложения в  кластере kubernetes пишем манифесты в каталоге "kube-manifests".
  - Секретные данные кодируем и помещаем в манифест "credentials.yaml":
  ```
  echo -n 'info_to_encode' | base64
  ```
  - Для приложения создадим отдельный нэймспейс, куда будем его диплоить:
  ```
  kubectl create namespace diplom
  ```
  - Переходим в каталог с манифестами и деплоим приложение в ранее развёрнутый K8S кластер:
  ```
  kubectl apply -f . -n diplom 
  ```
![kube_apply](https://github.com/vajierik/CI-CD/assets/150177457/6e7ae366-7ab8-4ae6-8a9f-9ca3b222240b)


  - Проверяем
  ```
  kubectl get -n diplom services 
  ```

  ```
  kubectl get po -A
  ```


![kubectl_apply](https://github.com/vajierik/CI-CD/assets/150177457/98e1e350-bf8a-400f-87b1-886c42b7b02b)


![app_web](https://github.com/vajierik/CI-CD/assets/150177457/0b6da1cc-1912-433b-acc7-01908e4aaed6)



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
![tree](https://github.com/vajierik/CI-CD/assets/150177457/160d4707-3d57-4e9b-a669-8599ffbb9878)

  - Редактируем созданную helm-заглушку. В первую очередь манифесты, которые берём с предыдущих шагов
  - Развернём наш Helm chart в кластере k8s. Переходим в папку c чартом и запускаем установку с учетом данных в credentials.yaml:
  ```
  helm upgrade --install -n diplom --values templates/credentials.yaml --set service.type=NodePort app-diplom .
  ```
![helm](https://github.com/vajierik/CI-CD/assets/150177457/cddc7a31-47de-4e15-9313-0134e903c0bd)

Подзадача 3: Описываем стадию деплоя в Helm
  - Архивируем созданный helm-chart из папки выше чарта:
  ```
   helm package ./app-diplom
  ```
  - Копируем и пушим созданный архив и сам чарт в репозиторий GitLab c CI/CD
  - Разрабатываем второй стейдж ci для развёртывания приложения в кластере k8s новой версии на основании изменения тэга
  То есть у нас тригер будет - тэг.

![build-CI](https://github.com/vajierik/CI-CD/assets/150177457/bbe73bc8-cbe4-437a-925a-75f1ea2b2148)

![deploy-CI](https://github.com/vajierik/CI-CD/assets/150177457/c8225330-b89c-43a8-bcae-8e8809b80152)

![deploy-CI-po-A](https://github.com/vajierik/CI-CD/assets/150177457/2aad03ae-7f1f-4d7f-8352-1b05eb10a8c2)

![app](https://github.com/vajierik/CI-CD/assets/150177457/57fbcd2d-b010-49b1-b304-da5d2bfd228e)


Приложение доступно по внешним IP-адреса (worker или master):
  ```
 http://158.160.86.191:30036/
  ```
или
  ```
  http://158.160.83.69:30036/
  ```
