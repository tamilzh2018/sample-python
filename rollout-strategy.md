Using **Argo CD**, note that **Argo CD itself does not perform automatic rollbacks based on metrics**. Argo CD is primarily a GitOps deployment controller.

For automated rollback based on:

* Error rate
* Latency
* Health checks
* CPU/Memory
* Synthetic tests

the recommended approach is:

**Argo CD + Argo Rollouts + Prometheus**

Architecture:

```text
Git Repo
   |
   v
Argo CD
   |
   v
Argo Rollouts (Canary/Blue-Green)
   |
   +--> Prometheus Metrics
   +--> Synthetic Tests
   +--> Health Checks

If analysis fails:
    -> Abort Rollout
    -> Automatically revert traffic
    -> Keep stable version running
```

### Example Rollout with Automatic Rollback

apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
name: my-app
spec:
replicas: 5

revisionHistoryLimit: 5

selector:
matchLabels:
app: my-app

template:
metadata:
labels:
app: my-app
spec:
containers:
- name: my-app
image: myrepo/my-app:v2

strategy:
canary:
maxSurge: 1
maxUnavailable: 0

```
  steps:
  - setWeight: 20

  - pause:
      duration: 5m

  - analysis:
      templates:
      - templateName: app-metrics-check

  - setWeight: 50

  - pause:
      duration: 5m

  - analysis:
      templates:
      - templateName: app-metrics-check

  - setWeight: 100
```

### Analysis Template

apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
name: app-metrics-check

spec:
metrics:

* name: error-rate
  interval: 1m
  count: 3
  failureLimit: 1
  successCondition: result[0] < 5
  provider:
  prometheus:
  address: [http://prometheus.monitoring.svc.cluster.local:9090](http://prometheus.monitoring.svc.cluster.local:9090)
  query: |
  (
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
  ) * 100

* name: latency-p95
  interval: 1m
  count: 3
  failureLimit: 1
  successCondition: result[0] < 1000
  provider:
  prometheus:
  address: [http://prometheus.monitoring.svc.cluster.local:9090](http://prometheus.monitoring.svc.cluster.local:9090)
  query: |
  histogram_quantile(
  0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
  ) * 1000

* name: cpu-usage
  interval: 1m
  count: 3
  failureLimit: 1
  successCondition: result[0] < 80
  provider:
  prometheus:
  address: [http://prometheus.monitoring.svc.cluster.local:9090](http://prometheus.monitoring.svc.cluster.local:9090)
  query: |
  avg(
  rate(container_cpu_usage_seconds_total{pod=~"my-app.*"}[5m])
  ) * 100

* name: memory-usage
  interval: 1m
  count: 3
  failureLimit: 1
  successCondition: result[0] < 85
  provider:
  prometheus:
  address: [http://prometheus.monitoring.svc.cluster.local:9090](http://prometheus.monitoring.svc.cluster.local:9090)
  query: |
  avg(
  container_memory_working_set_bytes{pod=~"my-app.*"}
  /
  container_spec_memory_limit_bytes{pod=~"my-app.*"}
  ) * 100

### Synthetic Test Rollback

You can also trigger rollback using a Kubernetes Job:

```yaml id="8ezv3i"
- name: checkout-flow
  interval: 1m
  count: 1
  failureLimit: 1
  provider:
    job:
      spec:
        template:
          spec:
            restartPolicy: Never
            containers:
            - name: synthetic-test
              image: curlimages/curl
              command:
              - sh
              - -c
              - |
                curl -f https://api.example.com/health
                curl -f https://api.example.com/login
```

### Argo CD Configuration

Your Argo CD Application should enable automated sync:

```yaml id="yxhl9d"
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### Rollback Flow

1. Developer pushes image `v2`.
2. Argo CD syncs manifests.
3. Argo Rollouts starts canary deployment.
4. Prometheus metrics are evaluated.
5. Synthetic tests execute.
6. If any check fails:

   * Rollout is **aborted**.
   * Traffic remains on the stable ReplicaSet.
   * New pods are scaled down.
7. Alert is sent to Slack/PagerDuty.

This is the standard production-grade pattern for GitOps deployments with Argo CD. Argo CD handles synchronization, while Argo Rollouts handles progressive delivery and automated rollback decisions.
