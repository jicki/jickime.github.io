# docker 容器日志集中 ELK + filebeat




# docker 容器日志集中 ELK

## 说明
ELK 基于 ovr 网络下

![此处输入图片的描述][1]

 

## docker-compose yaml 文件


```
version: '2'
networks:
  network-test:
    external:
      name: ovr0
services:
  elasticsearch:
    image: elasticsearch
    network-test:
      external:
    hostname: elasticsearch
    container_name: elasticsearch
    restart: always
    volumes:
      - /opt/elasticsearch/data:/usr/share/elasticsearch/data

  kibana:
    image: kibana
    network-test:
      external:    
    hostname: kibana
    container_name: kibana
    restart: always
    environment:
      ELASTICSEARCH_URL: http://elasticsearch:9200/
    ports:
      - 5601:5601

  logstash:
    image: logstash
    network-test:
      external:
    hostname: logstash
    container_name: logstash
    restart: always
    volumes:
      - /opt/logstash/conf:/opt/logstash/conf
    command: logstash -f /opt/logstash/conf/

  filebeat:
    image: prima/filebeat
    network-test:
      external:
    hostname: filebeat
    container_name: filebeat
    restart: always
    volumes:
      - /opt/filebeat/conf/filebeat.yml:/filebeat.yml
      - /opt/upload:/data/logs
      - /opt/filebeat/registry:/etc/registry
```
 

## filebeat 说明

filebeat.yml 挂载为 filebeat 的配置文件

logs 为 容器挂载日志的目录

registry 读取日志的记录，防止filebeat 容器挂掉，需要重新读取所有日志

 
## logstash 说明

logstash 配置文件如下：

使用 patterns 

logstash.conf： (配置了二种输入模式， filebeats, syslog)   

```
input {
    beats {
        port => 5044
        type => beats
    }

    tcp {
        port => 5000
        type => syslog
    }

}

filter {
  if[type] == "tomcat-log" {
    multiline {
      patterns_dir => "/opt/logstash/conf/patterns"
      pattern => "(^%{TOMCAT_DATESTAMP})|(^%{CATALINA_DATESTAMP})"
      negate => true
      what => "previous"
    }

    if "ERROR" in [message] {    #如果消息里有ERROR字符则将type改为自定义的标记
        mutate { replace => { type => "tomcat_catalina_error" } }
    }

    else if "WARN" in [message] {
        mutate { replace => { type => "tomcat_catalina_warn" } }
    }

    else if "DEBUG" in [message] {
        mutate { replace => { type => "tomcat_catalina_debug" } }
    }

    else {
        mutate { replace => { type => "tomcat_catalina_info" } }
    }

    grok{
      patterns_dir => "/opt/logstash/conf/patterns"
      match => [ "message", "%{TOMCATLOG}", "message", "%{CATALINALOG}" ]
      remove_field => ["message"]    #这表示匹配成功后是否删除原始信息，这个看个人情况，如果为了节省空间可以考虑删除
    }
    date {
      match => [ "timestamp", "yyyy-MM-dd HH:mm:ss,SSS Z", "MMM dd, yyyy HH:mm:ss a" ]
    }
  }    

  if[type] == "nginx-log" {

    if '"status":"404"' in [message] {
        mutate { replace => { type => "nginx_error_404" } }
    }

    else if '"status":"500"' in [message] {
        mutate { replace => { type => "nginx_error_500" } }
    }

    else if '"status":"502"' in [message] {
        mutate { replace => { type => "nginx_error_502" } }
    }

    else if '"status":"403"' in [message] {
        mutate { replace => { type => "nginx_error_403" } }
    }

    else if '"status":"504"' in [message] {
        mutate { replace => { type => "nginx_error_504" } }
    }

    else if '"status":"200"' in [message] {
        mutate { replace => { type => "nginx_200" } }
    }
    grok {
      remove_field => ["message"]    #这表示匹配成功后是否删除原始信息，这个看个人情况，如果为了节省空间可以考虑删除
    }
  }
}

output {
    elasticsearch { 
        hosts => ["elasticsearch:9200"]
    }

    #stdout { codec => rubydebug }    #输出到屏幕上
}
```
 

