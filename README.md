<!--
+----------+-----------+------------+--------------+----------+
| Server   | Node Name | Attributes | Roles        | JVM Heap |
+----------+-----------+------------+--------------+----------+
| master-1 | master-1  |            | master       | 768m     |
+----------+-----------+------------+--------------+----------+
| data-1   | data-1    | temp=hot   | data, ingest | 2g       |
+----------+-----------+------------+--------------+----------+
| data-2   | data-2    | temp=warm  | data, ingest | 2g       |
+----------+-----------+------------+--------------+----------+
-->

## Installing Elasticsearch in RPM systems

- Import the gpg key:

```bash
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

- Download the Elasticsearch package:

```bash
curl --remote-name \
  https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.0-x86_64.rpm
```

- Install the Elasticsearch package:

```bash
sudo rpm --install elasticsearch-7.6.0-x86_64.rpm
```

- Enable Elasticsearch service:

```bash
sudo systemctl enable elasticsearch
```

## Custom configurations:

### Master node:

- Elasticsearch config:

```yml
# /etc/elasticsearch/elasticsearch.yml

cluster.name: playground
node.name: master-1
network.host: [ _local_, _site_ ]
# Master IP addresses
discovery.seed_hosts: [ "<PRIVATE_MASTER_IP_ADDR>" ]
cluster.initial_master_nodes: [ "master-1" ]
node.master: true
node.data: false
node.ingest: false
node.ml: false
```

- JVM config:

```bash
# /etc/elasticsearch/jvm.options

# Xms Initial size of total heap space
# Xmx Maximum size of total heap space

-Xms768m
-Xmx768m
```

- Start the Elasticsearch service:

```bash
sudo systemctl start elasticsearch
```

### Data node 1:

- Elasticsearch config:

```yml
# /etc/elasticsearch/elasticsearch.yml

cluster.name: playground
node.name: data-1
node.attr.temp: hot
network.host: [ _local_, _site_ ]
# Master IP addresses
discovery.seed_hosts: [ "<PRIVATE_MASTER_IP_ADDR>" ]
cluster.initial_master_nodes: [ "master-1" ]
node.master: false
node.data: true
node.ingest: true
node.ml: false
```

- JVM config:

```bash
# /etc/elasticsearch/jvm.options

# Xms Initial size of total heap space
# Xmx Maximum size of total heap space

-Xms1g # default
-Xmx1g # default
```

- Start the Elasticsearch service:

```bash
sudo systemctl start elasticsearch
```

### Data node 2:

- Elasticsearch config:

```yml
# /etc/elasticsearch/elasticsearch.yml

cluster.name: playground
node.name: data-2
node.attr.temp: warm
network.host: [ _local_, _site_ ]
# Master IP addresses
discovery.seed_hosts: [ "<PRIVATE_MASTER_IP_ADDR>" ]
cluster.initial_master_nodes: [ "master-1" ]
node.master: false
node.data: true
node.ingest: true
node.ml: false
```

- JVM config:

```bash
# /etc/elasticsearch/jvm.options

# Xms Initial size of total heap space
# Xmx Maximum size of total heap space

-Xms1g # default
-Xmx1g # default
```

- Start the Elasticsearch service:

```bash
sudo systemctl start elasticsearch
```

## Testing Cluster

- Check the logs at:

```bash
less /var/log/elasticsearch/playground.log
```

- Check with local requests:

```bash
curl http://localhost:9200/_cat/nodes?v
curl http://localhost:9200/_cat/health?v
```

## Installing Kibana in RPM systems

- On master node, download the Kibana package:

```bash
curl --remote-name https://artifacts.elastic.co/downloads/kibana/kibana-7.6.0-x86_64.rpm
```

- Install the Kibana package:

```bash
sudo rpm --install kibana-7.6.0-x86_64.rpm
```

- Enable Kibana service:

```bash
sudo systemctl enable kibana
```

- Kibana config:

```yml
# /etc/kibana/kibana.yml

