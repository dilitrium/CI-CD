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

![test deploy1](https://github.com/vajierik/CI-CD/assets/150177457/fa0a36d5-0111-49e4-975f-985f672041fb)

![test deploy1_web](https://github.com/vajierik/CI-CD/assets/150177457/1bd3681c-e9e1-48ce-ba61-5ce6701772ad)


  - Удаляем тестовое разворачивание приложения в docker:

  ![delete test deploy](https://github.com/vajierik/CI-CD/assets/150177457/d49295ab-0a97-44a9-ba35-2d910e20bdc5)
  

  - В рамках подзадачи в роли Docker registry будем использовать Dockerhub. Логинимся в Dockerhub и создаем репозиторий.
  - В нашем случае это: otr580/diplom
![dockerhub_repo](https://github.com/vajierik/CI-CD/assets/150177457/2c04a30e-ace5-40e3-af4d-0616f2c947e8)
  
  - В качестве CI/CD будем использовать Gitlab-CI
  - Создаём репозиторий в gitlab.com под проект: https://gitlab.com/vajieriy1/diplom.git
  - Создаём и настраиваем в нём гитлаб-раннер по пути Settings-CI/CD-Runners и отключаем шарэд раннеры
  - Если видим раннер зелёным цветом, значит все нормально
![create_runner](https://github.com/vajierik/CI-CD/assets/150177457/4183b4be-2cb9-4874-bb0f-2c7dfc7a2af4)

  - Вносим пользователей в разрешение на docker
  ```
  usermod -aG docker ubuntu
  usermod -aG docker root
  usermod -aG docker gitlab-runner
  ```
  - Создаём нужные нам переменные для хранения приватных данных и другой информации по пути Settings-CI/CD-Variables:
![variables](https://github.com/vajierik/CI-CD/assets/150177457/0c736342-4230-48e1-8247-20789879d6b5)

  - Описываем pipeline в стандартном .gitlab-ci.yml файле состоящий из двух стадий - сборки приложения из докерфайла и выкатка - деплой приложения. Запускаем пробно pipeline, который собирает
    проект и пушит его как артефакт в докер хаб. Результат тестовой работы pipeline:
![build_push_CI](https://github.com/vajierik/CI-CD/assets/150177457/a05b538a-15b8-40ba-910b-0a9f99c84b0d)

![dockerhub_repo_push_CI](https://github.com/vajierik/CI-CD/assets/150177457/46f5957f-46ee-43a2-8fde-05aa3b2a7eae)


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
![kube_apply](https://github.com/vajierik/CI-CD/assets/150177457/250d805b-9745-4d25-a523-7b52e7bdfce8)

  - Проверяем
  ```
  kubectl get -n diplom services 
  ```

  ```
  kubectl get po -A
  ```


![kubectl_apply](https://github.com/vajierik/CI-CD/assets/150177457/acde1087-2bc8-4157-b4e1-2304e2fba1e3)


![app_web](https://github.com/vajierik/CI-CD/assets/150177457/e7d615b2-cd82-4433-aa54-217c196371e8)


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
![tree](https://github.com/vajierik/CI-CD/assets/150177457/8c3076e9-c447-489c-90d8-4e65b4b01a08)

  - Редактируем созданную helm-заглушку. В первую очередь манифесты, которые берём с предыдущих шагов
  - Развернём наш Helm chart в кластере k8s. Переходим в папку c чартом и запускаем установку с учетом данных в credentials.yaml:
  ```
  helm upgrade --install -n diplom --values templates/credentials.yaml --set service.type=NodePort app-diplom .
  ```
  
Подзадача 3: Описываем стадию деплоя в Helm
  - Архивируем созданный helm-chart из папки выше чарта:
  ```
   helm package ./app-diplom
  ```
  - Копируем и пушим созданный архив и сам чарт в репозиторий GitLab c CI/CD
  - Разрабатываем второй стейдж ci для развёртывания приложения в кластере k8s новой версии на основании изменения тэга
  То есть у нас тригер будет - тэг.

![Helm-deploy-2](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/2bb8f713-4330-4f46-acb8-b2d9c285eeec)

![Helm-deploy-3](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/eb34faf5-1f3f-43d2-b559-fbc37dff0d29)

![Helm-deploy](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/4b30cc55-2163-41a7-b5df-4c34b01cc7db)

![Helm-deploy-1](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/f800619a-da0a-4771-9b7e-d54ab7c4bdbb)

![pipeline](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/30afd39c-881a-4d83-9824-8a226ee37246)

![pipeline-1](https://github.com/MikhailRyzhkin/CI-CD/assets/69116076/6d17715c-d79f-4179-bdfd-66bdfdc1ae47)

Приложение доступно по любому из двух внешних IP (worker или master):
  ```
  http://158.160.75.53:30036/
  ```
или
  ```
  http://158.160.18.130:30036/
  ```





