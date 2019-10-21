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
In this scenario the only major diffrence would be to point HPA not to external metrics from AWS Cloud Watch but to metrics from metric-server (which has to be deployed as a plugin to the cluster beforehand)
