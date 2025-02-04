# Akeyless Zero Trust Web Access
 

## Introduction
This chart bootstraps a Akeyless-Zero-Trust-Web-Access deployment on a Kubernetes cluster using the Helm package manager.
This chart has been tested to work with [NGINX Ingress](https://kubernetes.github.io/ingress-nginx/) and [cert-manager](https://cert-manager.io/).


## Preparation

### Network
When using Embedded browser session behind load balancer such as ELB, the session can be closed due to **idle connection timeout**, so its advise to increase it
to a reasonable high value, or event unlimited.

e.g when running on AWS with ELB:
https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/config-idle-timeout.html?icmpid=docs_elb_console

## Storage
To be able to download files to your local machine, the helm chart requires a storage class with ReadWriteMany access modes.  
Since a storage class is more environment specific, you will need to provide one before proceeding.
In addition, please provide 1 PersistentVolumes and reference those PVs under `persistence` section in the `values.yaml` file

e.g when running on AWS with EKS:
https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html

For security reason, please limit the PersistentVolumes mount permissions to `0650`, example: 
```mountOptions:
   - dir_mode=0650
   - file_mode=0650
```

### Prerequisites

#### Horizonal Auto-Scaling
Horizontal auto-scaling is based on the HorizonalPodAutoscaler object.  
For it to work properly, Kubernetes metrics server must be installed in the cluster - https://github.com/kubernetes-sigs/metrics-server

To Support auto-scaling for `webWorker` pods based on **busy workers percentage**, please do the following:

Install `Prometheus adapter` - https://github.com/kubernetes-sigs/prometheus-adapter

```bash
helm install --name my-release-name stable/prometheus-adapter
```

Configure the adapter with akeyless custom rule

```bash
prometheus-adapter:
  prometheus:
    url: <prometheus-url>
    port: <prometheus-port>

  rules:
      custom:
        - seriesQuery: 'zero_trust_web_access_workers_stats_busy_workers{namespace!="",pod!="",service!=""}'
          resources:
            overrides:
              namespace:
                resource: namespace
              pod:
                resource: pod
              service:
                resource: service
          name:
            matches: "^(.*)"
            as: "workers_utilization"
          metricsQuery: round(zero_trust_web_access_workers_stats_busy_workers{<<.LabelMatchers>>})
     
```    

## Get Repo Info

```bash
$ helm repo add akeyless https://akeylesslabs.github.io/helm-charts
$ helm repo update
```
See [helm repo](https://helm.sh/docs/helm/helm_repo/) for command documentation.

## Installing the Chart

The `values.yaml` file holds default values, replace the values with the ones from your environment where needed.  

To install the chart run:
```bash
helm install RELEASE_NAME akeyless/akeyless-zero-trust-web-access
``` 

## Parameters

The following table lists the configurable parameters of the Zero Trust Web Access chart and their default values.

### Deployment parameters

| Parameter                                 | Description                                                                                                          | Default                                                      |
|-------------------------------------------|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| `image.dockerRepositoryCreds`             | Akeyless docker repository credentials - required                                                                    | `nil`                                                        |
| `dispatcher.image.repository`             | Zero Trust Web Access Dispatcher image name                                                                          | `akeyless/zero-trust-web-dispatcher`                         |
| `dispatcher.image.tag`                    | Zero Trust Web Access Dispatcher image tag                                                                           | `latest`                                                     |      
| `dispatcher.image.pullPolicy`             | Zero Trust Web Access Dispatcher image pull policy                                                                   | `Always`                                                     |  
| `dispatcher.containerName`                | Zero Trust Web Access Dispatcher container name                                                                      | `web-dispatcher`                                             |    
| `dispatcher.replicaCount`                 | Number of Zero Trust Web Access Dispatcher nodes                                                                     | `1`                                                          |
| `dispatcher.livenessProbe`                | Liveness probe configuration for Zero Trust Web Access Dispatcher                                                    | Check `values.yaml` file                                     |                   
| `dispatcher.readinessProbe`               | Readiness probe configuration for Zero Trust Web Access Dispatcher                                                   | Check `values.yaml` file                                     |         
| `dispatcher.resources.limits`             | The resources limits for Zero Trust Web Access Dispatcher containers (If HPA is enabled these must be set)           | `{}`                                                         |
| `dispatcher.resources.requests`           | The requested resources for Zero Trust Web Access Dispatcher containers (If HPA is enabled these must be set)        | `{}`                                                         |
| `webWorker.image.repository`              | Zero Trust Web Access Web Worker image name                                                                          | `akeyless/zero-trust-web-worker`                             |
| `webWorker.image.tag`                     | Zero Trust Web Access Web Worker image tag                                                                           | `latest`                                                     |      
| `webWorker.image.pullPolicy`              | Zero Trust Web Access Web Worker image pull policy                                                                   | `Always`                                                     |
| `webWorker.containerName`                 | Zero Trust Web Access Web Worker container name                                                                      | `web-worker`                                                 |
| `webWorker.replicaCount`                  | Number of Zero Trust Web Access Web Worker nodes                                                                     | `5`                                                          |
| `webWorker.livenessProbe`                 | Liveness probe configuration for Zero Trust Web Access Web Worker                                                    | Check `values.yaml` file                                     |                   
| `webWorker.readinessProbe`                | Readiness probe configuration for Zero Trust Web Access Web Worker                                                   | Check `values.yaml` file                                     |         
| `webWorker.resources.limits`              | The resources limits for Zero Trust Web Access Web Worker containers (If HPA is enabled these must be set)           | `{}`                                                         |
| `webWorker.resources.requests`            | The requested resources for Zero Trust Web Access Web Worker containers (If HPA is enabled these must be set)        | `{}`                                                         |

### Exposure parameters

| Parameter                                 | Description                                                                                                          | Default                                                      |
|-------------------------------------------|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| `dispatcher.service.type`                 | Kubernetes service type                                                                                              | `LoadBalancer`                                               |
| `dispatcher.service.port`                 | Dispatcher service port                                                                                              | `9000`                                                       |
| `dispatcher.service.annotations`          | Dispatcher service extra annotations                                                                                 | `{}`                                                         |
| `webProxy.service.port`                   | Web Proxy service port                                                                                               | `19414`                                                      |
| `webWorker.service.port`                  | Web Worker service port                                                                                              | `5800`                                                       |
| `webWorker.service.annotations`           | Web Worker service extra annotations                                                                                 | `{}`                                                         |
| `dispatcher.ingress.enabled`              | Enable ingress resource                                                                                              | `false`                                                      |
| `dispatcher.ingress.path`                 | Path for the default host                                                                                            | `/`                                                          |
| `dispatcher.ingress.certManager`          | Add annotations for cert-manager                                                                                     | `false`                                                      |
| `dispatcher.ingress.hostname`             | Default host for the ingress resource                                                                                | `aztwa.local`                                                |
| `dispatcher.ingress.annotations`          | Ingress annotations                                                                                                  | `[]`                                                         |
| `dispatcher.ingress.tls`                  | Enable TLS configuration for the hostname defined at `ingress.hostname` parameter                                    | `false`                                                      |
| `dispatcher.ingress.existingSecret`       | Existing secret for the Ingress TLS certificate                                                                      | `nil`                                                        |  
| `httpProxySettings.http_proxy`            | Standard linux HTTP Proxy, should contain the URLs of the proxies for HTTP                                           | `nil`                                                        |  
| `httpProxySettings.https_proxy`           | Standard linux HTTP Proxy, should contain the URLs of the proxies for HTTPS                                          | `nil`                                                        |  
| `httpProxySettings.no_proxy`              | Standard linux HTTP Proxy, should contain a comma-separated list of domain extensions proxy should not be used for   | `nil`                                                        |  

### HPA parameters

| Parameter                                 | Description                                            | Default |
|-------------------------------------------|--------------------------------------------------------|---------|
| `HPA.enabled`                             | Enable Zero Trust Web Access Horizontal Pod Autoscaler | `false` |
| `HPA.dispatcher.minReplicas`              | Dispatcher Minimum desired number of replicas          | `1`     |
| `HPA.dispatcher.maxReplicas`              | Dispatcher Minimum desired number of replicas          | `14`    |
| `HPA.dispatcher.cpuAvgUtil`               | Dispatcher CPU average utilization                     | `50`    |
| `HPA.dispatcher.memAvgUtil`               | Dispatcher Memory average utilization                  | `50`    |
| `HPA.webWorker.busyWorkersPercentage`     | Busy Workers utilization percentage                    | `50`    |

### Zero Trust Web Access configuration parameters

| Parameter                                             | Description                                                                                    | Default                    |
|-------------------------------------------------------|------------------------------------------------------------------------------------------------|----------------------------|
| `dispatcher.config.privilegedAccess.accessID`         | Access ID with "read" capability for privileged access.                                        | `nil`                      |
| `dispatcher.config.privilegedAccess.accessKey`        | Access Key of the provided access ID. (not required on cloud identity)                         | `nil`                      |
| `dispatcher.config.privilegedAccess.allowedAccessIDs` | Access will be permitted only to these access IDs. By default, any access ID is accepted.      | `[]`                       |
| `config.listOnlyCredentials.samlAccessID`             | Non-privileged SAML credentials with "list" only access.                                       | `nil`                      |
| `dispatcher.config.apiGatewayURL`                     | API Gateway URL to use to fetch the secrets.                                                   | `https://rest.akeyless.io` |
| `dispatcher.config.disableSecureCookie`               | Use browser secure cookie only (HTTPS)                                                         | `true`                     |
| `webWorker.config.displayWidth`                       | Web worker display Width (in pixels) of the application's window.                              | `2560`                     |
| `webWorker.config.displayHeight`                      | Web worker display Height (in pixels) of the application's window.                             | `1200`                     |
| `dispatcher.config.allowedBastionUrls`                | List of URLs that will be considered valid for redirection from the Portal back to the bastion | `[]`                       |
