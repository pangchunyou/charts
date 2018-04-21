## Appliance mode

This chart can be used to launch Redis in a black box appliance mode that you can think of like a managed service. To run as an appliance, change the service type for the master and slave LBs to enable local access from within the K8S cluster.

To launch in VPC-only appliance mode, set appliance.serviceType to "LoadBalancer". If using appliance mode in Google Cloud, set appliance.annotations to:
`cloud.google.com/load-balancer-type:Internal`

```bash
$ helm install \
  --set="servers.annotations.cloud\.google\.com/load-balancer-type=Internal,servers.serviceType=LoadBalancer" \
    stable/redis-ha
```

## Configuration

The following table lists the configurable parameters of the Redis chart and their default values.

| Parameter                        | Description                                                                                                                  | Default                                                   |
| -------------------------------- | -----------------------------------------------------                                                                        | --------------------------------------------------------- |
| `redis_image`                    | Redis image                                                                                                                  | `quay.io/smile/redis:4.0.6r2`                             |
| `resources.master`               | CPU/Memory for master nodes resource requests/limits                                                                         | Memory: `200Mi`, CPU: `100m`                              |
| `resources.slave`                | CPU/Memory for slave nodes  resource requests/limits                                                                         | Memory: `200Mi`, CPU: `100m`                              |
| `resources.sentinel`             | CPU/Memory for sentinel node resource requests/limits                                                                        | Memory: `200Mi`, CPU: `100m`                              |
| `replicas.servers`               | Number of redis master/slave pods                                                                                            | 3                                                         |
| `replicas.sentinels`             | Number of sentinel pods                                                                                                      | 3                                                         |
| `nodeSelector`                   | Node labels for pod assignment                                                                                               | {}                                                        |
| `tolerations`                    | Toleration labels for pod assignment                                                                                         | []                                                        |
| `servers.serviceType`            | Set to "LoadBalancer" to enable access from the VPC                                                                          | ClusterIP                                                 |
| `servers.annotations`            | See Appliance mode                                                                                                           | ``                                                        |
| `rbac.create`                    |  whether RBAC resources should be created                                                                                    | true                                                      |
| `serviceAccount.create`          | whether a new service account name that the agent will use should be created.                                                | true                                                      |
| `serviceAccount.name`            | service account to be used.  If not set and serviceAccount.create is `true` a name is generated using the fullname template. | ``                                                        |


Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example,

```bash
$ helm install \
  --set redis_image=quay.io/smile/redis:4.0.6r2 \
    stable/redis-ha
```

The above command sets the Redis server within  `default` namespace.

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. For example,

```bash
$ helm install -f values.yaml stable/redis-ha
```

> **Tip**: You can use the default [values.yaml](values.yaml)

## Internals
The customized Redis server image determines whether the pod that executes it will be a Redis Sentinel,
Master, or Slave and launches the appropriate service. This Helm chart signals Sentinel status with
environment variables. If not set, the newly launched pod will query K8S for an active master. If none
exists, it uses a deterministic means of sensing whether it should launch as master then writes "master"
or "slave" to the label called redis-role as appropriate. It's this label that determines which LB a pod
can be seen through.

The redis-role=master pod is the key for the cluster to get started. Sentinels will wait for it to appear
in the LB before they finish launching. All other pods wait for the Sentinels to ID the master. Running
Pods also set the labels podIP and runID. runID is the first few characters of the unique run_id value
generated by each Redis server.

During normal operation, there should be only one redis-role=master pod. If it fails, the Sentinels
will nominate a new master and change all the redis-role values appropriately.

To see the pod roles, run the following:

```bash
$ kubectl get pods -L redis-role
```
