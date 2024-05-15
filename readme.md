### Usage

build custom image
```bash
./gradlew buildAarch64DockerImage
```

using a prebuilt SNAPSHOT
* log in to https://docker-auth.elastic.co/github_auth and copy paste to command line after logging in
* docker.elastic.co/kibana/kibana:8.x.y-SNAPSHOT
* docker.elastic.co/elasticsearch/elasticsearch:8.x.y-SNAPSHOT
* (optional)  docker tag elasticsearch:8.x.y-SNAPSHOT elasticsearch:8.x.y-SNAPSHOT-localbuild (to disambigute which snapshot)
* (optional) docker image rm docker.elastic.co/elasticsearch/elasticsearch:8.x.y-SNAPSHOT 

```bash
rm -rf fulfilldata && rm -rf querydata # to ensure clean slate 
docker-compose up
curl -u elastic:changeme http://localhost:9200  # query cluster
curl -u elastic:changeme http://localhost:19200 # fulfill cluster
```

### Establish X cluster connection

Get api key from fulfilling cluster:

```json
POST _security/cross_cluster/api_key
{
  "name": "my-cross-cluster-api-key",
  "access": {
    "search": [  
      {
        "names": ["logs*"]
      }
    ],
    "replication": [  
      {
        "names": ["archive*"]
      }
    ]
  },
  "metadata": {
    "foo": "bar"
  }
}
```

Example output:
```json
{
    "id": "53junYkB153pRTGmyMDF",
    "name": "my-cross-cluster-api-key",
    "api_key": "3qQrmTMcStSH6RpokDmE1Q",
    "encoded": "NTNqdW5Za0IxNTNwUlRHbXlNREY6M3FRcm1UTWNTdFNINlJwb2tEbUUxUQ=="
}
```

Add a document to the fulfilling cluster.
```json
POST logs-test/_doc/1
{
  "foo" :true
}
```

Stop the query cluster:

```bash
docker-compose stop es-query
```

Download the version of the keystore that matches the query cluster version to ensure a compatilbe keystort tool. For example, if the query cluster version is 8.13.4, then use that version to update the keystore. If you built locally, then use `./gradlew localDistro` get a release like build.

```
cd ~/workspace/elasticsearch/releases/
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.13.4-darwin-x86_64.tar.gz 
tar xfx elasticsearch-8.13.4-darwin-x86_64.tar.gz 
cd elasticsearch-8.13.4 
```
or a local build

```
cd ~/workspace/elasticsearch/8.14
./gradlew localDistro
cd ~/workspace/elasticsearch/8.14/build/distribution/local/elasticsearch-8.14.0-SNAPSHOT
```

Copy the keystore used by docker so we can update it.
```bash
cp  ~/workspace/es-docker-cross-cluster-with-kibana/query-config/elasticsearch.keystore ./config/elasticsearch.keystore 
bin/elasticsearch-keystore add cluster.remote.my_remote_cluster.credentials 
# add the encoded string, i.e. NTNqdW5Za0IxNTNwUlRHbXlNREY6M3FRcm1UTWNTdFNINlJwb2tEbUUxUQ==
# confirm
bin/elasticsearch-keystore show cluster.remote.my_remote_cluster.credentials
# replace
cp ./config/elasticsearch.keystore ~/workspace/es-docker-cross-cluster-with-kibana/query-config/elasticsearch.keystore
```

Start the query cluster back up
```bash
docker-compose up -d --no-deps es-query
```

Now you the query cluster should be connected the fulfilling cluster


GET _remote/info
```json
{
    "my_remote_cluster": {
        "connected": true,
        "mode": "proxy",
        "proxy_address": "es-fulfill:9443",
        "server_name": "",
        "num_proxy_sockets_connected": 18,
        "max_proxy_socket_connections": 18,
        "initial_connect_timeout": "30s",
        "skip_unavailable": false,
        "cluster_credentials": "::es_redacted::"
    }
}
```
and

`GET my_remote_cluster:logs-test/_search`

should return success (assuming logs-foo exists in the fulfilling cluster)

Optinally if you want to upgrade a cluster:

update the verions in docker-compose and restart the node. for example:

docker-compose up -d --no-deps k-fulfill
docker-compose up -d --no-deps es-fulfill

### With a custom user

On the querying cluster (as elastic user):

```json
POST /_security/role/remote1
{
  "remote_indices": [
    {
      "names": [ "logs-*" ],
      "privileges": [ "read" ],
      "clusters" : ["my_remote_cluster"]
    }
  ]
}

POST /_security/user/bart
{
  "password": "eatmyshorts!",
  "roles": ["remote1", "kibana_admin"]
}
```

On the query cluster (as bart user) :

```json
GET my_remote_cluster:logs-test/_search

```




