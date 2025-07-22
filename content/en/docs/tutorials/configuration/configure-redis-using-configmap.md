---
reviewers:
- eparis
- pmorie
title: Configuring Redis using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

This tutorial describes how to configure a [Redis](https://redis.io/) cache using a ConfigMap and builds upon the [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/) task.

## {{% heading "objectives" %}}

* Create a ConfigMap with Redis configuration values.
* Create a Redis Pod that mounts and uses the ConfigMap.
* Update the configuration values.
* Verify that the configuration update was correctly applied.

## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

* Read and understand the [ConfigMap](/docs/concepts/configuration/configmap/) API object.
* Read and understand [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Ensure that `kubectl` is version 1.14 or above.

<!-- lessoncontent -->

## Create and apply a Redis cache using a ConfigMap

Follow the steps below to configure a Redis cache using data stored in a ConfigMap.

First [create a ConfigMap from a generator](/docs/tasks/configure-pod-container/configure-pod-configmap/#create-a-configmap-from-generator) with an empty configuration block:

```shell
cat <<EOF >./example-redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: ""
EOF
```

Apply the configuration in the ConfigMap, along with a Redis pod manifest:

```shell
kubectl apply -f example-redis-config.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

## Review the Redis pod manifest

Examine the contents of the Redis pod manifest and note the following details of the configuration:

* A [volume](/docs/concepts/storage/volumes/) named `config` is created by `spec.volumes[1]`.
* The `key` and `path` under `spec.volumes[1].configMap.items[0]` exposes the `redis-config` key from the 
  `example-redis-config` ConfigMap as a file named `redis.conf` on the `config` volume.
* The `config` volume is then mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.

This configuration exposes the data in `data.redis-config` from the `example-redis-config`
ConfigMap above as [`redis.conf`](https://redis.io/docs/latest/operate/oss_and_stack/management/config/) file inside the Pod.

{{% code_sample file="pods/config/redis-pod.yaml" %}}

## Verify the ConfigMap is correctly set up

Examine the created objects to make sure that the ConfigMap is correctly set up:

```shell
kubectl get pod/redis configmap/example-redis-config 
```

You should see the following output:

```
NAME        READY   STATUS    RESTARTS   AGE
pod/redis   1/1     Running   0          8s

NAME                             DATA   AGE
configmap/example-redis-config   1      14s
```

Ensure that the `redis-config` key is blank when you print a detailed description of the `example-redis-config` ConfigMap:

```shell
kubectl describe configmap/example-redis-config
```

The description should display an empty `redis-config` key:

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
```

## Updating configuration values

You can add Redis configuration values to the YAML file, and then check the configuration with [`redis-cli`](https://redis.io/docs/latest/develop/tools/cli/).

Add a `maxmemory` and `maxmemory-policy` value to the `example-redis-config` ConfigMap in order to [configure Redis as a cache](https://redis.io/docs/latest/operate/oss_and_stack/management/config/#configuring-redis-as-a-cache):

{{% code_sample file="pods/config/example-redis-config.yaml" %}}

Apply the updated ConfigMap:

```shell
kubectl apply -f example-redis-config.yaml
```

The Pod needs to be restarted to grab the updated values from any associated ConfigMaps. Delete and recreate the Pod:

```shell
kubectl delete pod redis
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

### Verify the ConfigMap updates with kubectl

Confirm that the ConfigMap was updated:

```shell
kubectl describe configmap/example-redis-config
```

You should see the configuration values we just added:

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
----
maxmemory 2mb
maxmemory-policy allkeys-lru
```

### Verify the configuration updates with redis-cli

Check the Redis Pod using `redis-cli` via `kubectl exec` to see if the configuration was applied:

```shell
kubectl exec -it redis -- redis-cli
```

Check `maxmemory`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It now reflects the updated value of 2097152:

```shell
1) "maxmemory"
2) "2097152"
```

Check `maxmemory-policy`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

It now reflects the desired value of `allkeys-lru`:

```shell
1) "maxmemory-policy"
2) "allkeys-lru"
```

## Cleanup your work

Clean up your work by deleting the created resources:

```shell
kubectl delete pod/redis configmap/example-redis-config
```

## {{% heading "whatsnext" %}}

* Follow an example of [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
* Learn how to [configure a Pod using a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Learn more about [Using ConfigMaps as files from a Pod](/docs/concepts/configuration/configmap/#using-configmaps-as-files-from-a-pod).
* Learn about [Configuration best practices](/docs/concepts/configuration/overview/).