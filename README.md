# Sample configuration: logstash output to http (or elastic)

This configuration do the following:
- configure logstash to
  - read a sample file: logs/access.log
  - send to output https endpoint (Option 1)
  - (or send to logstash output elasticsearch endpoint) (Option 2)

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

## Configure ssh tunnel to SCALINGO_ELASTICSEARCH_URL (scalingo db-tunnel)
This command will open an ssh tunnel from local port 10000 to your hidden elasticsearch DB

```
# https://localhost:10000 <- SSH -> SCALINGO_ELASTICSEARCH_URL
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
ELASTICSEARCH_USER=
ELASTICSEARCH_PASSWORD=
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

