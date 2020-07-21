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
To deploy the functions required in this application, we'll use the `ibm fn deploy` command. This command will look for a `manifest.yaml` file defining a collection of packages, actions, triggers, and rules to be deployed.
1. Let's clone the application.
    ```
    git clone git@github.com:IBM/cos-trigger-functions.git
    ```

1. Take a look at the `serverless/manifest.yaml file`. You should see manifest describing the various actions, triggers, packages, and sequences to be created. You will also notice that there are a number of environment variables you should set locally before running this manifest file.

1. Choose a package name, trigger name, and rule name and then save the environment variables.  *Note: The package you will create will hold all of the actions for this application.*
    ```
    export PACKAGE_NAME=<your_chosen_package_name>
    export RULE_NAME=<your_chosen_rule_name>
    export TRIGGER_NAME=<your_chosen_trigger_name>
    ```

1. You already chose a bucket name earlier when creating your COS Instance. Save that name as your BUCKET_NAME environment variable:
    ```
    export BUCKET_NAME=<your_bucket_name>
    ```

1. You will need to save the endpoint name, which is the COS Endpoint for your buckets. Since you selected us-south when selecting your buckets, the endpoint should be `s3.us-south.cloud-object-storage.appdomain.cloud`.

    ```
    export ENDPOINT=s3.us-south.cloud-object-storage.appdomain.cloud
    ```

*Note: If you selected a different region, you can find your endpoint by clicking your Cloud Object Storage service in the [Resource list](https://cloud.ibm.com/resources?groups=storage), finding your bucket in the list, and then looking under Configuration for that bucket. Use the public endpoint.*

1. Finally, you will need some information from the Visual Recognition service.  You saved your apikey earlier, so use that. This application is built against the version released on `2018-03-19`, so we'll use that value for VERSION.
    ```
    export API_KEY=<your_visual_recognition apikey>
    export VERSION=2018-03-19
    ```

1. You've set up some required credentials and various parameters for your IBM Cloud Functions. Let's deploy the functions now! Change directories to the `serverless` folder, and then deploy the functions.
    ```
    cd serverless
    ibmcloud fn deploy
    ```

### Bind Service Credentials to the Created Cloud Object Storage Package
1. The deploy command created a package for you called `cloud-object-storage`. This package contains some useful cloud functions for interacting with cloud object storage. Let's update the package with your endpoint information.
    ```
    ibmcloud fn package update cloud-object-storage --param endpoint $ENDPOINT
    ```

1. Let's bind the service credentials to this package.
    ```
    ibmcloud fn service bind cloud-object-storage cloud-object-storage --instance YOUR_COS_INSTANCE_NAME
    ```

1. Congratulations! If you went directly to your cloud object storage bucket and added a file, you should see your trigger fire and some processed actions showing up in your `mybucket-processed` bucket. Let's deploy a simple application for uploading the images and showing these results.

### Deploy the Web Application
Finally, let's deploy the web application that enables our users to upload images and see the resulting images. The application is a node.js with express app, and we can deploy it to IBM Cloud using the manifest.yaml file provided in the `/app` folder.
1. Change directories to the `app` folder:
    ```
    cd ../app
    ```

1. Update the `config.js` file with the required configuration values: your bucket name, your processed bucket name, and your endpoint url. You should've already found these values earlier.

1. Create a file named `credentials.json` based on the `credentials_template.json` file. You can easily get the credentials by going to the cloud object storage service page, and clicking `Service Credentials`. You can copy this entire block and paste it as a child to `"OBJECTSTORAGE_CREDENTIALS":`.

1. Open up the manifest.yaml file at `app/manifest.yaml`. You should see something like this:
    ```
    ---
    applications:
    - name: <YOUR_APPLICATION_NAME>
      memory: 256M
      instances: 1
    ```

    This manifest file describes the cloud foundry application we're about to deploy. Please rename the application to anything you would like.

1. Deploy the application:
    ```
    ibmcloud cf push
    ```

1. Finally, we'll need to update the CORS settings on our processed images bucket.  The `cloud-object-storage` package we installed earlier comes with a method to let us do that. Run the following command, but be sure to edit it first with your own `-processed` bucket name.
    ```
    ibmcloud fn action invoke cloud-object-storage/bucket-cors-put -b -p bucket myBucket-processed -p corsConfig "{\"CORSRules\":[{\"AllowedHeaders\":[\"*\"], \"AllowedMethods\":[\"POST\",\"GET\",\"DELETE\"], \"AllowedOrigins\":[\"*\"]}]}"
    ```

1. You should now be able to use the application you deployed! When the application was deployed, it should've output a URL where your application lives, something like `my-app-name.mybluemix.net`. You can now go there and start uploading images and seeing your results!
