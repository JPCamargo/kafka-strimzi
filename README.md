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

3) Crie o configMap [kafka-metrics.yaml](kafka-metrics.yaml) que será utilizado para coleta de métricas via JMX

Essas métricas serão utilizadas para configuração de alertas e dashboards no grafaba

```
oc apply -f kafka-metrics.yaml -n kafka-strimzi
```

4) Crie o cluster através do arquivo de configuração  [kafka-server.yaml](kafka-server.yaml)

```
oc apply -f kafka-server.yaml -n kafka-strimzi
```
Para verificar que o cluster está pronto para receber conexões, execute o comando a seguir.

```
oc get kafka
```
5) Criar usuário admin do cluster kafka através do 



Obs.: Foi necessário criar um storage class no openshift com a configuração Volume binding mode = Immediate