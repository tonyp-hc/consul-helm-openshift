# Consul traffic splitting example
This example configures two services: Web and API
- Web is configured with an upstream proxy to the API service.
- The API service has two versions: v1 and v2

Our steps:
1. deploy services
2. configure consul service-defaults, service-resolver, service-splitter 
3. weight all traffic to API service v1
4. weight traffic evenly between v1 and v2
5. weight all traffic to API service v2

```bash
# working out of consul-helm-openshift/examples/
$ cd 2-splitter/
$ oc apply -f api.yml
$ oc apply -f web.yml
``` 

Wait for deployments to complete:
```bash
$ oc get pods --selector="app in (web,api)"
NAME                                 READY   STATUS    RESTARTS   AGE
api-deployment-v1-fbbb79f5-h84ph     3/3     Running   0          22m
api-deployment-v2-564c96c94c-tc6ft   3/3     Running   0          45m
web-deployment-667ff8b4fb-7hv9f      3/3     Running   0          44m

$ oc get deployments --selector="app in (web,api)"
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
api-deployment-v1   1/1     1            1           45m
api-deployment-v2   1/1     1            1           45m
web-deployment      1/1     1            1           44m
```

To view the service configuration in consul, you can query the `catalog` API:
```bash
$ oc exec -it <CONSUL_CLIENT_POD> -- curl http://127.0.0.1:8500/v1/catalog/service/api | jq
[
  {
[ . . . snip . . . ]
    "ServiceName": "api",
    "ServiceTags": [],
    "ServiceAddress": "##.##.##.##"
[ . . . snip . . . ]
```

Open a shell session in one of the consul client pods
```bash
$ oc exec -it <CONSUL_CLIENT_POD> -- /bin/sh
```

You can confirm that you can see the api and web services by checking the catalog:
```bash
$ consul catalog services
api
api-sidecar-proxy
consul
web
web-sidecar-proxy
```

And that the web service is accessing the API backend by hitting its service port (9090 by default):
```bash
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v1",
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v2",
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v2",
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v1",
```

Now time to create some service configuration. Head to `/tmp`
```bash
$ cd tmp
```

We will now create service-defaults which supply default global variables for a service, like its protocol. More information is in our [documentation](https://www.consul.io/docs/agent/config-entries/service-defaults.html).
```bash
$ cat << EOF > api-service-defaults.json
{
  "kind": "service-defaults",
  "name": "api",
  "protocol": "http"
}
EOF
```

We will also create a service-resolver which defines how Consul's service discovery selects instances for a given service name. More information is in our [documentation](https://www.consul.io/docs/agent/config-entries/service-resolver.html).
```bash
$ cat << EOF > api-service-resolver.json
{
  "kind": "service-resolver",
  "name": "api",

  "subsets": {
    "v1": {
      "filter": "Service.Meta.version == v1"
    },
    "v2": {
      "filter": "Service.Meta.version == v2"
    }
  }
}
EOF
```

Apply both configs:
```
$ consul config write api-service-defaults.json
Config entry written: service-defaults/api

$ consul config write api-service-resolver.json
Config entry written: service-resolver/api
```

You can verify that they have been applied:
```bash
$ consul config read -kind service-defaults -name api
{
    "Kind": "service-defaults",
    "Name": "api",
    "Namespace": "default",
    "Protocol": "http",
    "MeshGateway": {},
    "Expose": {},
    "CreateIndex": 369521,
    "ModifyIndex": 369521
}

$ consul config read -kind service-resolver -name api
{
    "Kind": "service-resolver",
    "Name": "api",
    "Namespace": "default",
    "Subsets": {
        "v1": {
            "Filter": "Service.Meta.version == v1"
        },
        "v2": {
            "Filter": "Service.Meta.version == v2"
        }
    },
    "CreateIndex": 369550,
    "ModifyIndex": 369550
}
```

Now we will apply our first service-splitter. This will weight all traffic to v1. More information about service-spliters can be found in our [documentation](https://www.consul.io/docs/agent/config-entries/service-splitter). It is also possible to configure [multiple splits at once](https://www.consul.io/docs/connect/l7-traffic-management.html#splitting).
```bash
$ cat << EOF > api-service-splitter-a_100_0.json
{
  "kind": "service-splitter",
  "name": "api",
  "splits": [
    {
      "weight": 100,
      "service_subset": "v1"
    },
    {
      "weight": 0,
      "service_subset": "v2"
    }
  ]
}
EOF
```

You can verify this has been applied as well:
```bash
$ consul config read -kind service-splitter -name api
{
    "Kind": "service-splitter",
    "Name": "api",
    "Namespace": "default",
    "Splits": [
        {
            "Weight": 100,
            "ServiceSubset": "v1"
        },
        {
            "Weight": 0,
            "ServiceSubset": "v2"
        }
    ],
    "CreateIndex": 367658,
    "ModifyIndex": 367658
}
```

Now, if you hit the web service, the backend API will only use v1:
```bash
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v1",
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v1",
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v1",
```


Next, we will split traffic evenly.
```bash
$ cat << EOF > api-service-splitter-b_50_50.json
{
  "kind": "service-splitter",
  "name": "api",
  "splits": [
    {
      "weight": 50,
      "service_subset": "v1"
    },
    {
      "weight": 50,
      "service_subset": "v2"
    }
  ]
}
EOF

$ consul config write api-service-splitter-b_50_50.json
```

After which traffic will be distributed once more:
```bash
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v1",
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v1",
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v2",
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v1",
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v2",
```

Finally, we will flip all traffic to v2
```bash
$ cat << EOF > api-service-splitter-c_0_100.json
{
  "kind": "service-splitter",
  "name": "api",
  "splits": [
    {
      "weight": 0,
      "service_subset": "v1"
    },
    {
      "weight": 100,
      "service_subset": "v2"
    }
  ]
}
EOF

$ consul config write api-service-splitter-c_0_100.json
```

And traffic will only hit v2
```bash
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v2",
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v2",
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v2",
$ curl -s web:9090 | grep body
  "body": "Hello World",
      "body": "Response from API v2",
```
