# devops-assignment-easir
## Topic: How would you implement scalling for Kubernetes pods based on incomming HTTP connections and/or RabbitMQ queue length?

### Assumptions
Scenario 1: Using AWS technology stack only

Scenario 2: Some considaration around on-prem use case

### Scenario 1
My suggestions would be to use following tech stack:
* Amazon EKS with Horizontal Pod Autoscaler and Kubernetes Metrics Server support (supported since Aug 2018) 
* k8s-cloudwatch-adapter - allows to scale Kubernetes deployment using the HPA with CloudWatch metrics.
* AWS CloudWatch to gather metrics from Application Load Balancer on HTTP requests as well as from AWS SQS on queue status or RabbitMQ queue status
* In order to monitor RabbitMQ queue with CloudWatch we need to use following adapter: https://github.com/deepakputhraya/monitor-rabbitmq

If it comes to more detailed technical steps we could follow is:

(Assuming that we have K8s cluster spawned and configured)
* I'd follow AWS post around configuring k8s-cloudwatch-adapter - https://aws.amazon.com/blogs/compute/scaling-kubernetes-deployments-with-amazon-cloudwatch-metrics/ - of course adjusting where necessary
* Then I'd configure RabbitMQ adapter for CloudWatch to send relevant metrics
* Once I have all relevant metrics in CloudWatch (App Load Balancer metrics are supported by CloudWatch by default) I'd create k8s ExternalResource object to point to those metrics.
* And then map to it in configuration of HPA

That would be more or less it. 

### Scenario 2
In this scenario the only major diffrence would be to point HPA not to external metrics from AWS Cloud Watch but to custom metrics within on-prem cluster. From what I remember, EASI'R has already implemented Prometheus monitoring so in this case I'd use prometheus-adapter (https://github.com/DirectXMan12/k8s-prometheus-adapter) to leverage Custom Metrics API and use custom.metrics.k8s.io/v1beta1 which is suitable for use with the auto scaling/v2 Horizontal Pod Autoscaler.

Then we could levarage following ConfigMap to tell prometheus-adapter to gather specific metric (in this case HTTP requests):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: prometheus-adapter
    chart: prometheus-adapter-v0.1.2
    heritage: Tiller
    release: prometheus-adapter
  name: prometheus-adapter
data:
  config.yaml: |
    rules:
    - seriesQuery: 'http_requests_total{kubernetes_namespace!="",kubernetes_pod_name!=""}'
      resources:
        overrides:
          kubernetes_namespace: {resource: "namespace"}
          kubernetes_pod_name: {resource: "pod"}
      name:
        matches: "^(.*)_total"
        as: "${1}_per_second"
      metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
...
```

Finally we would need to set up the HPA (Horizontal Pod Autoscaler) for our deployment:
```yaml
---
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: test-app
spec:
  scaleTargetRef:
    # point the HPA to sample app
    #apiVersion: apps/v1
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: test-app
  minReplicas: 2
  maxReplicas: 5
  metrics:
  # use a "Pods" metric, which takes the average of the
  # given metric across all pods controlled by the autoscaling target
  - type: Pods
    pods:
      # use the metric that you used above: pods/http_requests
      metricName: http_requests
      # target 500 milli-requests per second,
      # which is 1 request every two seconds
      targetAverageValue: 500m
```
(samples from: https://icicimov.github.io/blog/kubernetes/Kubernetes_HPA_Autoscaling_with_Custom_Metrics/)