/opt/logstash/conf/patterns  下面存放 grok 文件

grok-patterns 文件

```
USERNAME [a-zA-Z0-9._-]+
USER %{USERNAME}
INT (?:[+-]?(?:[0-9]+))
BASE10NUM (?<![0-9.+-])(?>[+-]?(?:(?:[0-9]+(?:\.[0-9]+)?)|(?:\.[0-9]+)))
NUMBER (?:%{BASE10NUM})
BASE16NUM (?<![0-9A-Fa-f])(?:[+-]?(?:0x)?(?:[0-9A-Fa-f]+))
BASE16FLOAT \b(?<![0-9A-Fa-f.])(?:[+-]?(?:0x)?(?:(?:[0-9A-Fa-f]+(?:\.[0-9A-Fa-f]*)?)|(?:\.[0-9A-Fa-f]+)))\b

POSINT \b(?:[1-9][0-9]*)\b
NONNEGINT \b(?:[0-9]+)\b
WORD \b\w+\b
NOTSPACE \S+
SPACE \s*
DATA .*?
GREEDYDATA .*
QUOTEDSTRING (?>(?<!\\)(?>"(?>\\.|[^\\"]+)+"|""|(?>'(?>\\.|[^\\']+)+')|''|(?>`(?>\\.|[^\\`]+)+`)|``))
UUID [A-Fa-f0-9]{8}-(?:[A-Fa-f0-9]{4}-){3}[A-Fa-f0-9]{12}

# Networking
MAC (?:%{CISCOMAC}|%{WINDOWSMAC}|%{COMMONMAC})
CISCOMAC (?:(?:[A-Fa-f0-9]{4}\.){2}[A-Fa-f0-9]{4})
WINDOWSMAC (?:(?:[A-Fa-f0-9]{2}-){5}[A-Fa-f0-9]{2})
COMMONMAC (?:(?:[A-Fa-f0-9]{2}:){5}[A-Fa-f0-9]{2})
IPV6 ((([0-9A-Fa-f]{1,4}:){7}([0-9A-Fa-f]{1,4}|:))|(([0-9A-Fa-f]{1,4}:){6}(:[0-9A-Fa-f]{1,4}|((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3})|:))|(([0-9A-Fa-f]{1,4}:){5}(((:[0-9A-Fa-f]{1,4}){1,2})|:((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3})|:))|(([0-9A-Fa-f]{1,4}:){4}(((:[0-9A-Fa-f]{1,4}){1,3})|((:[0-9A-Fa-f]{1,4})?:((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3}))|:))|(([0-9A-Fa-f]{1,4}:){3}(((:[0-9A-Fa-f]{1,4}){1,4})|((:[0-9A-Fa-f]{1,4}){0,2}:((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3}))|:))|(([0-9A-Fa-f]{1,4}:){2}(((:[0-9A-Fa-f]{1,4}){1,5})|((:[0-9A-Fa-f]{1,4}){0,3}:((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3}))|:))|(([0-9A-Fa-f]{1,4}:){1}(((:[0-9A-Fa-f]{1,4}){1,6})|((:[0-9A-Fa-f]{1,4}){0,4}:((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3}))|:))|(:(((:[0-9A-Fa-f]{1,4}){1,7})|((:[0-9A-Fa-f]{1,4}){0,5}:((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3}))|:)))(%.+)?
IPV4 (?<![0-9])(?:(?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})[.](?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})[.](?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})[.](?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2}))(?![0-9])
IP (?:%{IPV6}|%{IPV4})
HOSTNAME \b(?:[0-9A-Za-z][0-9A-Za-z-]{0,62})(?:\.(?:[0-9A-Za-z][0-9A-Za-z-]{0,62}))*(\.?|\b)
HOST %{HOSTNAME}
IPORHOST (?:%{HOSTNAME}|%{IP})
HOSTPORT (?:%{IPORHOST=~/\./}:%{POSINT})

