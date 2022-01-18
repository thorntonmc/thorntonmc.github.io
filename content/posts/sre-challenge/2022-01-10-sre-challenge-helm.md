---
title: A Reddit SRE Challenge - Building a Helm Chart
date: 2022-01-10T00:00:00+
draft: true
tags: ["sre-challenge"]
---


### Configuring a Helm Chart

Helm is considered to be the package manager for Kubernetes, and so in addition to terraforming our application, I created a chart for it as well. The first thing to do is to initialize your chart like so:

```bash
helm create sre-challenge
```


This creates our base chart scaffolding, but before we get started tweaking it - a critical component of helm involves leveraging existing charts, so the first thing to do is find a redis chart. 


```bash
michaelthornton@home:~/sre-challenge/deployments/helm/sre-challenge [main]
$ helm search repo redis
NAME                 	CHART VERSION	APP VERSION	DESCRIPTION
bitnami/redis        	15.7.1       	6.2.6      	Open source, advanced key-value store. It is of...
bitnami/redis-cluster	7.1.0        	6.2.6      	Open source, advanced key-value store. It is of...
```

We can then add that as a dependency to a chart by updating `Chart.yaml`


```yaml
dependencies:
  - name: redis
    version: 15.7.1
    repository: https://charts.bitnami.com/bitnami
```

After that, we can run `helm dependency update` to pull in our redis dependency

```bash 
michaelthornton@work:~/sre-challenge/deployments/helm/sre-challenge [main]
$ helm dependency update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Downloading redis from repo https://charts.bitnami.com/bitnami
Deleting outdated charts

michaelthornton@work:~/sre-challenge/deployments/helm/sre-challenge [main]
$ ls charts
redis-15.7.1.tgz
```
