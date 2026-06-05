# Strimizi Mutual Authentication 

### Prerequisites

- streams for Apache Kafka operator installed in Openshift Container Platform
- openssl tool 
- KafkaNodePool already created in the Openshift project

### Create Kafka cluster

```sh
oc create -f kafka.yml
```

### Create the user in the kafka cluster

- Create the client certificate that will be used to authenticate in the Kafka cluster

```sh
oc create -f kafka-user.yml
```

### Prepare the certificates

- Get the certificates from the Kafka cluster

Get the user certificate:
```sh
oc get secret test-kafka-user -o jsonpath='{.data.user\.p12}'  | base64 -d  > test-kakfa-user.p12
```

Get the broker certificate:
```sh
oc get secret test-kafka-cluster-cluster-ca-cert -o jsonpath='{.data.ca\.p12}' | base64 -d  > test-kafka-cluster-ca.p12
```

### Run the client

- Run the client producer:
```sh
./kafka-console-producer.sh   --bootstrap-server citi-kafka-cluster-kafka-tls-bootstrap-citi-04461950.apps-crc.testing:443    --topic test-topic   --command-config client.properties
```

/Users/asouza/redhat/middleware/streams/streams-3.2/bin/kafka-console-producer.sh --bootstrap-server citi-kafka-cluster-kafka-tls-bootstrap-citi-04461950.apps-crc.testing:443 --topic test-topic --command-config client.properties

## Now with custom certificates

- Create the custom certificates
```sh 
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout custom-key.key -out custom-cert.crt
openssl pkcs12 -export -out custom-certificate.p12  -inkey custom-key.key -in custom-cert.crt
```

- Import the client certificate into a Opensshift secret (on your kafka cluster project)

```sh
oc create secret tls custom-certificate --cert=custom-cert.crt --key=custom-key.key
```

- Create Kafka cluster

```sh
oc create -f kafka-custom-certificate.yml
```

- Get the new custom certificate from the cluster

```sh
oc get secret custom-certificate -o jsonpath='{.data.tls\.crt}'  | base64 -d  > custom-cert.crt
```