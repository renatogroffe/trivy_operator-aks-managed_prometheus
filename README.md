# trivy_operator-aks-managed_prometheus
Instruções para configuração do Trivy Operator no scanning de vulnerabilidades em um cluster do Azure Kubernetes Service que faz uso do Azure Monitor managed service for Prometheus (Azure Managed Prometheus).

## Instruções para uso do Trivy Operator no Azure

O arquivo **trivy-values.yaml** contém algumas configurações para tornar possível a integração do Trivy Operator com o :

```yaml
serviceMonitor:
  # Desativa o deployment do serviceMonitor com configurações default
  enabled: false
trivy:
  ignoreUnfixed: true
service:
  # Disabled garante que o Pod receba um Cluster IP
  headless: false
operator:
  # Habilita a coleta de métricas (detalhamento de vulnerabilidades) do Trivy Operator
  metricsVulnIdEnabled: true
```

Instalando o Trivy Operator num cluster do Azure Kubernetes Service, com o Prometheus gerenciado já ativado:

```bash
helm install trivy-operator aqua/trivy-operator --namespace trivy-system --create-namespace --version 0.35.0 --values trivy-values.yaml
```

A execução deste comando produzirá um resultado similar ao seguinte:

```text
NAME: trivy-operator
LAST DEPLOYED: Sun Aug 16 18:38:58 2026
NAMESPACE: trivy-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
You have installed Trivy Operator in the trivy-system namespace.
It is configured to discover Kubernetes workloads and resources in
all namespace(s).

Inspect created VulnerabilityReports by:

    kubectl get vulnerabilityreports --all-namespaces -o wide

Inspect created ConfigAuditReports by:

    kubectl get configauditreports --all-namespaces -o wide

Inspect the work log of trivy-operator by:

    kubectl logs -n trivy-system deployment/trivy-operator
```

Executar agora o comando:

```bash
kubectl get svc trivy-operator -n trivy-system -o yaml
```

Um possível retorno do mesmo seria:

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: trivy-operator
    meta.helm.sh/release-namespace: trivy-system
  creationTimestamp: "2026-08-16T20:37:38Z"
  labels:
    app.kubernetes.io/instance: trivy-operator
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: trivy-operator
    app.kubernetes.io/version: 0.33.0
    helm.sh/chart: trivy-operator-0.35.0
  name: trivy-operator
  namespace: trivy-system
  resourceVersion: "6691757"
  uid: bd9b1290-1096-4f8e-9727-524188afce95
spec:
  clusterIP: 10.0.235.141
  clusterIPs:
  - 10.0.235.141
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - appProtocol: TCP
    name: metrics
    port: 80
    protocol: TCP
    targetPort: metrics
  selector:
    app.kubernetes.io/instance: trivy-operator
    app.kubernetes.io/name: trivy-operator
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

O nome da porta (**spec.ports.name**) que está na definição do Trivy Operator deve ser o mesmo que consta no arquivo **trivy-servicemonitor.yaml** (**spec.endpoints.port**). Já o uso de **azmonitoring.coreos.com/v1** em **apiVersion** indica ao Service Monitor que será criado o uso do Prometheus gerenciado no Azure: 

```yaml
apiVersion: azmonitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: trivy-operator
  namespace: trivy-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/instance: trivy-operator
      app.kubernetes.io/name: trivy-operator

  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
      honorLabels: true

  namespaceSelector:
    matchNames:
      - trivy-system
```

Criar agora o Service Monitor com a seguinte instrução (o processo de "scraping" do Prometheus ocorrerá a cada 30 segundos, como indicado em **trivy-servicemonitor.yaml**):

```bash
kubectl apply -f .\trivy-servicemonitor.yaml -n trivy-system
```

Produzindo como resultado:

```
servicemonitor.azmonitoring.coreos.com/trivy-operator created
```