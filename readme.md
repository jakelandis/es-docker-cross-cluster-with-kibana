### Usage

```bash
docker-compose up
curl -u elastic:changeme http://localhost:9200  # query cluster
curl -u elastic:changeme http://localhost:19200 # fulfill cluster
```

### Establish X cluster connection

Get api key from fulfilling cluster:

```json
POST {{es-fulfill}}/_security/cross_cluster/api_key
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

Stop the query cluster:

```bash
docker-compose stop es-query
```

Navigate to the rool of a recent released version of 8.x to update the docker keystore:

```bash
cp /home/jakelandis/workspace/es-docker-cross-cluster-with-kibana/elasticsearch.keystore.query ./config/elasticsearch.keystore
bin/elasticsearch-keystore add cluster.remote.my_remote_cluster.credentials 
# add the encoded string, i.e. NTNqdW5Za0IxNTNwUlRHbXlNREY6M3FRcm1UTWNTdFNINlJwb2tEbUUxUQ==
# confirm
bin/elasticsearch-keystore show cluster.remote.my_remote_cluster.credentials
# replace
cp ./config/elasticsearch.keystore /home/jakelandis/workspace/es-docker-cross-cluster-with-kibana/elasticsearch.keystore.query 
```

Start the query cluster back up
```bash
docker-compose up -d --no-deps es-query
```

Now you the query cluster should be connected the fulfilling cluster


GET {{es-query}}/_remote/info
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
GET {{es-query}}/my_remote_cluster:logs-foo/_search

should return success (assuming logs-foo exists in the fulfilling cluster)


