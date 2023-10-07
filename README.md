# Sample configuration: logstash output to http (or elastic)

In the following configuration, we see the following scenarios:

- either configure logstash to
  - read a sample file: logs/access.log
  - and send to output https endpoint (Option 1)
  - (or send to logstash output elasticsearch endpoint) (Option 2)

- either configure filebeat and logstash to
  - filebeat: read a sample file: logs/access.log
  - filebeat: send to local logstash
  - logstash: send to output https endpoint (Option 3)

(As filebeat does not support http output plugin, we use an intermediate logstash transformer)


Prereq:
- a logstash app (input http) is listening on your scalingo account
- LOGSTASH_URL: https://logstash-endpoint
- LOGSTASH_USER
- LOGSTASH_PASSWORD
- sample web log file in `logs/access.log`

## Option 1: Configure logstash to send output to https endpoint
### Prepare env file
```
# .env
LOGSTASH_URL=https://logstash-endpoint....
LOGSTASH_USER=_REPLACE_HERE_
LOGSTASH_PASSWORD=_REPLACE_HERE_
```

### Start logstash docker-compose

```
docker-compose up
```
```
# to stop and clean
docker-compose stop
docker-compose rm
```

### Config file in details

In the `logstash/pipeline/logstash.conf`, define input and output

```
# read logs/access.log
input {
  file {
    path => "/logs/access.log"
    type => "nginx"
    start_position => "beginning"
  }
}

# Send the message to output http endpoint
output {
  http {
    url => "${LOGSTASH_URL}"
    user => "${LOGSTASH_USER}"
    password => "${LOGSTASH_PASSWORD}"
    http_method => "put"
    format => "json_batch"
    http_compression => true
    retry_non_idempotent => true
  }
}
```

In the `docker-compose.yml`, define logstash config file and logs dir to inject
```yaml
version: '3'
services:
  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.3
    volumes:
      - ./logstash/pipeline/logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - ./logs/:/logs/:ro
    environment:
      # to send to http logstash
      LOGSTASH_URL: ${LOGSTASH_URL:-}
      LOGSTASH_USER: ${LOGSTASH_USER:-}
      LOGSTASH_PASSWORD: ${LOGSTASH_PASSWORD:-}
    networks:
      - elk

networks:
  elk:
    driver: bridge

```

## Option 2: Configure logstash to send to local ssh tunnel/elasticsearch

- open ssh tunnel to SCALINGO_ELASTICSEARCH_URL with `scalingo db-tunnel`
- configure logstash to send output to ssh tunner / elastic

The following command will open an ssh tunnel from local port 10000 to your hidden elasticsearch DB.
```
# open https://localhost:10000 <- SSH -> SCALINGO_ELASTICSEARCH_URL
scalingo -a $SCALINGO_APP db-tunnel -i $HOME/.ssh/scalingo-key SCALINGO_ELASTICSEARCH_URL
```

```
# To check tunnel works
curl "https://$ELASTICSEARCH_USER:$ELASTICSEARCH_PASSWORD@localhost:10000/_cat/indices?v"  -k
```

### Prepare env file
```
# .env
ELASTICSEARCH_HOST=your_local_ip:10000
ELASTICSEARCH_USER=_REPLACE_HERE_
ELASTICSEARCH_PASSWORD=_REPLACE_HERE_

```

### Start logstash docker-compose

```
docker-compose up
```
```
# to stop and clean
docker-compose stop
docker-compose rm
```


### Config file in details

In the `logstash/pipeline/logstash.conf`, define input and output

```
# read logs/access.log
input {
  file {
    path => "/logs/access.log"
    type => "nginx"
    start_position => "beginning"
  }
}

# Send the message to elasticsearch output
output {
   elasticsearch {
    hosts => "${ELASTICSEARCH_HOST}"
    user => "${ELASTICSEARCH_USER}"
    password => "${ELASTICSEARCH_PASSWORD}"
    ssl => true
    ssl_certificate_verification => false
    index => "change-me-%{+YYYY.MM.dd}"
  }
}
```

In the `docker-compose.yml`, define logstash config file and logs dir to inject
```yaml
version: '3'
services:
  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.3
    volumes:
      - ./logstash/pipeline/logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - ./logs/:/logs/:ro
    environment:
      # to send to elasticsearch
      ELASTICSEARCH_HOST: ${ELASTICSEARCH_HOST:-}
      ELASTICSEARCH_USER: ${ELASTICSEARCH_USER:-}
      ELASTICSEARCH_PASSWORD: ${ELASTICSEARCH_PASSWORD:-}
    networks:
      - elk

networks:
  elk:
    driver: bridge

```


## Option 3: Configure filebeat and logstash to send output to https endpoint
### Prepare env file
```
# .env
LOGSTASH_URL=https://logstash-endpoint....
LOGSTASH_USER=_REPLACE_HERE_
LOGSTASH_PASSWORD=_REPLACE_HERE_
```

### Start filebeat/logstash docker-compose

```
docker-compose -f docker-compose-filebeat.yml up
```
```
# to stop and clean
docker-compose -f docker-compose-filebeat.yml stop
docker-compose -f docker-compose-filebeat.yml rm
```

### Config file in details

In the `filebeat/filebeat.yml`, define filebeats modules and logstash output
```
logging.level: info
filebeat.modules:
  - module: nginx
    ingress_controller:
      enable: false
    error:
      enable: false
    access:
      enable: true
      var.paths: [ "/logs/access.log" ]
output:
  logstash:
    enabled: true
    hosts: [ "logstash:5044" ]
```

In the `logstash/pipeline/logstash-filebeat.conf`, define input and output

```
# read logs/access.log
input {
  beats {
    port => 5044
  }
}

# Send the message to output http endpoint
output {
  http {
    url => "${LOGSTASH_URL}"
    user => "${LOGSTASH_USER}"
    password => "${LOGSTASH_PASSWORD}"
    http_method => "put"
    format => "json_batch"
    http_compression => true
    retry_non_idempotent => true
  }
}
```

In the `docker-compose-filebeat.yml`, define filebeat/logstash config file and logs dir to inject
```yaml
version: '3'
services:
  filebeat:
    image: docker.elastic.co/beats/filebeat:${KIBANA_VERSION:-7.17.13}
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      # logs dirs
      - ./logs/:/logs/:ro
    # disable strict permission checks
    command: ["--strict.perms=false"]
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:${LOGSTASH_VERSION:-7.17.3}
    volumes:
      - ./logstash/pipeline/logstash.conf:/usr/share/logstash/pipeline/logstash-filebeat.conf:ro
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
    environment:
      #
      # send to http endpoint
      #
      LOGSTASH_URL: ${LOGSTASH_URL:-}
      LOGSTASH_USER: ${LOGSTASH_USER:-}
      LOGSTASH_PASSWORD: ${LOGSTASH_PASSWORD:-}
    networks:
      - elk
    depends_on:
      - filebeat

networks:
  elk:
    driver: bridge
```