server.port: 8080
server.host: "<PRIVATE_MASTER_IP_ADDR>"
elasticsearch.hosts: [ "http://localhost:9200" ]
```

- Start the Kibana service:

```bash
sudo systemctl start kibana
```

## Creating a Public Key Infrastructure (PKI)

- Create a Certificate Authority file:

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil ca \
  --out /etc/elasticsearch/ca \
  --pass elastic_ca
```

- Create a certificate for master node:

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil cert \
  --ca /etc/elasticsearch/ca \
  --ca-pass elastic_ca \
  --name master_1 \
  --dns <MASTER_DNS_ADDRESS> \
  --ip <MASTER_PRIVATE_IP_ADDRESS> \
  --out /etc/elasticsearch/master-1 \
  --pass elastic_master_1
```

- Create certificates for data nodes:

```bash
# Data 1

/usr/share/elasticsearch/bin/elasticsearch-certutil cert \
  --ca /etc/elasticsearch/ca \
  --ca-pass elastic_ca \
  --name data_1 \
  --dns <DATA_1_DNS_ADDRESS> \
  --ip <DATA_1_PRIVATE_IP_ADDRESS> \
  --out /etc/elasticsearch/data-1 \
  --pass elastic_data_1

# Data 2

/usr/share/elasticsearch/bin/elasticsearch-certutil cert \
  --ca /etc/elasticsearch/ca \
  --ca-pass elastic_ca \
  --name data_2 \
  --dns <DATA_2_DNS_ADDRESS> \
  --ip <DATA_2_PRIVATE_IP_ADDRESS> \
  --out /etc/elasticsearch/data-2 \
  --pass elastic_data_2
```

- Move certificates to respective nodes:

```bash
# Data 1

chown <YOUR_USER>:<YOUR_USER> data-1

scp data-1 <DATA_1_PRIVATE_IP_ADDRESS>:/tmp

chown root:elasticsearch data-1 # at data 1 node and move to /etc/elasticsearch

# Data 2

chown <YOUR_USER>:<YOUR_USER> data-2

scp data-2 <DATA_2_PRIVATE_IP_ADDRESS>:/tmp

chown root:elasticsearch data-2 # at data 2 node and move to /etc/elasticsearch
```

## Encrypt the Transport Network

- Change certificate permissions file:

```bash
chmod 640 [master-1, data-1, data-2]
```

### Node Configurations:

- Elasticsearch config:

```yml
# /etc/elasticsearch/elasticsearch.yml

xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: full
xpack.security.transport.ssl.keystore.path: [master-1, data-1, data-2] # certificate name
xpack.security.transport.ssl.truststore.path: [master-1, data-1, data-2] # certificate name
```

- Add pass-phrase to Elasticsearch keystore:

```bash
# keystore

/usr/share/elasticsearch/bin/elasticsearch-keystore add \
  xpack.security.transport.ssl.keystore.secure_password

> type the pass-phrase


# truststore

/usr/share/elasticsearch/bin/elasticsearch-keystore add \
  xpack.security.transport.ssl.truststore.secure_password

> type the pass-phrase
```

- Restart Elasticsearch:

```bash
sudo systemctl restart elasticsearch
```

## Define Elasticsearch user passwords

- Redefine default passwords:

```bash
# elastic, apm_system, kibana, logstash_system, beats_system, remote_monitoring_user

/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```

- Kibana config:

```yml
# /etc/kibana/kibana.yml

elasticsearch.username: "kibana"
elasticsearch.password: "<KIBANA_USER_PASSWORD>"
```

- Restart Kibana:

```bash
sudo systemctl restart kibana
```

## Encrypt the Client Network

**IMPORTANT**: In a production environment, is highly recommend to use a global signed certificate, instead of a self-signed certificate.

### Node Configurations:

- Elasticsearch config:

```yml
# /etc/elasticsearch/elasticsearch.yml

xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: [master-1, data-1, data-2]
xpack.security.http.ssl.truststore.path: [master-1, data-1, data-2]
```

- Add pass-phrase to Elasticsearch keystore:

```bash
# keystore

