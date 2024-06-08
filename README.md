# elk-stack-docker

# Step 1: Generate Certificates

Ensure the ca.crt and ca.key are present in the correct directory.

**Generate the CA Certificate**

```
sudo docker run --rm -v $(pwd):/certs docker.elastic.co/elasticsearch/elasticsearch:8.14.0 \
  /usr/share/elasticsearch/bin/elasticsearch-certutil ca --pem --out /certs/elastic-stack-ca.zip

sudo unzip elastic-stack-ca.zip -d certs
```

**Generate the Node Certificates**

Ensure the ca.crt file exists in the certs/ca directory.

```
sudo docker run --rm -v $(pwd):/certs docker.elastic.co/elasticsearch/elasticsearch:8.14.0 \
  /usr/share/elasticsearch/bin/elasticsearch-certutil cert --pem --ca-cert /certs/ca/ca.crt --ca-key /certs/ca/ca.key --out /certs/elastic-certificates.zip

sudo unzip elastic-certificates.zip -d certs
```

**Give the file permission**

```
sudo chown -R $(whoami):$(whoami) certs
```

Verify the certs directory structure after this step. It should look something like this:

```
certs/
├── ca
│   ├── ca.crt
│   └── ca.key
├── instance
│   ├── instance.crt
│   └── instance.key

```

# Step 2: Generate Service Account Token

Use the correct service account principal for Kibana: elastic/kibana.

```
sudo docker run --rm --network=host -v $(pwd)/certs:/certs docker.elastic.co/elasticsearch/elasticsearch:8.14.0 \
  /usr/share/elasticsearch/bin/elasticsearch-service-tokens create elastic/kibana default

```

This should generate a service account token for Kibana.

# Step 3: Create and Configure Docker Compose

Ensure your docker-compose.yml is correctly set up to reference the generated certificates and the service token.

```
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.14.0
    container_name: elasticsearch
    environment:
      - "discovery.type=single-node"
      - "xpack.security.enabled=true"
      - "xpack.security.http.ssl.enabled=true"
      - "xpack.security.http.ssl.key=/usr/share/elasticsearch/config/certs/ca.key"
      - "xpack.security.http.ssl.certificate=/usr/share/elasticsearch/config/certs/ca.crt"
      - "xpack.security.http.ssl.certificate_authorities=/usr/share/elasticsearch/config/certs/ca.crt"
      - "ELASTIC_PASSWORD=your_elastic_password"
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data
      - ./certs:/usr/share/elasticsearch/config/certs

  kibana:
    image: docker.elastic.co/kibana/kibana:8.14.0
    container_name: kibana
    environment:
      ELASTICSEARCH_HOSTS: "https://elasticsearch:9200"
      SERVER_SSL_ENABLED: true
      SERVER_SSL_CERTIFICATE: /usr/share/kibana/config/certs/ca.crt
      SERVER_SSL_KEY: /usr/share/kibana/config/certs/ca.key
      ELASTICSEARCH_SERVICE_TOKEN: "your_service_token_here"
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    volumes:
      - ./certs:/usr/share/kibana/config/certs

  logstash:
    build:
      context: ./logstash
    container_name: logstash
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - ./certs:/usr/share/logstash/config/certs
    depends_on:
      - elasticsearch
    ports:
      - "5044:5044"
    environment:
      - "xpack.monitoring.elasticsearch.hosts=https://elasticsearch:9200"
      - "xpack.monitoring.elasticsearch.username=elastic"
      - "xpack.monitoring.elasticsearch.password=your_elastic_password"
      - "xpack.monitoring.elasticsearch.ssl.certificate_authority=/usr/share/logstash/config/certs/ca.crt"
      - "xpack.monitoring.elasticsearch.ssl.verification_mode=certificate"

volumes:
  es_data:

```

# Step 4: Create Dockerfile for Logstash


Ensure you have a Dockerfile for Logstash that installs the MongoDB plugin.


```
# logstash/Dockerfile
FROM docker.elastic.co/logstash/logstash:8.14.0
RUN bin/logstash-plugin install logstash-input-mongodb
```

# Step 5: Create Logstash Configuration File


Create a logstash.conf file in your working directory.


```
input {
  mongodb {
    uri => 'mongodb://68.183.81.8:27011/mongoose_db'
    placeholder_db_dir => '/usr/share/logstash/config'
    placeholder_db_name => 'logstash_sqlite.db'
    collection => 'logs'
    batch_size => 5000
    since_time => 0
  }
}

output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    user => "elastic"
    password => "your_elastic_password"
    ssl => true
    cacert => "/usr/share/logstash/config/certs/ca.crt"
    index => "logs"
  }
}

```


# Step 6: Start Docker Compose

Run the Docker Compose command to start your services:


```
sudo docker-compose up -d
```

**Conclusion**

With these steps, you should have a fully functional Elastic Stack (Elasticsearch, Logstash, and Kibana) running locally using Docker Compose. Ensure all paths, passwords, and tokens are correctly set and securely stored.

* Elasticsearch: Accessible at https://localhost:9200
* Kibana: Accessible at https://localhost:5601
* Logstash: Configured to read from MongoDB and write to Elasticsearch


