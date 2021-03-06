# How to install if you have an existing Prometheus operator

<!-- TOC -->
 
- [Install Fluentd, Fluent-bit, and Falco](#install-fluentd-fluent-bit-and-falco) 
- [Overwrite Prometheus Remote Write Configuration](#overwrite-prometheus-remote-write-configuration) 
- [Merge Prometheus Remote Write Configuration](#merge-prometheus-remote-write-configuration)  

<!-- /TOC -->

## Install Fluentd, Fluent-bit, and Falco

Installation of collection in a cluster where a prometheus operator already exists requires that you modify Sumo Logic's [values.yaml](https://github.com/SumoLogic/sumologic-kubernetes-collection/blob/master/deploy/helm/sumologic/values.yaml) file. Run the following to download the `values.yaml` file

```bash
curl -LJO https://raw.githubusercontent.com/SumoLogic/sumologic-kubernetes-collection/release-v0.17/deploy/helm/sumologic/values.yaml
```

Edit the `values.yaml` file, setting `prometheus-operator.enabled = false`. This modification will instruct Helm to install all the needed collection components (FluentD, FluentBit, and Falco), but it will not install the Prometheus Operator. Run the following command to install collection on your cluster.

```bash
helm install sumologic/sumologic --name collection --namespace sumologic -f values.yaml --set sumologic.accessId=<SUMO_ACCESS_ID> --set sumologic.accessKey=<SUMO_ACCESS_KEY> --set sumologic.clusterName=<MY_CLUSTER_NAME> 
```

**NOTE**:
In case the prometheus-operator is installed in a different namespace as compared to where the Sumo Logic Solution is deployed, you would need to do the following two steps:

##### 1. Copy one of the configmaps that exposes the release name,  which is used in the remote write urls.

For example:\
If the Sumo Logic Solution is deployed in `<source-namespace>` and the existing prometheus-operator is in `<destination-namespace>`, run the below command:
```bash
kubectl get configmap sumologic-configmap --namespace=<source-namespace> --export -o yaml | kubectl apply --namespace=<destination-namespace> -f -
```
##### 2. Update Prometheus remote write URL's
Run the following commands to update the [remote write configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write) of the prometheus operator with the prometheus overrides we provide in our [prometheus-overrides.yaml](https://github.com/SumoLogic/sumologic-kubernetes-collection/blob/master/deploy/helm/prometheus-overrides.yaml).

Run the following command to download our prometheus-overrides.yaml file

```bash
curl -LJO https://raw.githubusercontent.com/SumoLogic/sumologic-kubernetes-collection/release-v0.17/deploy/helm/prometheus-overrides.yaml > prometheus-overrides.yaml
```

Next run

```bash
helm upgrade prometheus-operator stable/prometheus-operator -f prometheus-overrides.yaml --set prometheus-operator.prometheus-node-exporter.service.port=9200 --set prometheus-operator.prometheus-node-exporter.service.targetPort=9200
```

## Merge Prometheus Remote Write Configuration

If you have customized your Prometheus remote write configuration, follow these steps to merge the configurations. 

Helm supports providing multiple configuration files, and priority will be given to the last (right-most) file specified. You can obtain your current prometheus configuration by running

```bash
helm get values prometheus-operator > current-values.yaml
```

Any section of `current-values.yaml` that conflicts with sections of our `prometheus-overrides.yaml` will have to be removed from the `prometheus-overrides.yaml` file and appended to `current-values.yaml` in relevant sections. For any config that doesn’t conflict, you can leave them in `prometheus-overrides.yaml`. Then run

```bash
helm upgrade prometheus-operator stable/prometheus-operator -f current-values.yaml -f prometheus-overrides.yaml
```

__NOTE__ To filter or add custom metrics to Prometheus, [please refer to this document](additional_prometheus_configuration.md)
