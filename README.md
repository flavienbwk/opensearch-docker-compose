# OpenSearch - Docker - Compose

![OpenSearch version](https://img.shields.io/badge/OpenSearch%20version-1.0.0-blue)

Dockerized cluster architecture for OpenSearch with compose.

## Key concepts

- OpenSearch is [the successor of OpenDistro](https://opendistro.github.io/for-elasticsearch/blog/2021/06/forward-to-opensearch/)
- OpenSearch = Elasticsearch
- OpenSearch Dashboards = Kibana

## Installation

First, you will need to raise your host's ulimits for ElasticSearch to handle high I/O :

```bash
sudo sysctl -w vm.max_map_count=500000
```

Now, we will generate the certificates for the cluster :

```bash
# Copy paste from this project root directory

mkdir -p certs/{ca,os-dashboards,os01,os02,os03}
export OPENDISTRO_DN="/C=FR/ST=IDF/L=PARIS/O=EXAMPLE"   # Edit here and in opensearch.yml

# Root CA
openssl genrsa -out certs/ca/ca.key 2048
openssl req -new -x509 -sha256 -days 1095 -subj "$OPENDISTRO_DN/CN=CA" -key certs/ca/ca.key -out certs/ca/ca.pem

# Admin
openssl genrsa -out certs/ca/admin-temp.key 2048
openssl pkcs8 -inform PEM -outform PEM -in certs/ca/admin-temp.key -topk8 -nocrypt -v1 PBE-SHA1-3DES -out certs/ca/admin.key
openssl req -new -subj "$OPENDISTRO_DN/CN=ADMIN" -key certs/ca/admin.key -out certs/ca/admin.csr
openssl x509 -req -in certs/ca/admin.csr -CA certs/ca/ca.pem -CAkey certs/ca/ca.key -CAcreateserial -sha256 -out certs/ca/admin.pem

# Node 1
openssl genrsa -out certs/os01/os01-temp.key 2048
openssl pkcs8 -inform PEM -outform PEM -in certs/os01/os01-temp.key -topk8 -nocrypt -v1 PBE-SHA1-3DES -out certs/os01/os01.key
openssl req -new -subj "$OPENDISTRO_DN/CN=os01" -key certs/os01/os01.key -out certs/os01/os01.csr
openssl x509 -req -extfile <(printf "subjectAltName=DNS:localhost,IP:127.0.0.1,DNS:os01") -in certs/os01/os01.csr -CA certs/ca/ca.pem -CAkey certs/ca/ca.key -CAcreateserial -sha256 -out certs/os01/os01.pem

# Node 2
openssl genrsa -out certs/os02/os02-temp.key 2048
openssl pkcs8 -inform PEM -outform PEM -in certs/os02/os02-temp.key -topk8 -nocrypt -v1 PBE-SHA1-3DES -out certs/os02/os02.key
openssl req -new -subj "$OPENDISTRO_DN/CN=os02" -key certs/os02/os02.key -out certs/os02/os02.csr
openssl x509 -req -extfile <(printf "subjectAltName=DNS:localhost,IP:127.0.0.1,DNS:os02") -in certs/os02/os02.csr -CA certs/ca/ca.pem -CAkey certs/ca/ca.key -CAcreateserial -sha256 -out certs/os02/os02.pem

# Node 3
openssl genrsa -out certs/os03/os03-temp.key 2048
openssl pkcs8 -inform PEM -outform PEM -in certs/os03/os03-temp.key -topk8 -nocrypt -v1 PBE-SHA1-3DES -out certs/os03/os03.key
openssl req -new -subj "$OPENDISTRO_DN/CN=os03" -key certs/os03/os03.key -out certs/os03/os03.csr
openssl x509 -req -extfile <(printf "subjectAltName=DNS:localhost,IP:127.0.0.1,DNS:os03") -in certs/os03/os03.csr -CA certs/ca/ca.pem -CAkey certs/ca/ca.key -CAcreateserial -sha256 -out certs/os03/os03.pem

# OpenSearch Dashboards
openssl genrsa -out certs/os-dashboards/os-dashboards-temp.key 2048
openssl pkcs8 -inform PEM -outform PEM -in certs/os-dashboards/os-dashboards-temp.key -topk8 -nocrypt -v1 PBE-SHA1-3DES -out certs/os-dashboards/os-dashboards.key
openssl req -new -subj "$OPENDISTRO_DN/CN=os-dashboards" -key certs/os-dashboards/os-dashboards.key -out certs/os-dashboards/os-dashboards.csr
openssl x509 -req -in certs/os-dashboards/os-dashboards.csr -CA certs/ca/ca.pem -CAkey certs/ca/ca.key -CAcreateserial -sha256 -out certs/os-dashboards/os-dashboards.pem

# Cleanup
rm certs/ca/admin-temp.key certs/ca/admin.csr
rm certs/os01/os01-temp.key certs/os01/os01.csr
rm certs/os02/os02-temp.key certs/os02/os02.csr
rm certs/os03/os03-temp.key certs/os03/os03.csr
rm certs/os-dashboards/os-dashboards-temp.key certs/os-dashboards/os-dashboards.csr
unset OPENDISTRO_DN

# Changing permissions
chmod 700 certs/{ca,os-dashboards,os01,os02,os03}
chmod 600 certs/{ca/*,os-dashboards/*,os01/*,os02/*,os03/*}
```

Start the cluster :

```bash
docker-compose up -d
```

Finally, run `securityadmin` to initialize the security plugin :

```bash
docker-compose exec os01 bash -c "chmod +x plugins/opensearch-security/tools/securityadmin.sh && bash plugins/opensearch-security/tools/securityadmin.sh -cd plugins/opensearch-security/securityconfig -icl -nhnv -cacert config/certificates/ca/ca.pem -cert config/certificates/ca/admin.pem -key config/certificates/ca/admin.key -h localhost"
```

> Find all the configuration files in the container's `/usr/share/opensearch/plugins/opensearch-security/securityconfig` directory. You might want to [mount them as volumes](https://opendistro.github.io/for-elasticsearch-docs/docs/install/docker-security/).

Access OpenSearch Dashboards through [https://localhost:5601](https://localhost:5601)

> Default username is `os_dashboards` and password is `os_dashboards`

## Why OpenSearch

- Fully open source (including plugins)
- Fully under Apache 2.0 license
- Advanced security plugin (free)
- Alerting plugin (free)
- Allows you to [perform SQL queries against ElasticSearch](https://opendistro.github.io/for-elasticsearch-docs/docs/sql/)
- Maintained by AWS and used for its cloud services
