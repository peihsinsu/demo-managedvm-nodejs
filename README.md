# Google AppEngine - Managed VM

Managed VM是Google App Engine所推出的PaaS - AppEngine服務的延伸，透過Linux Container的技術與AppEngine frontend的搭配，讓整個AppEngine的功能無限延伸。下面介紹節錄自[Node.js in Example](https://www.gitbook.com/book/peihsinsu/node-js-500-samples/details)。

## 直接使用這個repository

假設gcloud與docker相關套件都已安裝完成，那麼您可以clone之後直接執行

```
$ git clone git@github.com:peihsinsu/demo-managedvm-nodejs.git
$ gcloud compute preview app run .
```

## 從無到有建置Node.js Managed VM環境

### 安裝boot2docker

您可以到boot2docker的官方下載點，下載boot2docker以供安裝。安裝後，即可執行boot2docker的指令，並解僵boot2docker環境啟動...

```
boot2docker init
$ boot2docker up
$ $(boot2docker shellinit)
```

### 安裝google cloud sdk

Mac環境下，透過curl即可下載安裝google cloud sdk...，其他環境可以參考Google官方說明

```
curl https://sdk.cloud.google.com | bash
```

安裝完cloud sdk之後，需要下載開發人員預覽版本的指令庫...

```
gcloud components update app
```

### 認證與設定專案

替您的google cloud sdk認證

```
gcloud auth login
```

接著會出現或是開啟認證用的url，完成認證流程即可。接下來則是將Google專案資訊設定至SDK環境...

```
gcloud config set project [project-id]
```


## 準備開發 - 以express為範例

### 安裝相關套件

```
$ sudo npm install express-generator -g
```

### 建置express專案
```
$ express -e myapp
$ cd myapp
```

## 增加app.yaml設定檔

app.yaml是AppEngine環境的設定檔，目的是提供frontend instance (appengine)與backend instance (managed vm)之間的呼叫介面，以下面設定為例，即是讓所有的路由均由app.js提供服務...

```
version: 2
runtime: custom
vm: true
api_version: 1

manual_scaling:
  instances: 1

handlers:
- url: /.*
  script: app.js
```


## 建立"Dockerfile"檔案

Dockerfile是作為未來部署Managed VM環境的依據，您可以依照希望執行的Backend狀態來庭整這個設定檔。在此，我們只有單純把docker賦予node.js的環境，因此僅僅使用Google所提供的base image即可...

```
FROM google/nodejs-runtime
```

如果對Docker有興趣，您可以到docker hub檢視Google node.js runtime的docker build file ([參考](https://registry.hub.docker.com/u/google/nodejs-runtime/dockerfile/))，可以了解實際Managed VM執行的環境與狀態...

## 編輯package.json，引入appengine模組

在package.json中加入appengine的dependency，該套件提供了與AppEngine整合的一些操作方法，讓應用程式得以與Google其他服務互動。

```
{
  "name": "test",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "start": "node ./bin/www"
  },
  "dependencies": {
    "body-parser": "~1.12.0",
    "cookie-parser": "~1.3.4",
    "debug": "~2.1.1",
    "ejs": "~2.3.1",
    "express": "~4.12.0",
    "morgan": "~1.5.1",
    "serve-favicon": "~2.2.0",
    "appengine" : "git://github.com/GoogleCloudPlatform/appengine-nodejs.git"
  }
}
```

## 修改express設定檔

加入appengine的import

```
var appengine = require('appengine');
```

並且於app.use部分加入appengine middleware模組

```
app.use(appengine.middleware.base);
```

## 建立系統預設路由 - /_ah/*

加入Managed VM的health check routes

```
app.get('/_ah/health', function(req, res) {
  res.set('Content-Type', 'text/plain');
  res.send(200, 'ok');
});

app.get('/_ah/start', function(req, res) {
  res.set('Content-Type', 'text/plain');
  res.send(200, 'ok');
});

app.get('/_ah/stop', function(req, res) {
  res.set('Content-Type', 'text/plain');
  res.send(200, 'ok');
  process.exit();
});
```


## 修改啟動port至8080

開啟bin/www檔案，將預設port修改為8080

```
var port = normalizePort(process.env.PORT || '8080');
```

## 預先測試

在未佈署到docker環境錢，該專案可以在npm install安裝完相依套件後，使用npm start測試是否可以正常執行。

```
npm install && npm start
```

可以如此測試是因為Docker build file中有指定docker instance開立起來後，是以npm start來喚起所要執行的專案... 因此我們可以使用這個方式來測試該專案到container中是否可以正常執行...


## 本地端執行Managed VM

本地端執行Managed VM時，需要搭配boot2docker已經開啟的狀態下，切換到專案目錄中，執行下面指令：

```
$ gcloud preview app run .
Module [default] found in file [/private/tmp/test/app.yaml]
INFO: Looking for the Dockerfile in /private/tmp/test
INFO: Using Dockerfile found in /private/tmp/test
INFO     2015-03-04 15:53:04,198 devappserver2.py:726] Skipping SDK update check.
INFO     2015-03-04 15:53:04,293 api_server.py:172] Starting API server at: http://localhost:62335
INFO     2015-03-04 15:53:04,365 containers.py:259] Building docker image managedvm-project.default.2 from /private/tmp/test/Dockerfile:
INFO     2015-03-04 15:53:04,365 containers.py:261] --------------------  DOCKER BUILD  --------------------
INFO     2015-03-04 15:53:04,366 dispatcher.py:186] Starting module "default" running at: http://localhost:8080
INFO     2015-03-04 15:53:04,370 admin_server.py:118] Starting admin server at: http://localhost:8000
INFO     2015-03-04 15:53:04,873 containers.py:280] # Executing 3 build triggers

INFO     2015-03-04 15:53:04,874 containers.py:280] Trigger 0, ADD package.json /app/
INFO     2015-03-04 15:53:04,874 containers.py:280] Step 0 : ADD package.json /app/
INFO     2015-03-04 15:53:04,879 containers.py:280] ---> Using cache
INFO     2015-03-04 15:53:04,879 containers.py:280] Trigger 1, RUN npm install
INFO     2015-03-04 15:53:04,879 containers.py:280] Step 0 : RUN npm install
INFO     2015-03-04 15:53:04,917 containers.py:280] ---> Using cache
INFO     2015-03-04 15:53:04,918 containers.py:280] Trigger 2, ADD . /app
INFO     2015-03-04 15:53:04,918 containers.py:280] Step 0 : ADD . /app
INFO     2015-03-04 15:53:04,937 containers.py:280] ---> Using cache
INFO     2015-03-04 15:53:04,937 containers.py:280] ---> cd272a90bc21
INFO     2015-03-04 15:53:04,938 containers.py:280] Successfully built cd272a90bc21
INFO     2015-03-04 15:53:04,948 containers.py:292] --------------------------------------------------------
INFO     2015-03-04 15:53:04,948 containers.py:304] Image managedvm-project.default.2 built, id = cd272a90bc21
INFO     2015-03-04 15:53:04,948 containers.py:534] Creating container...
INFO     2015-03-04 15:53:05,104 containers.py:560] Container ef5775aa5d26ade2d524dd44d0cf6a18fa6c5c7aa100ccb3f925e3af30e5b72a created.
INFO     2015-03-04 15:53:06,407 module.py:1692] New instance for module "default" serving on:
http://localhost:8080

INFO     2015-03-04 15:53:06,435 module.py:737] default: "GET /_ah/start HTTP/1.1" 200 2
INFO     2015-03-04 15:53:06,435 health_check_service.py:101] Health checks starting for instance 0.

```


## 佈署Managed VM

如開發上一切都順利，接下來可以執行部屬的動作，透過docker preview app deploy來作佈署...

```
$ gcloud preview app deploy .
11:53 PM Host: appengine.google.com
{bucket: vm-containers.managedvm-project.appspot.com, path: /containers}

Updating module [default] from file [/private/tmp/test/app.yaml]
Pushing image to Google Cloud Storage...managedvm-project
Sending image list
Pushing repository gcr.io/_m_managedvm-project/managedvm-project.default.2 (1 tags)
Image 541923dd11eb already pushed, skipping
Image 11971b6377ef already pushed, skipping
...(skip)
Image b62c4c483651 already pushed, skipping
Pushing
Buffering to disk:  2.56 kB
Image successfully pushed===================================>]  2.56 kB/2.56 kB 0s
Pushing
Buffering to disk: 9.834 MB
Image successfully pushed===================================>] 9.834 MB/9.834 MB 0ss
Pushing
Buffering to disk: 2.629 MB
Image successfully pushed===================================>] 2.629 MB/2.629 MB 0ss1s
Pushing tag for rev [c7685fe2aad2] on {https://gcr.io/v1/repositories/_m_managedvm-project/managedvm-project.default.2/tags/latest}
11:54 PM Host: appengine.google.com
11:54 PM Application: managedvm-project (was: None); version: 2
11:54 PM
Starting update of app: managedvm-project, version: 2
11:54 PM Getting current resource limits.managedvm-project
11:54 PM Scanning files on local disk.
^@11:54 PM WARNING: Performance settings included in this update are being ignored because your application is not using the Modules feature. See the Modules documentation for more information. Python: https://developers.google.com/appengine/docs/python/modules/ Java: https://developers.google.com/appengine/docs/java/modules/ Go: https://developers.google.com/appengine/docs/go/modules/ PHP: https://developers.google.com/appengine/docs/php/modules/.
11:54 PM Cloning 291 application files.
11:54 PM Uploading 51 files and blobs.
11:55 PM Uploaded 51 files and blobs.
11:55 PM Starting deployment.
11:55 PM Checking if deployment succeeded.
11:55 PM Deployment successful.
11:55 PM Checking if updated app version is serving.
11:55 PM Will check again in 5 seconds.
...(skip)
11:57 PM Checking if updated app version is serving.
11:57 PM Enough VMs ready (1/1 ready).
11:57 PM Completed update of app: managedvm-project, version: 2
```

## 測試

當deploy完成後，則可以透過appengine的url規則(如下)來存取您的服務:

```
https://[your-project-id].appspot.com
```

## 進階

Managed VM提供scale的方法，設定上很簡單，只要在app.yaml中指定所要擴充的最高數量在"manual_scaling"中即可：

```
version: 2
runtime: custom
vm: true
api_version: 1

manual_scaling:
  instances: 5

handlers:
- url: /.*
  script: app.js
```

在使用manual_scaling時候，主機的調降需要由user自己動作，而如果在系統的設計上已經有考慮到交易的安全性，那使用全自動的scale方式也相當方便：

```
version: 2
runtime: custom
vm: true
api_version: 1

automatic_scaling:
  min_num_instances: 2
  max_num_instances: 20
  cool_down_period_sec: 60
  cpu_utilization:
    target_utilization: 0.5

handlers:
- url: /.*
  script: app.js
```

## 附註一

由於Google Managed VM改版，設定檔部分需要稍作調整，如果您看到下面這樣的訊息...

```
$ gcloud preview app deploy app.yaml
WARNING: The [version] field is specified in file [/path/to/your-project/app.yaml].  This field is not used by gcloud and should be removed.
ERROR: The version [test] declared in [/path/to/your-project/app.yaml] does not match the current gcloud version [20150428t143637].
ERROR: (gcloud.preview.app.deploy) Errors occurred while parsing the App Engine app configuration.
```

此時，您需要修改app.yaml，將"version: your-version"的部分移掉，重新執行deploy應該就可以正常deploy了！

## 附註二

筆者曾遇到Managed VM在Local執行時候連線到docker時候無法連線，錯誤訊息是因為SSL認證問題...，這時可以檢查您的Python是否為3.x或是2.7.9以上版本，如果是的話，可以試著透過brew將python移除，讓python版本回到2.7.6，或是透過您的方法將python版本降版到2.7.8以下，應該就可以正常執行部屬。

