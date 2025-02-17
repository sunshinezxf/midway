# Prometheus

Prometheus (Prometheus) is a monitoring system originally built on the SoundCloud.  Since 2012, it has become a community open source project with a very active developer and user community. In order to emphasize open source and independent maintenance, Prometheus joined the Cloud Native Cloud Computing Foundation (CNCF) in 2016, becoming the second hosting project after Kubernetes.

Grafana is an open source measurement analysis and visualization suite. A front-end tool developed purely Javascript displays custom reports and charts by accessing libraries (such as InfluxDB). Grafana supports many different data sources. Each data source has a specific query editor, and the features and functions customized by the editor are the specific data sources that are exposed. The Prometheus is exactly one of its supported data sources.

This article introduces how Midway accesses Grafana + Prometheus.

The access effect is as follows:

![](https://cdn.nlark.com/yuque/0/2021/png/187105/1617259935548-a2df4339-3229-4391-bd3d-4ba8e6979d4d.png)

## Installation dependency

First install the indicator monitoring component provided by Midway:

```bash
$ npm install @midwayjs/prometheus@3 --save
```

Or reinstall the following dependencies in `package.json`.

```json
{
  "dependencies": {
    "@midwayjs/prometheus": "^3.0.0",
    // ...
  },
  "devDependencies": {
    // ...
  }
}
```



## Introducing components

In `configuration.ts`, introduce this component:

```typescript
// src/configuration.ts
import { Configuration } from '@midwayjs/decorator';
Import * as prometheus from '@midwayjs/prometheus'; // Import module
import { join } from 'path';

@Configuration({
  imports: [
    // ...
    prometheus
  ],
  importConfigs: [join(__dirname, 'config')]
})
export class MainConfiguration {}
```

When we started our application, there was an extra `${host }:${ port}/metrics` when we visited it.

:::info
Prometheus the monitoring data is obtained based on HTTP, please load any framework of web/koa/express and start in multi-framework mode.
:::

Access interface, return as follows, the contents are the current indicators.

![](https://cdn.nlark.com/yuque/0/2021/png/187105/1617260048533-4f725824-9471-40c9-be8b-6dcbf27d9cca.png)

## Other configurations

The indicator component also provides relevant configurations to facilitate developers to configure.

You can modify the configuration of the prometheus in `config.default.ts`.

```typescript
// src/config/config.default
export default {
  // ...
  prometheus: {
    labels: {
      APP_NAME: 'demo_project',
    },
  },
}
```

For more configurations, we can look at the definitions for configuration.

Through configuration, for example, we can classify which nodes are the same application, because when we deploy, the node program is distributed. For example, we added APP_NAME above to distinguish different applications, so that we can distinguish different applications in the monitoring index.

## Data acquisition

The components we introduced earlier in Midway are mainly to add indicator modules to Node. Next, we need Prometheus to collect our index data.

If the developer's department already has Prometheus + grafana, it only needs to report the application's indicator address to PE or through the interface. Here we assume that everyone has no Prometheus + grafana, and then follow the following description.

## Build Prometheus

Here we use docker-compose to build Prometheus. The docker-compose.yml file is as follows:

```yaml
version: '2.2'
services:
  tapi:
    logging:
      driver: 'json-file'
      options:
        max-size: '50m'
    image: prom/prometheus
    restart: always
    volumes:
      - ./prometheus_data:/prometheus_data:rw
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./targets.json:/etc/prometheus/targets.json
    command:
      - '--storage.tsdb.path=/prometheus_data'
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention=10d'
      - '--web.enable-lifecycle'
    ports:
      - '9090:9090'
```

The `prometheus.yml` file is as follows:

```yaml
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
scrape_configs:
  - job_name: 'node'
    file_sd_configs:
      - refresh_interval: 1m
        files:
          - '/etc/prometheus/targets.json'
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

Then, the collected `targets.json` is as follows: `${ip}` in the following file is replaced with the ip address of the server where the Node.js application is located.

```json
[
  {
    "targets": ["${ip}:7001"]
    "labels": {
      "env": "prod ",
      "job": "api"
    }
  }
]
```



Then we start the `docker-compose.yml` file,

```bash
$ docker-compose up
```

At this point, Prometheus will already pull the indicator data of our Node application.

What to do if you want to update target:
After modifying this targets.json file, it is hot loaded by the prometheus reload method.
The method is as follows:

```bash
$ curl-X POST http://${prometheus ip}:9090/-/reload
```

Then we can check the prometheus page or confirm whether it takes effect. The interface address is:

```text
http://${prometheus ip}:9090/classic/targets
```

The next step is how to show the collected data.

## Data presentation

We can use Grafana to show our data.

Here we simply use Docker to build a Grafana:

```bash
$ docker run -d --name=grafana -p 3000:3000 grafana/grafana
```

Then we visit 127.0.0.1:3000, and the default account password is admin:admin.
Then visit the following effect:

<img src="https://cdn.nlark.com/yuque/0/2021/png/187105/1617260561047-c2643a69-6258-491b-937d-9bfc4558252f.png#height=346&id=yNdWZ&margin=%5Bobject%20Object%5D&name=image.png&originHeight=692&originWidth=1496&originalType=binary&ratio=1&size=551202&status=done&style=none&width=748" width="748" />

Then we let Grafana access our Prometheus data sources:

<img src="https://cdn.nlark.com/yuque/0/2021/png/187105/1617260581029-1e2e06a8-3054-4ad8-96b5-d50ab9bb1612.png#height=286&id=atAvT&margin=%5Bobject%20Object%5D&name=image.png&originHeight=572&originWidth=1490&originalType=binary&ratio=1&size=169944&status=done&style=none&width=745" width="745" />

Then we click Grafana to add the chart:

<img src="https://cdn.nlark.com/yuque/0/2021/png/187105/1620725466020-28793a78-c03b-48fa-bf16-0c9c8ecc1a94.png#clientId=u070308fc-4e5d-4&from=paste&height=741&id=uce167575&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1482&originWidth=2626&originalType=binary&ratio=1&size=310590&status=done&style=none&taskId=uedd61eb7-8e61-488f-963f-f70adb9a651&width=1313" width="1313" />

Select 14403 ID here, then click load, then click Next, and then click import to see the effect we have just accessed.

<img src="https://cdn.nlark.com/yuque/0/2021/png/187105/1620725497338-a32a8982-d51f-4e74-b511-dc10a7c66d80.png#clientId=u070308fc-4e5d-4&from=paste&height=996&id=uba6ac1f0&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1992&originWidth=3836&originalType=binary&ratio=1&size=1951604&status=done&style=none&taskId=ua7c2fc08-0633-4614-9af0-5bf2da800ef&width=1918" width="1918" />

<img src="https://cdn.nlark.com/yuque/0/2021/png/187105/1620725514630-4f654f10-ef3a-41f7-b403-02832d3ef7d8.png#clientId=u070308fc-4e5d-4&from=paste&height=998&id=u27a3ae30&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1996&originWidth=3830&originalType=binary&ratio=1&size=2201307&status=done&style=none&taskId=ucee30610-4c1f-4fa8-82fd-a952d5aa9e1&width=1915" width="1915" />

In this way, developers can operate and operate their own Node programs, for example, whether an NPM package has recently been introduced to cause any memory leakage, and whether there has been an application restart recently.

Of course, it can also support other custom operations.


## Socket-io scene

Usage:

```bash
$ npm install @midwayjs/prometheus-socket-io@3 --save
```

Usage:

```typescript
import { Configuration } from '@midwayjs/decorator';
import { join } from 'path';
import * as prometheus from '@midwayjs/prometheus';
import * as prometheusSocketIo from '@midwayjs/prometheus-socket-io';

@Configuration({
  imports: [prometheus, prometheusSocketIo],
  importConfigs: [join(__dirname, './config')]
})
export class ContainerLifeCycle {}
```

Then you can see the socket-io data on the/metrics side.

<img src="https://cdn.nlark.com/yuque/0/2021/png/187105/1631090438583-d925c13c-371a-4037-9f53-edaa34580aab.png#clientId=u24adff00-2245-4&from=paste&height=459&id=u2862ab6b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=918&originWidth=1470&originalType=binary&ratio=1&size=481525&status=done&style=none&taskId=ua4ce06b2-a75d-4e4a-8bd9-c94496dca33&width=735" width="735" />

A total of 8 new indicators have been added.
The Grafana template ID will be provided for everyone to use later.


## Function introduction

- [x] Sort by appName
- [x] View the qps situation of different paths
- [x] View the distribution of different statuses
- [x] Query the rt situation of different paths
- [x] CPU usage of the process
- [x] memory usage of the process
- [x] stack situation
- [x] Event Loop
- [ ] Wait