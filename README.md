# kafka-strimzi
Laboratório de instalação de um kafka no openshift

O strimzi facilita o deploy de um cluster kafka na plataforma kubernetes

## Documentação do strimzi:
https://strimzi.io/docs/operators/latest/overview

1) Crie o namespace onde será instalado o operator do kafka strimzi

```
oc create namespace kafka-strimzi

oc config set-context --current --namespace=kafka-strimzi
```

2) Instale o strimzi no namespace criado através do operator

Nesse lab foi utilizado a versão 0.45.0 do kafka strimzi

3) Crie o configMap kafka-metrics que será utilizado para coleta de métricas via JMX

Essas métricas serão utilizadas para configuração de alertas e dashboards no grafaba

```
oc apply -f Kafka-Metrics.yaml -n kafka-strimzi
```
