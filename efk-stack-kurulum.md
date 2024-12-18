# ECK
[https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html)

## Install Operator
```bash
kubectl create -f https://download.elastic.co/downloads/eck/2.14.0/crds.yaml
```

```bash
kubectl apply -f https://download.elastic.co/downloads/eck/2.14.0/operator.yaml
```

```bash
# Monitor operator logs
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```

## Deploy an Elasticsearch cluster
```bash
cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
spec:
  version: 8.15.0
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
EOF
```

```bash
kubectl get elasticsearch
```

```bash
PASSWORD=$(kubectl get secret quickstart-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
```

## Deploy a Kibana instance
```bash
cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: quickstart
spec:
  version: 8.15.0
  count: 1
  elasticsearchRef:
    name: quickstart
EOF
```

```bash
kubectl get kibana
```

```yaml
# kibana-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: quickstart-kibana-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: kibana.example.com # Kullanmak istediğiniz host adıyla değiştirin
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: quickstart-kb-http
            port:
              number: 5601
  tls:
  - hosts:
    - kibana.example.com # Kullanmak istediğiniz host adıyla değiştirin
    secretName: quickstart-kb-http-certs-public # TLS sertifika sırrı için isim
```

```bash
kubectl apply -f kibana-ingress.yaml
```

# Deploy Fluent-bit
[https://docs.fluentbit.io/manual/installation/kubernetes](https://docs.fluentbit.io/manual/installation/kubernetes)
```bash
helm show values fluent/fluent-bit > fluent-bit-values.yaml
```
[https://docs.fluentbit.io/manual/pipeline/outputs](https://docs.fluentbit.io/manual/pipeline/outputs)
```toml
[OUTPUT]
    Name es
    Match kube.*
    Host quickstart-es-http
    Logstash_Prefix_Key kubernetes['pod_name']
    Port 9200
    HTTP_User elastic
    HTTP_Passwd <pass>
    tls on
    tls.verify false
    Logstash_Format On
    Retry_Limit False
    Suppress_Type_Name On
```

```bash
helm install fluent-bit fluent/fluent-bit -f fluent-bit-values.yaml
```