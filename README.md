# apiops-strimzi-tutorial

## Prereqs

This tutorial assumes you have:

- A gravitee license with Kafka Gateway enabled
- A Gravitee API management control plane
- Ability to run a local Minikube cluster (or similar, with minor adjustments required for north/south routing depending on what you use)

## Create a Minikube cluster


```sh
minikube start --memory=4096
```

## Start a Kafka cluster with Strimzi

Install the Strimzi Operator using kubectl or Helm. The below instructions are taken from https://strimzi.io/quickstarts/.

```sh
kubectl create namespace kafka
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
kubectl get pod -n kafka --watch
```

You can check the operator logs as follows:

```sh
kubectl logs deployment/strimzi-cluster-operator -n kafka -f
```

Once the operator is running, create a single node Kafka cluster. The second command will wait till the cluster is ready, takes about a minute for me.

```sh
kubectl apply -f https://strimzi.io/examples/latest/kafka/kraft/kafka-single-node.yaml -n kafka
kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka
```

You can now consume and produce messages. Start a producer with:

```sh
kubectl -n kafka run kafka-producer -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-3.9.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic
```

And a consumer with:

```sh
kubectl -n kafka run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-3.9.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
```

You should be able to pass messages across them. 

## Install cert manager & create certificate

The Gravitee Kafka Gateway will expose a Kafka endpoint over TLS, so we'll create a self-signed certificate using cert-manager. 

First install cert-manager on the cluster:

```sh
helm repo add jetstack https://charts.jetstack.io --force-update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.0 \
  --set crds.enabled=true
```

Create the certificate:

```sh
kubectl apply -f certs.yml
```

*Note: strange thing happening here, my "clusterissuer" is getting created with no namespace assigned. But this isn't preventing things from working for me. Let me know if you have an answer as to why this is.*

## Install Gravitee Kafka Gateway

Kafka Gateway is an enterprise feature. Follow the instructions below to add your enterprise license as a Helm argument:

To get the license.key value, encode your file license.key in base64 and add it to an environment variable (example for MacOS):

```sh
export GRAVITEESOURCE_LICENSE_B64="$(cat license.key | base64)"
```

Obtain your control plane URL, and username and password values for a user, add them to environment variables (replace the values with your own):

```sh
export MANAGEMENT_HTTP_URL="https://your-apim-control-plane.com/"
export MANAGEMENT_HTTP_USERNAME="username"
export MANAGEMENT_HTTP_PASSWORD="password"
```

Now we can run the Helm install command with the example `gateway-values.yaml` provided in this repo:

```sh
helm install gravitee-gateway graviteeio/apim \
    --set license.key=${GRAVITEESOURCE_LICENSE_B64} \
    --set gateway.management.http.url=${MANAGEMENT_HTTP_URL} \
    --set gateway.management.http.username=${MANAGEMENT_HTTP_USERNAME} \
    --set gateway.management.http.password=${MANAGEMENT_HTTP_PASSWORD} \
    -f gateway-values.yaml --create-namespace --namespace gravitee
```

Check inside the gateway pod, that the cert is loaded:

```sh
% kubectl exec -it pods/gravitee-gateway-apim-gateway-7458d9cfb6-p8f9d -- /bin/sh
/opt/graviteeio-gateway $ openssl s_client -showcerts -connect localhost:9092 2>/dev/null | openssl x509 -inform pem -noout -text
```

To make the default service created accessible outside the cluster, we'll run this command in a separate terminal and leave it running:

```sh
minikube tunnel
```

This approach is obviously specific to Minikube, you'll need to adapt it if you're using Kind or another local Kubernetes provider.

You should see the Kafka Gateway's service now exposed to consumers outside the cluster:

```sh
% kubectl get svc
NAME                            TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
gravitee-gateway-apim-gateway   LoadBalancer   10.96.52.164   127.0.0.1     8082:32528/TCP,9092:31339/TCP   10m

```

Now we can run the openssl again direclty from the host machine, because the port 9092 is being redirected to my Kafka Gateway pod:

```sh
openssl s_client -showcerts -connect localhost:9092 2>/dev/null | openssl x509 -inform pem -noout -text
```

This produces a result like the below, notice the "subject alternative name" `*.kafka.local`.

