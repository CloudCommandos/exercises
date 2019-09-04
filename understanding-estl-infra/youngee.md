# Logging & Monitoring

The main difference is the deployment of containers, where we used Kubernetes to deploy our containers.

## Monitoring

Both our monitoring tools are the same, which is Prometheus. However, our Prometheus is deploy in our Kubernetes. Though our Prometheus would give us the HA capability, I do understand the dependency on Kubernetes which might lead to down time should there be an update on Kubernetes.
Our Prometheus deployment is also mainly rely on [prometheus-operator](https://github.com/coreos/kube-prometheus), which have a sets of pre-define configuration and making it easier to utilise.
One of the difference I have noticed is the configuration on alertmanager.yaml, where a template file is used instead of putting the Slack template in the yaml file.
Generally, for monitoring, the setup is quite similar.

## Logging
The logging stack we are using is EFK Stack. Currently, we are still in the midst of improvising it. Happened to found [Elasticsearch Operator]( https://github.com/elastic/cloud-on-k8s) and looking into it to integrate with our infra. I am try to understand Graylog for logging, the implementation is quite difference from EFK Stack. Moreover, our EFK stack is deploy in Kubernetes and minimally configured.
I noticed the recent change in the log collector/aggregator to Fluent Bit on your configuration. I am looking into Fluent Bit as well as it is more lightweight than FluentD.
Currently, there is still a lot of work to improvise our logging stack.
