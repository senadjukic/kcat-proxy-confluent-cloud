# kcat-proxy-confluent-cloud
kcat / kafkacat with Confluent Cloud and Nginx proxy

## Documentation
- https://docs.confluent.io/platform/current/app-development/kafkacat-usage.html
- https://www.zghurskyi.com/kafkacat-overview/
- http://nginx.org/en/docs/varindex.html
- https://docs.confluent.io/cloud/current/networking/ccloud-console-access.html#configure-a-proxy

## Instructions
**Note:** Create topic (CCLOUD_TOPIC) before Produce or Consume

**Create environment variable file**
```
cat > $PWD/.env <<EOF
CCLOUD_BOOTSTRAP_SERVER=pkc-xxxxx.eu-central-1.aws.confluent.cloud:9092
CCLOUD_API_KEY=
CCLOUD_API_SECRET=
CCLOUD_BOOTSTRAP_SERVER_NO_PORT=pkc-xxxxx.eu-central-1.aws.confluent.cloud
CCLOUD_TOPIC=test
EOF
```

**Start docker compose**
```
# Consumer Mode
docker compose -f kcat-consume-nginx.yaml --env-file .env up

# Producer Mode
docker compose -f kcat-produce-nginx.yaml --env-file .env up
```

## Single kcat conatainers run without Nginx proxy 
**List brokers and topics in cluster**
```
docker run --tty --rm \
        confluentinc/cp-kafkacat \
        kafkacat -b ${CCLOUD_BOOTSTRAP_SERVER} \
        -X security.protocol=SASL_SSL \
        -X sasl.mechanisms=PLAIN \
        -X sasl.username=${CCLOUD_API_KEY} \
        -X sasl.password=${CCLOUD_API_SECRET} \
        -X api.version.request=true \
        -L
```

**Produce to topic**
```
docker run --interactive --rm \
        confluentinc/cp-kafkacat \
        kafkacat -b ${CCLOUD_BOOTSTRAP_SERVER} \
        -X security.protocol=SASL_SSL \
        -X sasl.mechanisms=PLAIN \
        -X sasl.username=${CCLOUD_API_KEY} \
        -X sasl.password=${CCLOUD_API_SECRET} \
        -X api.version.request=true \
        -t ${CCLOUD_TOPIC} \
        -K: \
        -P <<EOF
1:{"order_id":1,"order_ts":1670574113,"total_amount":10.00,"customer_name":"Senad Jukic"}
2:{"order_id":2,"order_ts":1670574114,"total_amount":20.00,"customer_name":"Max Muster"}
3:{"order_id":3,"order_ts":1670574115,"total_amount":30.00,"customer_name":"Franz Kafka"}
EOF
```

**Produce to topic from file**
```
cat >> produce.txt <<EOF
1:{"order_id":1,"order_ts":1670574113,"total_amount":10.00,"customer_name":"Senad Jukic"}
2:{"order_id":2,"order_ts":1670574114,"total_amount":20.00,"customer_name":"Max Muster"}
3:{"order_id":3,"order_ts":1670574115,"total_amount":30.00,"customer_name":"Franz Kafka"}
EOF

docker run --interactive --rm \
        --volume produce.txt:/data/produce.txt \
        confluentinc/cp-kafkacat \
        kafkacat -b ${CCLOUD_BOOTSTRAP_SERVER} \
        -X security.protocol=SASL_SSL \
        -X sasl.mechanisms=PLAIN \
        -X sasl.username=${CCLOUD_API_KEY} \
        -X sasl.password=${CCLOUD_API_SECRET} \
        -X api.version.request=true \
        -t ${CCLOUD_TOPIC} \
        -K: \
        -P -l /data/produce.txt
```

**Consume from topic**
```
docker run --tty \
        confluentinc/cp-kafkacat \
        kafkacat -b ${CCLOUD_BOOTSTRAP_SERVER} \
        -X security.protocol=SASL_SSL \
        -X sasl.mechanisms=PLAIN \
        -X sasl.username=${CCLOUD_API_KEY} \
        -X sasl.password=${CCLOUD_API_SECRET} \
        -X api.version.request=true \
        -t ${CCLOUD_TOPIC} \
        -C -f '\nKey (%K bytes): %k\t\nValue (%S bytes): %s\n\Partition: %p\tOffset: %o\n--\n'
```

## Troubleshoot
```
# Change resolver to cloud specific resolver
169.254.169.253 for AWS
168.63.129.16 for Azure
169.254.169.254 for GCP

# Run single nginx
docker run -d \
--name nginx1 \
-p 9092:9092 \
-p 443:443 \
-v $(pwd)/default.conf:/etc/nginx/nginx.conf \
-v $(pwd)/stream_9092.log:/var/log/nginx/stream_9092.log \
-v $(pwd)/stream_443.log:/var/log/nginx/stream_443.log \
ossobv/nginx

# Get IP from Nginx container
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nginx1

# Manual run for listing brokers
kafkacat -b ${CCLOUD_BOOTSTRAP_SERVER} -L -X security.protocol=SASL_SSL -X sasl.mechanisms=PLAIN -X sasl.username=${CCLOUD_API_KEY} -X sasl.password=${CCLOUD_API_SECRET} -X api.version.request=true
```
