
# Введение

В инструкции собран материал с нескольких источников с небольшими изменениями.
Инструкция преведена для настройки одиночного хоста. В качестве хоста используется вирутальная машина, и порты, который будут видны снаружи будут отличаться от стандартных портов, которые использует ELK. В данной статье будут самые азы - как сделать быструю установку и базовую настройку, чтобы можно было собирать логи в Elasticsearch и смотреть их в Kibana.  Инструкция по установке будет полностью основана на [официальной документации](https://www.elastic.co/guide/index.html). Если вы хорошо читаете английские тексты, то можете использовать ее. В этой статье в одном месте  собран необходимый минимум для запуска Elasticsearch, Logstash, Kibana.

## Что такое ELK 

Расскажу своими словами о том, что мы будем настраивать.Когда становится много логов, хранить их в текстовых файлах неудобно. Надо как-то обрабатывать и выводить в удобном виде для просмотра. Именно эти проблемы и решает представленный стек программ:

* Elasticsearch используется для хранения, анализа, поиска по логам.
* Kibana представляет удобную и красивую web панель для работы с логами.
* Logstash сервис для сбора логов и отправки их в Elasticsearch. В самой простой конфигурации можно обойтись без него и отправлять логи напрямую в еластик. Но с logstash это делать удобнее.
* Beats - агенты для отправки логов в Logstash или Elasticsearch. 


## Добавим репозиторий elasticsearch

Все три компененты ELK  доступны из единого репозитория:

``` bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

sudo apt-get install apt-transport-https

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```


## Установка Elasticsearch

Текущая развернутая версия Elasticsearch 8.13.2. Важно: установка через apt не работала без VPN на этапе написания инструкции. Используйте VPN или воспользуйтесь скачанным пакетом для Ubuntu (инструкции будут ниже).

``` bash
sudo apt-get update && sudo apt-get install elasticsearch=8.13.2
```

Можно воспользоваться скачанным пакетом и установить его вручную. 

``` bash
wget "https://cloud.iszf.irk.ru/index.php/s/AyjAk8tIB2P24H8/download?path=%2F&files=elasticsearch-8.13.2-amd64.deb" -O elasticsearch-8.13.2-amd64.deb

wget "https://cloud.iszf.irk.ru/index.php/s/AyjAk8tIB2P24H8/download?path=%2F&files=elasticsearch-8.13.2-amd64.deb.sha512" -O elasticsearch-8.13.2-amd64.deb.sha512

shasum -a 512 -c elasticsearch-8.13.2-amd64.deb.sha512

sudo dpkg -i elasticsearch-8.13.2-amd64.deb
```

После установки мы получим вывод с полезной информацией которую будем использовать дальше для настройки:

``` bash

--------------------------- Security autoconfiguration information ------------------------------

Authentication and authorization are enabled.
TLS for the transport and HTTP layers is enabled and configured.

The generated password for the elastic built-in superuser is : 2I+VcTrdsllHVoI-rb-R

If this node should join an existing cluster, you can reconfigure this with
'/usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token <token-here>'
after creating an enrollment token on your existing cluster.

You can complete the following actions at any time:

Reset the password of the elastic built-in superuser with 
'/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic'.

Generate an enrollment token for Kibana instances with 
 '/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana'.

Generate an enrollment token for Elasticsearch nodes with 
'/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node'.

-------------------------------------------------------------------------------------------------

### NOT starting on installation, please execute the following statements to configure elasticsearch 
service to start automatically using systemd
 sudo systemctl daemon-reload
 sudo systemctl enable elasticsearch.service
### You can start elasticsearch service by executing
 sudo systemctl start elasticsearch.service


``` 

Нас будут интересовать следующие строки c паролкем пользователя, под которым мы будем входить в систему *elastic* и команда для сброса этого пароля:

``` bash
The generated password for the elastic built-in superuser is : 2I+VcTrdsllHVoI-rb-R

Reset the password of the elastic built-in superuser with 
'/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic'.
``` 

Запустим elasticsearch:

``` bash
sudo systemctl start elasticsearch.service
```

И проверим что он работает выполнив:

``` bash
curl -k --user elastic:'2I+VcTrdsllHVoI-rb-R' https://127.0.0.1:9200
```

и получим:


``` bash
{
  "name" : "elktest",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "uyYQPsPYSxGg767GI6Vx9Q",
  "version" : {
    "number" : "8.13.2",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "16cc90cd2d08a3147ce02b07e50894bc060a4cbf",
    "build_date" : "2024-04-05T14:45:26.420424304Z",
    "build_snapshot" : false,
    "lucene_version" : "9.10.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}

```


## Настройка Elastic

Настройки храняться в файле `/etc/elasticsearch/elasticsearch.yml`. Ниже приведены настройки, которые используются у нас на сервере:

``` bash

cat /etc/elasticsearch/elasticsearch.yml | grep "^[^#;]"

path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 127.0.0.1
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
cluster.initial_master_nodes: ["elktest"]
http.host: 127.0.0.1

```


Видно Elasticsearch слушает localhost (127.0.0.1). Нам это и нужно, так как данные в него будет передавать logstash, который будет установлен локально. Если предполагается установка на разных машинах используйте 0.0.0.0 вместо 127.0.0.1 и добавьте в конфигурацию @ discovery.seed_hosts: ["127.0.0.1", "[::1]"]@. Обращаю отдельное внимание на параметр для директории с данными (path.data). Чаще всего они будут занимать значительное место, иначе зачем нам Elasticsearch. Подумайте заранее, где вы будете хранить логи. После изменения настроек, надо перезапустить службу:

``` bash
sudo systemctl restart elasticsearch.service
```

Порт по умолчанию 9200. Проверим что Elastic слушает его:

``` bash
sudo netstat -tulnp | grep 9200
  tcp6       0      0 127.0.0.1:9200          :::*                    LISTEN      62578/java
```


## Установка Kibana

_Обязательно выполните пункт где мы установили репозиторий elasticsearch._ Попробуйте установить через apt-get

``` bash
sudo apt-get update && sudo apt-get install kibana=8.13.2
```

Если нет доступа используйте VPN или установку из пакета:

``` bash
wget "https://cloud.iszf.irk.ru/index.php/s/AyjAk8tIB2P24H8/download?path=%2F&files=kibana-8.13.2-amd64.deb" -O kibana-8.13.2-amd64.deb

wget "https://cloud.iszf.irk.ru/index.php/s/AyjAk8tIB2P24H8/download?path=%2F&files=kibana-8.13.2-amd64.deb.sha512" -O kibana-8.13.2-amd64.deb.sha512

shasum -a 512 -c kibana-8.13.2-amd64.deb.sha512
# should see:
# kibana-8.13.2-amd64.deb: OK
sudo dpkg -i kibana-8.13.2-amd64.deb
```

Перезагрузим службу:

``` bash
sudo systemctl daemon-reload
sudo systemctl enable kibana.service
sudo systemctl start kibana.service
```

Проверим что Kibana работает:

``` bash
sudo systemctl status kibana.service


● kibana.service - Kibana
     Loaded: loaded (/lib/systemd/system/kibana.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2024-04-19 08:36:07 UTC; 17s ago
       Docs: https://www.elastic.co
   Main PID: 63472 (node)
      Tasks: 11 (limit: 9274)
     Memory: 305.5M
        CPU: 18.831s
     CGroup: /system.slice/kibana.service
             └─63472 /usr/share/kibana/bin/../node/bin/node /usr/share/kibana/bin/../src/cli/dist

```

И слушает порты. По умолчанию, Kibana слушает порт 5601. Только не спешите его проверять после запуска. Кибана стартует долго. Подождите примерно минуту и проверяйте.

``` bash
sudo netstat -tulnp | grep 5601
  tcp        0      0 127.0.0.1:5601          0.0.0.0:*               LISTEN      63472/node   
```

### Настройка Kibana

Файл с настройками Кибана располагается по пути - /etc/kibana/kibana.yml. На начальном этапе нам нужно корректно настроить подключение к серверу elasticsearch. По умолчанию Kibana слушает только localhost и не позволяет подключаться удаленно. Это нормальная ситуация, если у вас будет на этом же сервере установлен nginx в качестве reverse proxy, который будет принимать подключения и проксировать их в Кибана. Так и нужно делать в production, когда системой будут пользоваться разные люди из разных мест. С помощью nginx можно будет разграничивать доступ, использовать сертификат, настраивать нормальное доменное имя и т.д. Если же у вас это тестовая установка, то можно обойтись без nginx. Для этого надо разрешить Кибана слушать внешний интерфейс и принимать подключения. Измените параметр server.host, указав ip адрес сервера или указав 0.0.0.0, чтобы Kibana слушала все интерфейсы. Список настроек будет в конце этой секции. 

Сбрасываем проль:

``` bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system
This tool will reset the password of the [kibana_system] user to an autogenerated value.
The password will be printed in the console.
Please confirm that you would like to continue [y/N]y


Password for the [kibana_system] user successfully reset.
New value: iv7bMIk0i21A2cC8zOd_
```

И прописываем его в конфигурационном файле:

``` bash
echo elasticsearch.username: \"kibana_system\" | sudo tee -a  /etc/kibana/kibana.yml\
echo elasticsearch.password: \"iv7bMIk0i21A2cC8zOd_\" | sudo tee -a  /etc/kibana/kibana.yml
```


Теперь нам нужно указать сертификат от центра сертификации, который использовал elasticsearch для генерации tls сертификатов. Напомню, что обращения он принимает только по HTTPS. Так как мы не устанавливали подтверждённый сертификат, будем использовать самоподписанный, который был сгенерирован автоматически во время установки elasticsearch. Все созданные сертификаты находятся в директории /etc/elasticsearch/certs. Скопируем их в /etc/kibana и настроим доступ со стороны веб панели:

``` bash
sudo cp -R /etc/elasticsearch/certs /etc/kibana
sudo chown -R root:kibana /etc/kibana/certs
```

Добавим этот CA в конфигурацию Кибаны:

``` bash
echo elasticsearch.ssl.certificateAuthorities: [ \"/etc/kibana/certs/http_ca.crt\" ] | sudo tee -a  /etc/kibana/kibana.yml
```


И последний момент - указываем подключение к elasticsearch по HTTPS:

``` bash
echo elasticsearch.hosts: [\"https://localhost:9200\"] | sudo tee -a  /etc/kibana/kibana.yml
```

Теперь Kibana можно перезапустить:

``` bash
systemctl restart kibana.service
```

Мы используем следующие настройки:

``` bash
sudo cat /etc/kibana/kibana.yml | grep "^[^#;]"

server.host: "0.0.0.0"
logging:
  appenders:
    file:
      type: file
      fileName: /var/log/kibana/kibana.log
      layout:
        type: json
  root:
    appenders:
      - default
      - file
pid.file: /run/kibana/kibana.pid
elasticsearch.username: "kibana_system"
elasticsearch.password: "iv7bMIk0i21A2cC8zOd_"
elasticsearch.ssl.certificateAuthorities: [ "/etc/kibana/certs/http_ca.crt" ]
elasticsearch.hosts: ["https://localhost:9200"]

```

Теперь мы можем подключиться к Elasticsearch используя учетную запись *elastic*, пароль для которой сгенерирован во время установки elasticsearch (см. пункт *Установка Elasticsearch*).

![elastic_greet](/graphics/elastic_greet.png)

После входа в учетную запись мы увидим старотовую страницу:

![elastic_start_page](/graphics/elastic_start_page.png)

При первом запуске Kibana предлагает настроить источники для сбора логов. Это можно сделать, нажав на Add integrations. К сбору данных мы перейдем чуть позже.

## Установка Logstash

_Обязательно выполните пункт, где мы установили репозиторий elasticsearch._ Попробуйте установить через apt-get

``` bash
sudo apt-get update && sudo apt-get install logstash=8.13.2
```

Если нет доступа используйте VPN или установку из пакета:

``` bash
wget "https://cloud.iszf.irk.ru/index.php/s/AyjAk8tIB2P24H8/download?path=%2F&files=logstash-8.13.2-amd64.deb" -O logstash-8.13.2-amd64.deb

wget "https://cloud.iszf.irk.ru/index.php/s/AyjAk8tIB2P24H8/download?path=%2F&files=logstash-8.13.2-amd64.deb.sha512" -O logstash-8.13.2-amd64.deb.sha512

shasum -a 512 -c logstash-8.13.2-amd64.deb.sha512
# should see:
# logstash-8.13.2-amd64.deb: OK

sudo dpkg -i logstash-8.13.2-amd64.deb
```

Добавим logstash в автозагрузку:

``` bash
sudo systemctl enable logstash.service
# should see
# Created symlink /etc/systemd/system/multi-user.target.wants/logstash.service → /lib/systemd/system/logstash.service.
```

Запуск с помощью @sudo systemctl start kibana.service@ сделаем чуть позже. Основной конфиг logstash лежит по адресу /etc/logstash/logstash.yml. Я его не трогаю, а все настройки буду по смыслу разделять по разным конфигурационным файлам в директории `/etc/logstash/conf.d`. Создаем первый конфиг input.conf, который будет описывать приём информации с beats агентов. Тут все просто. Указываю, что принимаем информацию на 5044 порт и этого достаточно:

``` bash
sudo echo "input {
  beats {
    port => 5044
  }
  tcp {
    port => 5000
  }

}" | sudo tee -a /etc/logstash/conf.d/input.conf 

```

Если потребуется выпустить elk наружу, то можно настроить ssl сертификаты [для соединения beats и logstash](https://www.elastic.co/guide/en/beats/filebeat/current/configuring-ssl-logstash.html) это можно проделать с помощью центра [lets crypt](https://letsencrypt.org) для которого есть удобный клиент [certbot](https://certbot.eff.org/instructions?ws=other&os=ubuntufocal). На данном этапе это непотребуется

Мы определили конфигурацию для beats в файле input.conf, так как они являются источниками данных для logstash. Теперь нам нужно определить конфигурацию для elasticserach, так как он является потребителем данных logstash то мы это делаем в файле output.conf. 

``` bash
sudo echo "output {
        elasticsearch {
            hosts    => \"https://localhost:9200\"
            index    => \"logs-%{+YYYY.MM}\"
	    user => \"elastic\"
	    password => \"2I+VcTrdsllHVoI-rb-R\"
	    cacert => \"/etc/logstash/certs/http_ca.crt\"
        }
}" | sudo tee -a /etc/logstash/conf.d/output.conf 

```

Настройки читаются следующим образом:

Передавать все данные в elasticsearch под указанным индексом с маской в виде даты. Разбивка индексов по дням и по типам данных удобна с точки зрения управления данными. Потом легко будет выполнять очистку данных по этим индексам. Для аутентификации я указал учётку elastic, но вы можете сделать и отдельную через Kibana (раздел Stack Management -> Users). При этом можно создавать под каждый индекс свою учётную запись с разрешением доступа только к этому индексу.

*Важно: после настройки убедитесь что index указанный в конфигурационном файле так же настроен в log indices на сайте:*

![elastic_index_setting](/graphics/elastic_index_setting.png)

Как и в случае с Kibana, для Logstash нужно указать CA, иначе он будет ругаться на недоверенный сертификат elasticsearch и откажется к нему подключаться. Копируем сертификаты и назначаем права:

``` bash
sudo cp -R /etc/elasticsearch/certs /etc/logstash
sudo chown -R root:logstash /etc/logstash/certs
```

И запускаем службу:

``` bash
sudo systemctl start logstash.service
```

h2. Протестируем отправку логов по TCP

Для начала установим зависимости:

``` bash
pip install python-logstash-async
pip install fastapi
pip install uvicorn
```

Подготовим небольшое приложение с помощью FastAPI. Сохраните код ниже в файл с названием @logging_app.py@

``` python
from fastapi import FastAPI, Request
import logging
from logstash_async.handler import AsynchronousLogstashHandler

app = FastAPI()
ELASTIC_HOST = "0.0.0.0"
ELASTIC_TCP_PORT = 5000 # must be the same as in /etc/logstash/conf.d/input.conf

logger = logging.getLogger(f"app_logger")
logger.setLevel(logging.INFO)

formatter = logging.Formatter("%(asctime)s - %(name)s - %(levelname)s - %(message)s")

file_handler = logging.FileHandler('/tmp/app.log')
logstash_handler = AsynchronousLogstashHandler(ELASTIC_HOST, ELASTIC_TCP_PORT, database_path=None)

file_handler.setFormatter(formatter)
logstash_handler.setFormatter(formatter)

logger.addHandler(file_handler)
logger.addHandler(logstash_handler)

@app.get("/greet")
async def greet(name: str, request: Request):
    client_ip = request.client.host
    logger.info(f"Greet {name} for {client_ip}")
    return {"greeting": f"Hello {name}"}
```


### Запустим приложение:

![fastapi_startup](/graphics/fastapi_startup.png)

И перейдем в браузер (см. URL на скриншоте) и несколько раз выполним запрос.

![swagger_request](/graphics/swagger_request.png)

В терминале где запущено приложени появится информация о запросах:

![fastapi_requests](/graphics/fastapi_requests.png)

И проверим что мы видим эти же сообщения в elastic:

![elastic_logs_list](/graphics/elastic_logs_list.png)


## Отправка логов через filebeat

Процесс отправки хорошо описан [здесь](https://www.elastic.co/guide/en/cloud/current/ec-getting-started-search-use-cases-python-logs.html).

Загрузить пакеты можно череp VPN или так:

``` bash
wget "https://cloud.iszf.irk.ru/index.php/s/AyjAk8tIB2P24H8/download?path=%2F&files=filebeat-8.13.2-amd64.deb" -O filebeat-8.13.2-amd64.deb

wget "https://cloud.iszf.irk.ru/index.php/s/AyjAk8tIB2P24H8/download?path=%2F&files=filebeat-8.13.2-amd64.deb.sha512" -O filebeat-8.13.2-amd64.deb.sha512

shasum -a 512 -c filebeat-8.13.2-amd64.deb.sha512
# should see:
# filebeat-8.13.2-amd64.deb: OK

sudo dpkg -i filebeat-8.13.2-amd64.deb
```

Для Windows используйте [установщик](https://cloud.iszf.irk.ru/index.php/s/AyjAk8tIB2P24H8/download?path=%2F&files=filebeat-8.13.3-windows-x86_64.msi)

Источники:

https://serveradmin.ru/ustanovka-i-nastroyka-elasticsearch-logstash-kibana-elk-stack/