/usr/share/elasticsearch/bin/elasticsearch-keystore add \
  xpack.security.http.ssl.keystore.secure_password

> type the pass-phrase


# truststore

/usr/share/elasticsearch/bin/elasticsearch-keystore add \
  xpack.security.http.ssl.truststore.secure_password

> type the pass-phrase
```

- Restart Elasticsearch:

```bash
sudo systemctl restart elasticsearch
```

- Kibana config:

```yml
# /etc/kibana/kibana.yml

elasticsearch.hosts: [ "https://localhost:9200" ]
elasticsearch.ssl.verificationMode: none
```

- Restart Kibana:

```bash
sudo systemctl restart kibana
```

## Create Roles

- Create a role with read-only permissions to all indexes:

```json
POST _secutiry/role/read_only
{
  "indices": [
    {
      "names": [ "*" ],
      "privileges": [ "read" ]
    }
  ]
}

GET _security/role/read_only
```

## Create Users

- Create a user:

```json
POST _security/user/new_user
{
  "roles": [ "read_only", "kibana_user" ],
  "full_name": "User Full Name",
  "email": "user@company.com",
  "password": "mypwd123"
}

GET _security/user/new_user
```

## Create Indices

- List indices:

```bash
GET _cat/indices?v
```

- Index's basic structure:

```json
PUT my_first_index_1
{
  "aliases": {
    "my_first_index": {}
  },
  "mappings": {
    "properties": {
      "field_1": {
        "type": "keyword"
      },
      "field_2": {
        "properties": {
          "sub_field_1": {
            "type": "integer"
          }
        }
      }
    }
  },
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  }
}

GET my_first_index_1

GET my_first_index // using index alias

DELETE my_first_index_1
```

## Data Samples

- Account data:

```bash
curl --remote-name https://raw.githubusercontent.com/linuxacademy/content-elasticsearch-deep-dive/master/sample_data/accounts.json
```

- Logs data:

```bash
curl --remote-name https://raw.githubusercontent.com/linuxacademy/content-elasticsearch-deep-dive/master/sample_data/logs.json
```

- Shakespeare data:

```bash
curl --remote-name https://raw.githubusercontent.com/linuxacademy/content-elasticsearch-deep-dive/master/sample_data/shakespeare.json
```

## Bulk API

- Importing **[accounts, shakespeare, logs]** data:

```bash
curl \
  --user elastic \
  --insecure \
  --header 'Content-Type: application/x-ndjson' \
  --request POST \
  https://localhost:9200/[ accounts, shakespeare, logs ]/_bulk \
  --data-binary @[ accounts, shakespeare, logs ].json
```

- Refresh manually the indices:

```bash
POST /bank/_refresh

POST /shakespeare/_refresh

POST /logs/_refresh
```

## CRUD with indices

- To read a record in an indice:

```json
GET bank/_doc/1000
```

- To create a new record in an indice:

```json
PUT bank/_doc/1000
{
  "account_number" : 1000,
  "balance" : 1000000,
  "firstname" : "Afonso",
  "lastname" : "Costa",
  "age" : 24,
  "gender" : "M",
  "address" : "123 Main Street",
  "employer" : "Elastic",
  "email" : "afonso@elastic.com",
  "city" : "São Paulo",
  "state" : "SP"
}
```

- To update an existence record in an indice:

```json
POST bank/_update/1000
{
  "doc": {
    "city" : "Minas Gerais",
    "state" : "MG",
    "favorite_color": "red"
  }
}
```

- To delete a record from an indice:

```json
DELETE bank/_doc/1000
```

## Non-Analyzed Search

- To make a term level search:

```json
GET bank/_search?size=1
{
  "query": {
    "term": {
      "state.keyword": {
        "value": "CO"
      }
    }
  }
}