# paths
PATH (?:%{UNIXPATH}|%{WINPATH})
UNIXPATH (?>/(?>[\w_%!$@:.,-]+|\\.)*)+
TTY (?:/dev/(pts|tty([pq])?)(\w+)?/?(?:[0-9]+))
WINPATH (?>[A-Za-z]+:|\\)(?:\\[^\\?*]*)+
URIPROTO [A-Za-z]+(\+[A-Za-z+]+)?
URIHOST %{IPORHOST}(?::%{POSINT:port})?
# uripath comes loosely from RFC1738, but mostly from what Firefox
# doesn't turn into %XX
URIPATH (?:/[A-Za-z0-9$.+!*'(){},~:;=@#%_\-]*)+
#URIPARAM \?(?:[A-Za-z0-9]+(?:=(?:[^&]*))?(?:&(?:[A-Za-z0-9]+(?:=(?:[^&]*))?)?)*)?
URIPARAM \?[A-Za-z0-9$.+!*'|(){},~@#%&/=:;_?\-\[\]]*
URIPATHPARAM %{URIPATH}(?:%{URIPARAM})?
URI %{URIPROTO}://(?:%{USER}(?::[^@]*)?@)?(?:%{URIHOST})?(?:%{URIPATHPARAM})?

# Months: January, Feb, 3, 03, 12, December
MONTH \b(?:Jan(?:uary)?|Feb(?:ruary)?|Mar(?:ch)?|Apr(?:il)?|May|Jun(?:e)?|Jul(?:y)?|Aug(?:ust)?|Sep(?:tember)?|Oct(?:ober)?|Nov(?:ember)?|Dec(?:ember)?)\b
MONTHNUM (?:0?[1-9]|1[0-2])
MONTHDAY (?:(?:0[1-9])|(?:[12][0-9])|(?:3[01])|[1-9])

# Days: Monday, Tue, Thu, etc...
DAY (?:Mon(?:day)?|Tue(?:sday)?|Wed(?:nesday)?|Thu(?:rsday)?|Fri(?:day)?|Sat(?:urday)?|Sun(?:day)?)

# Years?
YEAR (?>\d\d){1,2}
HOUR (?:2[0123]|[01]?[0-9])
MINUTE (?:[0-5][0-9])
# '60' is a leap second in most time standards and thus is valid.
SECOND (?:(?:[0-5][0-9]|60)(?:[:.,][0-9]+)?)
TIME (?!<[0-9])%{HOUR}:%{MINUTE}(?::%{SECOND})(?![0-9])
# datestamp is YYYY/MM/DD-HH:MM:SS.UUUU (or something like it)
DATE_US %{MONTHNUM}[/-]%{MONTHDAY}[/-]%{YEAR}
DATE_EU %{MONTHDAY}[./-]%{MONTHNUM}[./-]%{YEAR}
ISO8601_TIMEZONE (?:Z|[+-]%{HOUR}(?::?%{MINUTE}))
ISO8601_SECOND (?:%{SECOND}|60)
TIMESTAMP_ISO8601 %{YEAR}-%{MONTHNUM}-%{MONTHDAY}[T ]%{HOUR}:?%{MINUTE}(?::?%{SECOND})?%{ISO8601_TIMEZONE}?
DATE %{DATE_US}|%{DATE_EU}
DATESTAMP %{DATE}[- ]%{TIME}
TZ (?:[PMCE][SD]T|UTC)
DATESTAMP_RFC822 %{DAY} %{MONTH} %{MONTHDAY} %{YEAR} %{TIME} %{TZ}
DATESTAMP_OTHER %{DAY} %{MONTH} %{MONTHDAY} %{TIME} %{TZ} %{YEAR}

# Syslog Dates: Month Day HH:MM:SS
SYSLOGTIMESTAMP %{MONTH} +%{MONTHDAY} %{TIME}
PROG (?:[\w._/%-]+)
SYSLOGPROG %{PROG:program}(?:\[%{POSINT:pid}\])?
SYSLOGHOST %{IPORHOST}
SYSLOGFACILITY <%{NONNEGINT:facility}.%{NONNEGINT:priority}>
HTTPDATE %{MONTHDAY}/%{MONTH}/%{YEAR}:%{TIME} %{INT}

# Shortcuts
QS %{QUOTEDSTRING}

# Log formats
SYSLOGBASE %{SYSLOGTIMESTAMP:timestamp} (?:%{SYSLOGFACILITY} )?%{SYSLOGHOST:logsource} %{SYSLOGPROG}:
COMMONAPACHELOG %{IPORHOST:clientip} %{USER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] "(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})" %{NUMBER:response} (?:%{NUMBER:bytes}|-)
COMBINEDAPACHELOG %{COMMONAPACHELOG} %{QS:referrer} %{QS:agent}

# Log Levels
LOGLEVEL ([A-a]lert|ALERT|[T|t]race|TRACE|[D|d]ebug|DEBUG|[N|n]otice|NOTICE|[I|i]nfo|INFO|[W|w]arn?(?:ing)?|WARN?(?:ING)?|[E|e]rr?(?:or)?|ERR?(?:OR)?|[C|c]rit?(?:ical)?|CRIT?(?:ICAL)?|[F|f]atal|FATAL|[S|s]evere|SEVERE|EMERG(?:ENCY)?|[Ee]merg(?:ency)?)

# Java Logs
JAVATHREAD (?:[A-Z]{2}-Processor[\d]+)
JAVACLASS (?:[a-zA-Z0-9-]+\.)+[A-Za-z0-9$]+
JAVAFILE (?:[A-Za-z0-9_.-]+)
JAVASTACKTRACEPART at %{JAVACLASS:class}\.%{WORD:method}\(%{JAVAFILE:file}:%{NUMBER:line}\)
JAVALOGMESSAGE (.*)
# MMM dd, yyyy HH:mm:ss eg: Jan 9, 2014 7:13:13 AM
CATALINA_DATESTAMP %{MONTH} %{MONTHDAY}, 20%{YEAR} %{HOUR}:?%{MINUTE}(?::?%{SECOND}) (?:AM|PM)
# yyyy-MM-dd HH:mm:ss,SSS ZZZ eg: 2014-01-09 17:32:25,527 -0800
TOMCAT_DATESTAMP 20%{YEAR}-%{MONTHNUM}-%{MONTHDAY} %{HOUR}:?%{MINUTE}(?::?%{SECOND}) %{ISO8601_TIMEZONE}
CATALINALOG %{CATALINA_DATESTAMP:timestamp} %{JAVACLASS:class} %{JAVALOGMESSAGE:logmessage}
# 2014-01-09 20:03:28,269 -0800 | ERROR | com.example.service.ExampleService - something compeletely unexpected happened...
TOMCATLOG %{TOMCAT_DATESTAMP:timestamp} \| %{LOGLEVEL:level} \| %{JAVACLASS:class} - %{JAVALOGMESSAGE:logmessage}
```
 

 

## 容器日志输出

docker里，标准的日志方式是用Stdout, docker 里面配置标准输出，只需要指定： syslog 就可以了。

对于 stdout 标准输出的 docker 日志，我们使用 logstash 来收集日志就可以。

我们在 docker-compose 中配置如下既可：

```
      logging:
        driver: syslog
        options:
          syslog-address: 'tcp://logstash:5000'
```

但是一般来说我们都是文件日志，那么我们就可以直接用filebeat

对于 filebeat 我们使用 官方的 dockerhub 的 prima/filebeat 镜像。

官方的镜像中，我们需要编译一个filebeat.yml 文件， 官方说明中有两种方案：

第一是 -v 挂载 -v /path/filebeat.yml:/filebeat.yml

第二是 dockerfile 的时候

```
FROM prima/filebeat

COPY filebeat.yml /filebeat.yml
```


编译一个 filebeat.yml 文件。

filebeat.yml 支持单一路径的 prospector， 也支持多个 prospector或者每个prospector多个路径。

paths 可使用多层匹配， 如： /var/log/messages* , /var/log/* , /opt/nginx/*/*.log


例：

```
filebeat:
  prospectors:
    -
      paths:
          - "/data/logs/catalina.*.out"
      input_type: filebeat-log
      document_type: tomcat-log
    -  
      paths:
          - "/data/logs/nginx*/logs/*.log"
      input_type: filebeat-log
      document_type: nginx-log
    
  registry_file: /etc/registry/mark
      
output:
  logstash:
    hosts: ["logstash:5044"]

logging:
  files:
    rotateeverybytes: 10485760 # = 10MB
```


filebeat 需要在每台需要采集的机器上面都启动一个容器，或者日志存于分布式存储中。


执行 docker-compose up -d 查看启动的 容器


## 加载 filebeat 模板

进入 elasticsearch 容器中  (  docker exec -it elasticsearch bash  )

```
curl -O https://gist.githubusercontent.com/thisismitch/3429023e8438cc25b86c/raw/d8c479e2a1adcea8b1fe86570e42abab0f10f364/filebeat-index-template.json
    
curl -XPUT 'http://elasticsearch:9200/_template/filebeat?pretty' -d@filebeat-index-template.json
```


filebeat-index-template.json

```
{
  "mappings": {
    "_default_": {
      "_all": {
        "enabled": true,
        "norms": {
          "enabled": false
        }
      },
      "dynamic_templates": [
        {
          "template1": {
            "mapping": {
              "doc_values": true,
              "ignore_above": 1024,
              "index": "not_analyzed",
              "type": "{dynamic_type}"
            },
            "match": "*"
          }
        }
      ],
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "message": {
          "type": "string",
          "index": "analyzed"
        },
        "offset": {
          "type": "long",
          "doc_values": "true"
        },
        "geoip"  : {
          "type" : "object",
          "dynamic": true,
          "properties" : {
            "location" : { "type" : "geo_point" }
          }
        }
      }
    }
  },
  "settings": {
    "index.refresh_interval": "5s"
  },
  "template": "filebeat-*"
}
```


访问 http://kibana-IP:5601 可以看到已经出来 kibana 了，但是还没有数据

 

## 测试

启动一个 nginx 容器

docker-compose

```
  nginx:
    image: alpine-nginx
    networks:
      network-test:
    hostname: nginx
    container_name: nginx
    restart: always
    ports:
      - 80:80
    volumes:
      - /opt/upload/nginx/conf/vhost:/etc/nginx/vhost
      - /opt/upload/nginx/logs:/opt/nginx/logs
```

本地目录 /opt/upload/nginx   必须挂载到 filebeat 容器里面，让filebeat 可以采集到。


## 转换 nginx 格式

将nginx日志格式转换为json 格式,方便后续的分析

```
log_format main
'{"@timestamp":"$time_iso8601",'
'"host":"$server_addr",'
'"clientip":"$remote_addr - $remote_user [$time_local]",'
'"size":$body_bytes_sent,'
'"responsetime":$request_time,'
'"upstreamtime":"$upstream_response_time",'
'"upstreamhost":"$upstream_addr",'
'"http_host":"$host",'
'"url":"$request",'
'"domain":"$host",'
'"xff":"$http_x_forwarded_for",'
'"referer":"$http_referer",'
'"user_agent":"$http_user_agent",'
'"status":"$status"}';
```

最后我们查看效果: 
![此处输入图片的描述][2]
 

 


  [1]: https://jicki.cn/img/posts/elk/1.png
  [2]: https://jicki.cn/img/posts/elk/2.png