```sh
% openssl s_client -showcerts -connect localhost:9092 2>/dev/null | openssl x509 -inform pem -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            4b:ba:96:e7:1f:9a:fd:77:76:f7:e8:85:39:c9:11:7d
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: O=my-org
        Validity
            Not Before: Mar  5 12:42:48 2025 GMT
            Not After : Jun  3 12:42:48 2025 GMT
        Subject: O=my-org
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:d4:9d:ca:8c:51:bd:34:61:aa:5e:ff:1d:cd:e2:
                    d1:85:d9:83:3b:4b:52:ec:af:4c:ba:d5:95:d3:bb:
                    50:aa:5c:f1:9f:87:b7:56:eb:ef:80:6f:38:70:c5:
                    a0:3a:41:9b:60:c5:76:02:70:3c:5b:94:6b:42:45:
                    c2:73:eb:f5:36:db:42:00:4c:68:c8:e7:ac:07:52:
                    09:3e:a5:68:56:35:12:fb:27:1b:c8:d1:e1:fc:05:
                    67:2c:16:df:99:0e:10:22:23:9a:69:4c:e7:64:13:
                    6d:af:7d:55:10:4f:9d:f6:09:89:45:12:f6:69:38:
                    e7:5e:d2:f9:0c:17:7d:9c:8e:dd:46:d5:bf:05:5c:
                    ca:e2:4f:76:16:38:4e:89:1a:69:e9:1c:c1:1d:b4:
                    63:77:a5:4c:89:5d:96:8d:cd:ac:8b:06:17:8c:08:
                    95:45:13:77:07:3b:db:91:e0:52:3b:10:f4:95:a5:
                    d7:31:54:8c:a7:99:10:8d:dd:8f:ca:fc:d4:d3:0a:
                    20:91:be:0e:95:0f:52:82:c3:07:4d:c4:da:51:36:
                    c5:a6:fe:f3:4d:d4:54:c9:b6:84:b6:33:97:e7:df:
                    d6:ed:21:af:0f:a0:06:da:37:89:c4:ee:14:ea:07:
                    0a:6c:ef:c8:ad:62:96:cf:7e:1f:f6:cc:70:bd:80:
                    46:f5
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Alternative Name: 
                DNS:*.kafka.local
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        40:f4:40:a8:d1:83:3d:1b:5a:cd:f0:22:95:7a:8c:0e:36:2f:
        82:ee:a2:fa:eb:52:f6:3a:11:0c:45:2c:59:aa:19:42:e9:a1:
        84:17:ce:93:fc:cf:3e:58:ac:32:46:11:4d:86:9f:ce:5f:56:
        78:5a:c0:37:6e:cd:b8:c6:aa:b8:e9:cf:60:5e:80:8d:c9:8b:
        c3:7a:9e:ab:1c:ae:0c:24:5f:40:cd:53:14:39:de:57:2f:f7:
        2c:e7:fd:9e:ba:46:58:a5:37:79:10:1d:b4:f1:12:e0:bd:bb:
        cc:62:d7:6d:d6:21:64:6b:2a:cc:28:b6:74:49:8c:52:6b:71:
        98:ae:80:c4:17:05:f2:39:78:35:cb:f0:be:82:5d:da:ac:85:
        0b:c4:18:71:46:c2:67:42:02:5d:05:fc:27:6d:cc:2c:78:15:
        ad:33:30:11:2e:83:bb:12:3e:fa:e9:72:80:28:35:e5:6d:ea:
        ca:69:d5:4d:7f:14:d1:49:a5:b1:ca:4d:8b:68:a5:41:74:b5:
        cc:a6:45:8c:3f:bf:1f:f8:0a:e8:0a:e4:14:ef:6b:bc:cf:dc:
        db:ea:fc:3e:80:cb:42:38:e1:7f:e4:ae:18:3e:db:71:f0:47:
        d3:1c:2c:e2:29:68:6b:04:ee:bb:d0:53:44:c9:aa:38:ba:90:
        19:99:8c:56

```

## Install the Gravitee Kubernetes Operator (GKO)

```sh
helm install graviteeio-gko graviteeio/gko --set manager.logs.format=console -n gravitee
```

Create a management context that will point GKO to your Gravitee APIM control plane.

To do this, you can make a copy of the file `template-management-context.yaml` and call it `management-context.yaml`.

Fill in your own APIM control plane URL, org & env IDs, and credentials. To learn more about how to configure a management context for GKO, take a look at the user guide: https://documentation.gravitee.io/gravitee-kubernetes-operator-gko/guides/define-an-apim-service-account-for-gko.

When you've filled in the file, apply it:

```sh
kubectl apply -f management-context.yaml -n gravitee
```

## Create a Native Kafka API, i.e. a Kafka proxy

Now we can create a native kafka API using the operator. This is going to expose a new host on the Kafka Gateway for Kafka clients to connect to, and will route them to the Strimzi cluster.

```sh
kubectl apply -f native-kafka-api.yaml -n gravitee
```

Note the bootstrap server URL which is pointing to the Strimzi cluster bootstrap service's URL: `bootstrapServers: "my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092"`.

The gateway logs should show this API getting picked up:

```sh
13:04:09.253 [gio.sync-deployer-0] [] INFO  i.g.g.h.a.m.impl.ApiManagerImpl - API id[939c2151-509a-b3a2-b3b5-4330bda1a192] name[kafka-proxy-api] version[1.0] has been deployed
```

## Edit /etc/hosts

This is to enable external access from your machine to the gateway in minikube.

We'll use the following setup because we've only started a single node Kafka cluster, otherwise we would have added other brokers to this list.

```sh
::1          localhost myapi.kafka.local broker-0-myapi.kafka.local
127.0.0.1    localhost myapi.kafka.local broker-0-myapi.kafka.local
```

## Prepare and run a Kafka client

Download and unarchive a Kafka client (in this same directory for simplicity): https://kafka.apache.org/quickstart.

Adapt this to the version you downloaded:

```sh
cd kafka_2.13-3.9.0/bin
```

Get the self-signed cert that we created in Minikube with cert-manager, and store it in a local truststore that we'll give to our Kafka clients later:

```sh
% kubectl get secret my-certificate -o json | jq '.data."ca.crt"' | tr -d '"' | base64 --decode > /tmp/ca.crt
keytool -importcert -alias ca -file /tmp/ca.crt -keystore truststore.jks -storepass gravitee
```

Then you can start a Kafka consumer on your machine (outside of Kubernetes):

```sh
./kafka-console-consumer.sh --bootstrap-server myapi.kafka.local:9092 --topic orders --consumer.config ../../kafka-keyless.properties 
```

As well as a producer:

```sh
./kafka-console-producer.sh --bootstrap-server myapi.kafka.local:9092 --topic orders --producer.config ../../kafka-keyless.properties
```

And if all went well, these should be able to communicate! 

## Next possible evolutions for this guide:

* adding new policies to the `native-kafka-api.yaml` API definition, like ACLs or topic mapping. 
* Setting up a multi-node cluster, and making sure the routing works. See e.g. https://strimzi.io/blog/2019/04/17/accessing-kafka-part-1/