GET bank/_search
{
  "size": 50,
  "query": {
    "terms": {
      "state.keyword": [
        "CO",
        "PA",
        "NV"
      ]
    }
  }
}
```

## Analyzed Search

- To generate tokens from a given sentence:

```json
GET _analyze
{
  "analyzer": "english",
  "text": "The QUICK brown Foxes jumped over the fence."
}
```

- To make a full-text analyzed search:

```json
GET shakespeare/_search
{
  "query": {
    "match": {
      "text_entry": "King lord"
    }
  }
}
```

- To make a full-text analyzed search with combined tokens:

```json
GET shakespeare/_search
{
  "query": {
    "match_phrase": {
      "text_entry": "My lord"
    }
  }
}
```

## Aggregations

### Metric Aggregations

- To aggregate metrics:

```json
GET bank/_search
{
  "size": 0,
  "aggs": {
    "avg_age": {
      "avg": {
        "field": "age"
      }
    },
    "max_age": {
      "max": {
        "field": "age"
      }
    },
    "min_age": {
      "min": {
        "field": "age"
      }
    }
  }
}
```

### Bucket Aggregations

- To bucketize aggregations:

```json
GET logs/_search
{
  "size": 0,
  "aggs": {
    "events_per_day": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "day"
      }
    }
  }
}

GET bank/_search
{
  "size": 0,
  "aggs": {
    "accounts_per_state": {
      "terms": {
        "field": "state.keyword",
        "size": 50
      }
    }
  }
}
```

### Sub-Aggregations

- To combine both metric and bucket aggregations:

```json
GET logs/_search
{
  "size": 0,
  "aggs": {
    "extension": {
      "terms": {
        "field": "extension.keyword",
        "size": 10
      },
      "aggs": {
        "bytes": {
          "avg": {
            "field": "bytes"
          }
        }
      }
    }
  }
}

GET bank/_search
{
  "size": 0,
  "aggs": {
    "states": {
      "terms": {
        "field": "state.keyword",
        "size": 50
      },
      "aggs": {
        "cities": {
          "cardinality": {
            "field": "city.keyword"
          }
        }
      }
    }
  }
}
```

## Pipeline Aggregations

- To combine aggregations through pipelines:

```json
GET logs/_search
{
  "size": 0,
  "aggs": {
    "hour": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "hour"
      },
      "aggs": {
        "clients": {
          "cardinality": {
            "field": "clientip.keyword"
          }
        },
        "sum_of_clients": {
          "cumulative_sum": {
            "buckets_path": "clients"
          }
        },
        "clients_per_minute": {
          "derivative": {
            "buckets_path": "sum_of_clients",
            "unit": "1m"
          }
        }
      }
    },
    "min_clients": {
      "min_bucket": {
        "buckets_path": "hour>clients"
      }
    },
    "max_clients": {
      "max_bucket": {
        "buckets_path": "hour>clients"
      }
    }
  }
}
```

- Bank example:

1.  How many unique employers are there among our account holders?
2.  How many accounts do we have in each of the 50 US states?
3.  What is the average balance for each of the 50 US states, and what state has the maximum average balance?

```json
GET bank/_search
{
  "size": 0,
  "aggs": {
    "employers": {
      "cardinality": {
        "field": "employer.keyword"
      }
    },
    "states": {
      "terms": {
        "field": "state.keyword",
        "size": 50
      },
      "aggs": {
        "balance_sum": {
          "sum": {
            "field": "balance"
          }
        },
        "balance_avg": {
          "avg": {
            "field": "balance"
          }
        }
      }
    },
    "max_avg_balance": {
      "max_bucket": {
        "buckets_path": "states>balance_avg"
      }
    }
  }
}
```

## Diagnose Shard Issues

- To check indices status:

```json
GET _cat/indices?v
```

- To show if there is some problem with a given index:

```json
GET _cat/shards/logs?v
```

- To show if there is some problem with shards:

```json
GET _cluster/allocation/explain
{
  "index": "logs",
  "shard": 0,
  "primary": true
}
```

- To configure manually a given index:

```json
PUT logs/_settings
{
  "index.routing.allocation.include._name": "data-3",
  "number_of_replicas": 3
}
```
