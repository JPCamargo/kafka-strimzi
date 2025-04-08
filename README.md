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

oc get kafka my-cluster -o jsonpath='{.status}' | jq

# Editar configurações do cluster
# Aumentar o número de brokers
oc edit kafka my-cluster

# zookeeper shell
kubectl exec -ti my-cluster-zookeeper-1 -- bin/zookeeper-shell.sh localhost:12181 ls /
kubectl exec -ti my-cluster-zookeeper-1 -- bin/zookeeper-shell.sh localhost:12181 ls /brokers/ids
kubectl exec -ti my-cluster-zookeeper-1 -- bin/zookeeper-shell.sh localhost:12181 ls /brokers/topics
```

5) Criar tópico a partir do arquivo [kafka-topic.yaml](topico-teste-2.yaml)

```
oc apply -f topico-teste-2.yaml

oc get kafkatopics

#verificar as informações sobre o tópico
oc get kafkatopics topico-teste-2 -o json | jq .
```


5) Criar usuário admin do cluster kafka através do arquivo [kafka-admin.yaml](kafka-admin.yaml)

```
oc apply -f kafka-admin.yaml

#listar usuários do kafka
oc get kafkausers

#verificar as informações do usuário
oc get kafkauser admin -o json | jq .

```

### Internal Pod
Vamos iniciar um POD com  imagem da Confluent para realizar alguns testes e gerar nossa keystore.

```
kubectl run my-shell --rm -i --tty --image confluentinc/cp-kafka -- bash
```

- Instale Kubectl.

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && mkdir bin && mv kubectl bin && chmod +x bin/kubectl
```

- Gerando keystore

```
kubectl get secret my-cluster-cluster-ca-cert -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
keytool -import -trustcacerts -alias root -file ca.crt -keystore truststore.jks -storepass password -noprompt
```

- Password user admin

Crie a variável com password do usuário admin.
```
passwd=$(kubectl get secret user-admin -o jsonpath='{.data.password}' | base64 -d)
```

Crie o properties
```
cat <<EOF> conf.properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password=passwd;
ssl.truststore.location=truststore.jks
ssl.truststore.password=password
EOF
```

Adicione o password do usuario admin no conf.properties
```
sed -i "s/passwd/$passwd/g" ./conf.properties
```

- Listar todos os topicos
```
kafka-topics --bootstrap-server my-cluster-kafka-bootstrap:9093 --list --command-config conf.properties
```

- Produzir mensagem

```
kafka-console-producer --bootstrap-server my-cluster-kafka-bootstrap:9093 --producer.config conf.properties --topic topic-test
```

- Consumir mensagem

```
kafka-console-consumer --bootstrap-server my-cluster-kafka-bootstrap:9093 --consumer.config conf.properties --from-beginning --topic topic-test
```

### Produzindo e consumindo com user específicos

```
# Criar o tópico que será utilizado para o teste
oc apply -f topic-kafka-user.yaml
oc get kafkatopics

#criar usuários para produção e consumo nesse tópico
oc apply -f producer-user.yaml
oc apply -f consumer-user.yaml

oc get kafkausers
oc get kafkauser producer-user -o json | jq .
oc get kafkauser consumer-user -o json | jq .

#Acessar o container para produzir mensagens no tópico
oc exec my-shell -i --tty -- bash

#crie a váriavel com o password do producer
oc exec my-shell -i --tty -- bash
passwd=$(kubectl get secret user-producer-user -o jsonpath='{.data.password}' | base64 -d)

#Crie o properties

cat <<EOF> conf.properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="producer-user" password=passwd;
ssl.truststore.location=truststore.jks
ssl.truststore.password=password
EOF

#Adicione o password do usuario producer-user no conf.properties

sed -i "s/passwd/$passwd/g" ./conf.properties

#Listar todos os topicos

kafka-topics --bootstrap-server my-cluster-kafka-bootstrap:9093 --list --command-config conf.properties

#Produzir mensagem

kafka-console-producer --bootstrap-server my-cluster-kafka-bootstrap:9093 --producer.config conf.properties --topic topic-test

```

```
#Hora de consumir as mensagens produzidas
kafka-console-consumer --bootstrap-server my-cluster-kafka-bootstrap:9093 --consumer.config conf.properties --from-beginning --topic topic-test

#Crie o properties

cat <<EOF> confconsumer.properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="consumer-user" password=passwd;
ssl.truststore.location=truststore.jks
ssl.truststore.password=password
EOF

#crie a váriavel com o password do consumer

passwdconsumer=$(kubectl get secret user-consumer-user -o jsonpath='{.data.password}' | base64 -d)

#Adicione o password do usuario consumer-user no conf.properties

sed -i "s/passwd/$passwdconsumer/g" ./confconsumer.properties

kafka-console-consumer --bootstrap-server my-cluster-kafka-bootstrap:9093 --consumer.config confconsumer.properties --from-beginning --topic kafka-user-test

```


- Producer perf test

```

kafka-producer-perf-test --topic topico-teste-2 --num-records 1000000 --record-size 100 --throughput -1 --producer-props acks=all --producer-props partitioner.class=org.apache.kafka.clients.producer.RoundRobinPartitioner bootstrap.servers=my-cluster-kafka-bootstrap:9093 --producer.config conf.properties

```


#comando para acessar o cluster externamente
~/kafka_2.13-3.9.0/bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-external1-bootstrap-kafka-strimzi.apps.okd.34-75-7-172.nip.io:443 --list --command-config conf.properties


Obs.: Foi necessário criar um storage class no openshift com a configuração Volume binding mode = Immediate