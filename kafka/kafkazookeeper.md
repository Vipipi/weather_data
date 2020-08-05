# [CS502] Week 1 实战课1 - Kafka & Data Producer
## Run Kafka & ZooKeeper in Docker
Docker is a convenient way to launch systems which requires extensive configuration work. We will use docker to launch ZooKeeper and Kafka:
https://github.com/confluentinc/docker-images


**Bash Command:**
```bash
docker rm -f $(docker ps -a -q)
docker run -d -p 2181:2181 -p 2888:2888 -p 3888:3888 --name zookeeper confluent/zookeeper
docker run -d -p 9092:9092 -e KAFKA_ADVERTISED_HOST_NAME=localhost -e KAFKA_ADVERTISED_PORT=9092 --name kafka --link zookeeper:zookeeper confluent/kafka
```

## Weather API
We need to install **kafka-python**, **requests** and **schedule** package
```bash
python3 -m venv cs_503_env
source cs_503_env/bin/activate

pip3 install schedule
pip3 install kafka-python
pip3 install requests

pip3 freeze > requirements.txt
```

**data-producer.py**
```python
import argparse
import atexit
import json
import logging
import requests
import schedule
import time
import sys

from kafka import KafkaProducer
from kafka.errors import KafkaError, KafkaTimeoutError

formatter = logging.Formatter('[%(asctime)s] - %(funcName)s - %(message)s', datefmt='%a, %d %b %Y %H:%M:%S')
fh = logging.FileHandler('./producer.log')
sh = logging.StreamHandler(sys.stdout)
fh.setFormatter(formatter)
sh.setFormatter(formatter)

logger = logging.getLogger('data-producer')
logger.setLevel(logging.DEBUG)

logger.addHandler(fh)
logger.addHandler(sh)

API_BASE = 'http://localhost:3001'

def fetch_weather_data(producer, topic_name):
    """
    Helper function to retrieve data and send it to Kafka
    :param producer: KafkaProducer
    :param topic_name: topic name of/for Kafka
    :return: None
    """

    logger.debug("Start to fetch weather data.")

    try:
        response = requests.get('%s/weather' % API_BASE)

        weather_data = response.json()['weather-data']
        logger.debug('Retrieved weather data: %s', weather_data)

        timestamp = time.time()

        payload = {
            'WeatherData': str(weather_data),
            'Timestamp': str(timestamp)
        }

        producer.send(topic=topic_name, value=json.dumps(payload).encode('utf-8'), timestamp_ms=int(time.time()) * 1000)
        logger.debug('Sent weather information to Kafka.')

    except KafkaTimeoutError as time_error:
        logger.warning('Failed to send message to kafka, caused by: %s', time_error.message)

    except Exception as e:
        logging.warning('Failed to fetch weather data because %s', e)


def shudown_hook(producer):
    try:
        producer.flush(10)
    except KafkaError as kafka_error:
        logger.warning('Failed to fluash pending message to kafka, cased by: %s', kafka_error.message)
    finally:
        try:
            producer.close(10)
            logger.debug('Kafka producer closed successfully')
        except Exception as e:
            logger.warning('Failed to close kafka connection, caused by: %s', e.message)

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('topic_name', help='the kafka topic push to.')
    parser.add_argument('kafka_broker', help='the address of the kafka broker.')

    # Parse argument
    args = parser.parse_args()
    topic_name = args.topic_name
    kafka_broker = args.kafka_broker

    # Instantiate a simple kafka producer
    producer = KafkaProducer(bootstrap_servers=kafka_broker)
    schedule.every(1).second.do(fetch_weather_data, producer, topic_name)

    # Setup proper shutdown hook
    atexit.register(shudown_hook, producer)

    while True:
        schedule.run_pending()
        time.sleep(1)


```

Execute it:
```bash
python3 data-producer.py test 127.0.0.1:9092
```

## Data Consumer
**data-consumer.py**

``` python
import argparse

from kafka import KafkaConsumer

def consume(topic_name, kafka_broker):
    consumer = KafkaConsumer(topic_name, bootstrap_servers=kafka_broker, auto_offset_reset='smallest')

    for message in consumer:
        print(message)

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('topic_name')
    parser.add_argument('kafka_broker')

    args = parser.parse_args()
    topic_name = args.topic_name
    kafka_broker = args.kafka_broker

    consume(topic_name, kafka_broker)
```

Execute and start consuming messages:
```bash
python3 data-consumer.py test 127.0.0.1:9092
```

