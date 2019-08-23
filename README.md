# yc-k8s-elastic
## Pre requisits
As using any elastic software you're presumably read and agree with following [license](https://github.com/elastic/cloud-on-k8s/blob/master/LICENSE.txt)
## Description
Followed yaml manifests bootstrap [Elasticsearch & Kibana on Kubernetes with Elastic Cloud on Kubernetes](https://www.elastic.co/elasticsearch-kubernetes), or Elastic and Kibana operator on top of K8s in short.

This repository was used as a starting point: [cloud-on-k8s](https://github.com/elastic/cloud-on-k8s)

With following specific details:

* Leveraging SSD class storage of Yandex Managed Kubernetes service for persisting Elasticsearch data, sized for 300Gb each.
* Using 5 nodes with master+data+ingest roles
* Configured resource limits and request with tuned java options to utilize 6GB memory. Leverage this [article](https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html) for configure suitable resource usage for your cluster.
```yaml
- name: elasticsearch
  resources:
    limits:
      memory: 8Gi
      cpu: 4
    request:
      memory: 10Gi
      cpu: 8
  env:
  - name: ES_JAVA_OPTS
    value: "-Xms6g -Xmx6g"
```
For more detail on configuring operator, you may refer to this [docs](https://github.com/elastic/cloud-on-k8s/tree/master/docs)

## Installation

```
kubectl apply -f 01_elasticsearch_operator.yaml
kubectl apply -f 02_elasticsearch_resource.yaml
kubectl apply -f 04_elasticsearch_operator_kibana.yaml
```
For allowing logs from Kubernetes running application to be collected and stored in Elasticsearch we need to deploy filebeat/fluentd/logstash.

Due to mechanics how Elastic operator works, Elasticsearch is exposed via usual `9200` port, but leveraging tls encryption and also have random generated password. In example below we leverage Filebeat, but you could use any other solutions for your convinience.  In order to properly configure Filebeat, we need to get password and *CA* certificate.

```bash
kubectl get secret es-log-es-http-certs-public -o json --namespace=elastic-system | jq '.metadata.namespace = "kube-system"' | jq 'del(.metadata.creationTimestamp,.metadata.ownerReferences,.metadata.resourceVersion,.metadata.selfLink,.metadata.uid)' | kubectl --namespace=kube-system apply -f -
```

```bash
elasticpassword="$(kubectl --namespace=elastic-system get secret es-log-es-elastic-user -o go-template='{{.data.elastic | base64decode }}')" && sed -i.bak "s/passwwwwordanchor/${elasticpassword}/g" 03_filebeat.yaml

kubectl apply -f 03_filebeat.yaml
```




## Performance
Rally was used for meter performance of 5 nodes cluster with  NVME class disks for data with same resource limitation and same Java settings at `8 vCPU 100% with 24 GB RAM and system disk on network HDD`, enabled tls, and [esrally](https://github.com/elastic/rally) run from very same Kubernetes cluster.

```bash
esrally --track=http_logs \
--target-hosts=es-log-es-http.elastic-system.svc.cluster.local:9200 \
--pipeline=benchmark-only \
--client-options="use_ssl:true,verify_certs:true,ca_certs:'/tmp/ca.pem',basic_auth_user:'elastic',basic_auth_password:changeme"
```

```
------------------------------------------------------
    _______             __   _____
   / ____(_)___  ____ _/ /  / ___/_________  ________
  / /_  / / __ \/ __ `/ /   \__ \/ ___/ __ \/ ___/ _ \
 / __/ / / / / / /_/ / /   ___/ / /__/ /_/ / /  /  __/
/_/   /_/_/ /_/\__,_/_/   /____/\___/\____/_/   \___/
------------------------------------------------------

|   Lap |                                                         Metric |                Task |       Value |    Unit |
|------:|---------------------------------------------------------------:|--------------------:|------------:|--------:|
|   All |                     Cumulative indexing time of primary shards |                     |     296.566 |     min |
|   All |             Min cumulative indexing time across primary shards |                     |  0.00223333 |     min |
|   All |          Median cumulative indexing time across primary shards |                     |     2.42022 |     min |
|   All |             Max cumulative indexing time across primary shards |                     |     47.2659 |     min |
|   All |            Cumulative indexing throttle time of primary shards |                     |           0 |     min |
|   All |    Min cumulative indexing throttle time across primary shards |                     |           0 |     min |
|   All | Median cumulative indexing throttle time across primary shards |                     |           0 |     min |
|   All |    Max cumulative indexing throttle time across primary shards |                     |           0 |     min |
|   All |                        Cumulative merge time of primary shards |                     |     57.5216 |     min |
|   All |                       Cumulative merge count of primary shards |                     |        9504 |         |
|   All |                Min cumulative merge time across primary shards |                     |           0 |     min |
|   All |             Median cumulative merge time across primary shards |                     |           0 |     min |
|   All |                Max cumulative merge time across primary shards |                     |     32.2821 |     min |
|   All |               Cumulative merge throttle time of primary shards |                     |     15.8489 |     min |
|   All |       Min cumulative merge throttle time across primary shards |                     |           0 |     min |
|   All |    Median cumulative merge throttle time across primary shards |                     |           0 |     min |
|   All |       Max cumulative merge throttle time across primary shards |                     |     8.92353 |     min |
|   All |                      Cumulative refresh time of primary shards |                     |      37.856 |     min |
|   All |                     Cumulative refresh count of primary shards |                     |       82565 |         |
|   All |              Min cumulative refresh time across primary shards |                     |   0.0160333 |     min |
|   All |           Median cumulative refresh time across primary shards |                     |    0.169033 |     min |
|   All |              Max cumulative refresh time across primary shards |                     |       30.75 |     min |
|   All |                        Cumulative flush time of primary shards |                     |     10.6002 |     min |
|   All |                       Cumulative flush count of primary shards |                     |         155 |         |
|   All |                Min cumulative flush time across primary shards |                     | 0.000566667 |     min |
|   All |             Median cumulative flush time across primary shards |                     |   0.0117667 |     min |
|   All |                Max cumulative flush time across primary shards |                     |     1.99122 |     min |
|   All |                                             Total Young Gen GC |                     |     307.884 |       s |
|   All |                                               Total Old Gen GC |                     |        2.29 |       s |
|   All |                                                     Store size |                     |     26.2452 |      GB |
|   All |                                                  Translog size |                     |     15.6639 |      GB |
|   All |                                         Heap used for segments |                     |     85.3598 |      MB |
|   All |                                       Heap used for doc values |                     |   0.0841103 |      MB |
|   All |                                            Heap used for terms |                     |     70.4624 |      MB |
|   All |                                            Heap used for norms |                     |   0.0231323 |      MB |
|   All |                                           Heap used for points |                     |     5.93185 |      MB |
|   All |                                    Heap used for stored fields |                     |     8.85834 |      MB |
|   All |                                                  Segment count |                     |         375 |         |
|   All |                                                 Min Throughput |        index-append |      142899 |  docs/s |
|   All |                                              Median Throughput |        index-append |      150555 |  docs/s |
|   All |                                                 Max Throughput |        index-append |      159034 |  docs/s |
|   All |                                        50th percentile latency |        index-append |     208.455 |      ms |
|   All |                                        90th percentile latency |        index-append |      286.45 |      ms |
|   All |                                        99th percentile latency |        index-append |     622.145 |      ms |
|   All |                                      99.9th percentile latency |        index-append |      6757.9 |      ms |
|   All |                                     99.99th percentile latency |        index-append |     8428.99 |      ms |
|   All |                                       100th percentile latency |        index-append |     9171.74 |      ms |
|   All |                                   50th percentile service time |        index-append |     208.455 |      ms |
|   All |                                   90th percentile service time |        index-append |      286.45 |      ms |
|   All |                                   99th percentile service time |        index-append |     622.145 |      ms |
|   All |                                 99.9th percentile service time |        index-append |      6757.9 |      ms |
|   All |                                99.99th percentile service time |        index-append |     8428.99 |      ms |
|   All |                                  100th percentile service time |        index-append |     9171.74 |      ms |
|   All |                                                     error rate |        index-append |           0 |       % |
|   All |                                                 Min Throughput |             default |        8.01 |   ops/s |
|   All |                                              Median Throughput |             default |        8.01 |   ops/s |
|   All |                                                 Max Throughput |             default |        8.01 |   ops/s |
|   All |                                        50th percentile latency |             default |     7.87766 |      ms |
|   All |                                        90th percentile latency |             default |     12.2709 |      ms |
|   All |                                        99th percentile latency |             default |     84.0716 |      ms |
|   All |                                       100th percentile latency |             default |     84.1245 |      ms |
|   All |                                   50th percentile service time |             default |     7.68457 |      ms |
|   All |                                   90th percentile service time |             default |     12.0836 |      ms |
|   All |                                   99th percentile service time |             default |     83.9065 |      ms |
|   All |                                  100th percentile service time |             default |     83.9224 |      ms |
|   All |                                                     error rate |             default |           0 |       % |
|   All |                                                 Min Throughput |                term |       49.97 |   ops/s |
|   All |                                              Median Throughput |                term |       50.02 |   ops/s |
|   All |                                                 Max Throughput |                term |       50.06 |   ops/s |
|   All |                                        50th percentile latency |                term |     7.76154 |      ms |
|   All |                                        90th percentile latency |                term |     17.8898 |      ms |
|   All |                                        99th percentile latency |                term |     24.6585 |      ms |
|   All |                                       100th percentile latency |                term |     25.9128 |      ms |
|   All |                                   50th percentile service time |                term |     7.61068 |      ms |
|   All |                                   90th percentile service time |                term |     13.6046 |      ms |
|   All |                                   99th percentile service time |                term |       24.52 |      ms |
|   All |                                  100th percentile service time |                term |     25.7986 |      ms |
|   All |                                                     error rate |                term |           0 |       % |
|   All |                                                 Min Throughput |               range |           1 |   ops/s |
|   All |                                              Median Throughput |               range |        1.01 |   ops/s |
|   All |                                                 Max Throughput |               range |        1.01 |   ops/s |
|   All |                                        50th percentile latency |               range |     22.3311 |      ms |
|   All |                                        90th percentile latency |               range |     96.7736 |      ms |
|   All |                                        99th percentile latency |               range |     97.5008 |      ms |
|   All |                                       100th percentile latency |               range |     100.117 |      ms |
|   All |                                   50th percentile service time |               range |     21.2958 |      ms |
|   All |                                   90th percentile service time |               range |     95.7593 |      ms |
|   All |                                   99th percentile service time |               range |     96.4567 |      ms |
|   All |                                  100th percentile service time |               range |     99.0919 |      ms |
|   All |                                                     error rate |               range |           0 |       % |
|   All |                                                 Min Throughput |          hourly_agg |         0.2 |   ops/s |
|   All |                                              Median Throughput |          hourly_agg |         0.2 |   ops/s |
|   All |                                                 Max Throughput |          hourly_agg |         0.2 |   ops/s |
|   All |                                        50th percentile latency |          hourly_agg |     3202.61 |      ms |
|   All |                                        90th percentile latency |          hourly_agg |     3275.79 |      ms |
|   All |                                        99th percentile latency |          hourly_agg |     3323.96 |      ms |
|   All |                                       100th percentile latency |          hourly_agg |      3434.1 |      ms |
|   All |                                   50th percentile service time |          hourly_agg |     3200.76 |      ms |
|   All |                                   90th percentile service time |          hourly_agg |     3273.89 |      ms |
|   All |                                   99th percentile service time |          hourly_agg |     3322.04 |      ms |
|   All |                                  100th percentile service time |          hourly_agg |      3432.2 |      ms |
|   All |                                                     error rate |          hourly_agg |           0 |       % |
|   All |                                                 Min Throughput |              scroll |       25.03 | pages/s |
|   All |                                              Median Throughput |              scroll |       25.05 | pages/s |
|   All |                                                 Max Throughput |              scroll |       25.11 | pages/s |
|   All |                                        50th percentile latency |              scroll |     590.556 |      ms |
|   All |                                        90th percentile latency |              scroll |     622.417 |      ms |
|   All |                                        99th percentile latency |              scroll |     649.784 |      ms |
|   All |                                       100th percentile latency |              scroll |     662.208 |      ms |
|   All |                                   50th percentile service time |              scroll |     590.094 |      ms |
|   All |                                   90th percentile service time |              scroll |     621.994 |      ms |
|   All |                                   99th percentile service time |              scroll |     649.279 |      ms |
|   All |                                  100th percentile service time |              scroll |     661.741 |      ms |
|   All |                                                     error rate |              scroll |           0 |       % |
|   All |                                                 Min Throughput | desc_sort_timestamp |        0.68 |   ops/s |
|   All |                                              Median Throughput | desc_sort_timestamp |        0.68 |   ops/s |
|   All |                                                 Max Throughput | desc_sort_timestamp |        0.68 |   ops/s |
|   All |                                        50th percentile latency | desc_sort_timestamp |       55718 |      ms |
|   All |                                        90th percentile latency | desc_sort_timestamp |     64624.4 |      ms |
|   All |                                        99th percentile latency | desc_sort_timestamp |     66703.3 |      ms |
|   All |                                       100th percentile latency | desc_sort_timestamp |     66933.6 |      ms |
|   All |                                   50th percentile service time | desc_sort_timestamp |     1468.87 |      ms |
|   All |                                   90th percentile service time | desc_sort_timestamp |     1541.38 |      ms |
|   All |                                   99th percentile service time | desc_sort_timestamp |     1601.69 |      ms |
|   All |                                  100th percentile service time | desc_sort_timestamp |     1606.79 |      ms |
|   All |                                                     error rate | desc_sort_timestamp |           0 |       % |
|   All |                                                 Min Throughput |  asc_sort_timestamp |        0.71 |   ops/s |
|   All |                                              Median Throughput |  asc_sort_timestamp |        0.71 |   ops/s |
|   All |                                                 Max Throughput |  asc_sort_timestamp |        0.71 |   ops/s |
|   All |                                        50th percentile latency |  asc_sort_timestamp |     42312.4 |      ms |
|   All |                                        90th percentile latency |  asc_sort_timestamp |     48421.6 |      ms |
|   All |                                        99th percentile latency |  asc_sort_timestamp |     49846.1 |      ms |
|   All |                                       100th percentile latency |  asc_sort_timestamp |       49996 |      ms |
|   All |                                   50th percentile service time |  asc_sort_timestamp |     1414.08 |      ms |
|   All |                                   90th percentile service time |  asc_sort_timestamp |      1477.3 |      ms |
|   All |                                   99th percentile service time |  asc_sort_timestamp |     1513.77 |      ms |
|   All |                                  100th percentile service time |  asc_sort_timestamp |     1541.05 |      ms |
|   All |                                                     error rate |  asc_sort_timestamp |           0 |       % |


----------------------------------
[INFO] SUCCESS (took 4262 seconds)

```
