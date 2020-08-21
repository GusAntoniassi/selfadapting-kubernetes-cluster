# Self-Adapting Kubernetes Cluster
Esse é o código para provisionamento da infraestrutura de exemplo para a talk "Infraestrutura autoadaptável e 
autorregenerativa com software livre". Esse repositório ainda é um trabalho em progresso.

## Requisitos
- kops
- Helm V3

## Criação do cluster
```shell script
export STATE_BUCKET_NAME="my-bucket" # nome de um bucket no S3 para armazenar o state do kops
kops create -f infrastructure/selfadapting.k8s.local --state s3://${STATE_BUCKET_NAME}
kops create secret sshpublickey admin -i ~/.ssh/id_rsa.pub --state s3://${STATE_BUCKET_NAME} --name selfadapting.k8s.local
kops update cluster --name selfadapting.k8s.local --state s3://${STATE_BUCKET_NAME} --yes
kops validate cluster --wait 10m --state s3://${STATE_BUCKET_NAME}
```

## Implementar stack de proxy
```shell script
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx -f infrastructure/proxy/ingress-nginx-values.yaml -n proxy --create-namespace
# Rodar esse comando sempre que quiser acessar uma aplicação no cluster
kubectl --namespace proxy port-forward $(kubectl --namespace proxy get pods -o jsonpath="{.items[0].metadata.name}" -l "app.kubernetes.io/name=ingress-nginx") 8080:80
```

## Implementar stack de monitoramento
```shell script
# metrics server
helm install metrics-server stable/metrics-server -f infrastructure/monitoring/metrics-server-values.yaml -n monitoring --create-namespace
# prometheus 
helm install prometheus stable/prometheus-operator -f infrastructure/monitoring/prometheus-values.yaml -n monitoring
```

## Configuração do arquivo de hosts
Para não precisar mexer com criação de DNS, utilizarei o `kubectl proxy` e o arquivo de hosts locais para fazer o
redirecionamento dos domínios do ingress.

Edite seu arquivo de hosts (linux: `/etc/hosts`) adicionando as seguintes entradas:
```shell script
127.0.0.1 alertmanager.local
127.0.0.1 prometheus.local
127.0.0.1 grafana.local
127.0.0.1 kube-dashboard.local
127.0.0.1 graylog.local
127.0.0.1 gcp-store.local
```

Acessos:
- http://grafana.local:8080
- http://alertmanager.local:8080
- http://prometheus.local:8080

## Implementar stack aplicações de exemplo
```shell script
kubectl create namespace gcp-microservices
kubectl apply -n gcp-microservices -f gcp-microservices-demo/kubernetes-manifests.yaml
```

Acessos:
- http://frontend.local:8080

## Limpeza
```shell script
kops delete cluster --name selfadapting.k8s.local --state s3://${STATE_BUCKET_NAME} --yes
```