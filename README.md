# Использование Cloud Object Storage для обработки изображений в Serverless окружении
## Введение
С помощью этого приложения вы сможете загрузить фото в облачное хранилище IBM Cloud Object Storage, и автоматически обработать его с помощью serverless процедуры, которая автоматически обработает и проанализирует изображение. После завершения анализа и обработки результаты сохраняются в другом хранилище облачных объектов, которое затем можно прочитать.

## Архитектура
   ![](images/architecture.png)

## Инструкции
### Что нужно для развертывания приложения?
1. Аккаунт в [Облаке IBM](https://ibm.biz/alexwebex)
1. Установленный локально интерфейс для работы из командной строки [IBM Cloud CLI](https://cloud.ibm.com/docs/cli/reference/ibmcloud?topic=cloud-cli-install-ibmcloud-cli#install_use) с плагином IBM Cloud Functions [Plugin](https://cloud.ibm.com/docs/openwhisk?topic=cloud-functions-cli_install) installed.

### Создать требуемые сервисы в IBM Cloud
Чтобы запустить это приложение, вам нужно сначала настроить IBM Object Storage и сервис распознавания IBM Visual Recognition Service в IBM Cloud.
1. Создать инстанс IBM Cloud Object Storage:
    * Откройте [Object Storage](https://cloud.ibm.com/catalog/services/cloud-object-storage) в каталоге IBM Cloud.
    * Дайте вашему сервису имя, нажмите `Create`.
    * Слева в меню-выберите `Buckets`, а затем - `Create bucket` - `Custom bucket`. Bucket ("ведро") - это большая  корзина, в которой будут храниться ваши данные.
    * Дайте вашему bucket уникальное имя.
    * Выберите `Regional` в параметрах `Resiliency`, а в Location  - `eu-de`. *Примечание: Можете использовать и другую геолокацию из списка, в нашем примере мы взяли датацентр в Германии*
    * Оставьте выбранным Tier по-умолчанию
		* Включите Activity tracker (опция Read and Write) и Cloud Monitoring, нажмите `Create Bucket`.
    * Создайте еще один bucket, с таким же именем, но добавьте суффикс `-processed`. Если сначала вы создали `my-bucket`, то новое имя будет `my-bucket-processed`.
    * Убедитесь, что вы выбрали тот же самый регион - `Regional` и `eu-de`.
    * В левом меню нажмите `Service Credentials`, а затем - `New Credential`.
    * Включите `Include HMAC Credential` (в Advanced Options). Нажмите `Add`.

1. Создайте инстанс сервиса Visual Recognition
    * Выберите в каталоге [Visual Recognition](https://cloud.ibm.com/catalog/services/visual-recognition)
		* Выберите регион `Frankfurt`
    * Придумайте имя вешему сервису, нажмите `Create`.
    * В левом меню нажмите `Service Credentials`. Если там пока нет существующих записей, нажмимте `New Credential`. Как только запись появится, запишите `apikey`.

### Настройка командного интерфейса IBM Cloud CLI
1. Войдите в интерфейс командной строки IBM Cloud:
    ```
    ibmcloud login
    ```

1. Просмотрите список доступных namespaces:
    ```
    ibmcloud fn namespace list
    ```

1. Выберите необходимый namespace:
    ```
    ibmcloud fn property set --namespace <namespace_id>
    ```

### Создание необходимой политики IAM для облачных функций для доступа к Cloud Object Storage
1. Перед созданием триггера событий, вы должны назначить роль менеджера уведомлений соответствующему namespace. Такая роль позволяет просматривать, изменять и удалять уведомления сервиса Cloud Object Storage.

    ```
    ibmcloud iam authorization-policy-create functions cloud-object-storage "Notifications Manager" --source-service-instance-name <functions_namespace_name> --target-service-instance-name <cos_service_instance_name>
    ```

### Создание необходимых переменных среды и развертывание облачных функций
Для развертывания функций, необходимых в этом приложении, мы будем использовать команду `ibm fn deploy`. Эта команда использует файл `manifest.yaml`, определяющий набор пакетов, действий, триггеров и правил для развертывания.

1. Склонируем приложение
    ```
    git clone git@github.com:agavrinibm/cos-trigger-functions.git
    ```

1. Взгляните на файл `serverless/manifest.yaml`. Вы увидите манифест-файл, описывающий различные действия, триггеры, пакеты и последовательности, которые будут созданы. Вы также увидите ряд переменных, которые вы должны установить локально перед запуском этого файла.

1. Выберите имя пакета, имя триггера и имя правила, а затем сохраните переменные среды.  *Примечание: пакет, который вы создадите, будет содержать все действия для этого приложения.*
    ```
    export PACKAGE_NAME=<your_chosen_package_name>
    export RULE_NAME=<your_chosen_rule_name>
    export TRIGGER_NAME=<your_chosen_trigger_name>
    ```

1. Вы уже выбрали имя группы ранее при создании экземпляра COS. Сохраните это имя как переменную среды BUCKET_NAME:
    ```
    export BUCKET_NAME=<your_bucket_name>
    ```

1. Вам нужно будет сохранить имя точки входа (endpoint), которая является точкой COS для ваших сегментов. Поскольку при выборе сегментов вы выбрали eu-de, конечной точкой должна быть `s3.eu-de.cloud-object-storage.appdomain.cloud`.

    ```
    export ENDPOINT=s3.eu-de.cloud-object-storage.appdomain.cloud
    ```

*Примечание. Если вы выбрали другой регион, вы можете найти свой endpoint, выбрав службу облачных хранилищ объектов в [списке ресурсов] (https://cloud.ibm.com/resources?groups=storage), чтобы найти ваш сегмент в список, а затем искать в разделе «Endpoint» для этого сегмента. Используйте общедоступный (public) endpoint.*

1. Наконец, вам понадобится некоторая информация из сервиса распознавания изображений Watson. Вы сохранили свой apikey ранее, так что используйте его. Это приложение сделано для версии API, датированной `2018-03-19`, поэтому мы будем использовать это значение для VERSION.

    ```
    export API_KEY=<ваш apikey для IBM Watson Visual Recognition>
    export VERSION=2018-03-19
    ```

1. Вы настроили некоторые необходимые учетные данные и различные параметры для своих функций IBM Cloud. Давайте развернем наше serverless-приложение. Перейдите в папку `serverless` и разверните функцию.
    ```
    cd serverless
    ibmcloud fn deploy
    ```

### Привязка учетные данные службы к пакету хранения созданных облачных объектов
1. Команда deploy создала для вас пакет под названием  `cloud-object-storage`. Этот пакет содержит несколько полезных облачных функций для взаимодействия с хранилищем облачных объектов. Давайте обновим пакет информацией о вашем endpoint.

    ```
    ibmcloud fn package update cloud-object-storage --param endpoint $ENDPOINT
    ```

1. Свяжем учетные данные службы с этим пакетом.
    ```
    ibmcloud fn service bind cloud-object-storage cloud-object-storage --instance YOUR_COS_INSTANCE_NAME
    ```

1. Поздравляем! Если вы обратились непосредственно к своему хранилищу облачных объектов и добавили файл, вы должны увидеть срабатывание триггера и некоторые действия, отображаемые в вашем хранилище `mybucket-processed`. Давайте развернем простое приложение для загрузки изображений и отображения результатов.

### Развертывание web-приложения
Наконец, давайте развернем веб-приложение, которое позволяет нашим пользователям загружать изображения и просматривать полученные изображения. Приложение представляет собой node.js приложение с фреймворком express, и мы можем развернуть его в IBM Cloud с помощью файла manifest.yaml, который находится в папке `/app`.
1. Переходим в папку `app` folder:
    ```
    cd ../app
    ```

1. Обновим файл `config.js`, указав необходимые значения конфигурации: имя bucket, имя bucket с обработанными данными и URL-адрес endpoint. Все эти значения вы уже прописали ранее.

1. Создайте файл с именем `credentials.json` на основе файла `credentials_template.json`. Вы можете легко получить учетные данные, перейдя на страницу сервиса COS и нажав `Service Credentials`. Вы можете скопировать весь этот блок и вставить его как дочерний в `"OBJECTSTORAGE_CREDENTIALS":`.


1. Откройте файл manifest.yaml в `app/manifest.yaml`. Вы должны увидеть что-то вроде этого:
    ```
    ---
    applications:
    - name: <YOUR_APPLICATION_NAME>
      memory: 256M
      instances: 1
    ```

    Этот файл манифеста описывает приложение для облачного развертывания с использованием cloud foundry, которое мы собираемся развернуть. Пожалуйста, переименуйте приложение, дав ему уникальное название.

1. Разверните приложение:
    ```
    ibmcloud cf push
    ```

1. Наконец, нам нужно обновить настройки CORS для нашего bucket. Пакет `cloud-object-storage`, который мы установили ранее, поставляется с методом, который позволяет это сделать. Запустите следующую команду, но сначала отредактируйте ее, указав свое собственное имя bucket `-processed`.

    ```
    ibmcloud fn action invoke cloud-object-storage/bucket-cors-put -b -p bucket myBucket-processed -p corsConfig "{\"CORSRules\":[{\"AllowedHeaders\":[\"*\"], \"AllowedMethods\":[\"POST\",\"GET\",\"DELETE\"], \"AllowedOrigins\":[\"*\"]}]}"
    ```

1. Теперь вы сможете использовать ваше приложение! После развертывания система выведет ссылку URL-адрес, по которой развернуто ваше приложение, наподобие `my-app-name.mybluemix.net`. Зайдите по этой ссылке и посмотрите как оно работает!
