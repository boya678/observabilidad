la idea de este stack es usarlo como observabilidad y para el escalada basado en el troughtput de las apis.

lo primero es instalar istio con los siguientes comandos

helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
kubectl create namespace istio-system
helm install istio-base istio/base -n istio-system

para este comando existe el archivo istio-values.yaml en el cual puedes configurar los recursos que le vas a dar al sidecar de istio. en la ruta del yaml

global:
  proxy:
    # Resources for the sidecar.
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 256Mi

el resto de los valores estan por defecto.

helm install istiod istio/istiod -n istio-system --values istio-values.yaml


luego de instalar istio debes habilitar los namespace que deseas que se usen para que el sidecar sea instalado en los pods existentes. Agregando un label al namespace:

istio-injection: enabled

es importante que para los componentes que no sean apis, ejemplo consumer de colas que no reciben peticiones, no te dara mucha información el sidecar, por esta razón es un desperdicio de recursos, esto se evita poniendo la etiqueta:

sidecar.istio.io/inject: "false"

a nivel del pod(osea a nivel de template) en los deployments.


Luego de esto instalamos el stack de observabilidad con los siguientes comandos

kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring-stack prometheus-community/kube-prometheus-stack --namespace monitoring --values values.yaml

en el archivo values esta modificado para que se cree un volumen persistente para prometheus. 
tambien se configura la clave para el grafana en el campo:

adminPassword: admin

puedes agregar un ingress si deseas, etc.

La configuración mas importante del values.yaml del estack de observabilidad es que tiene el config para que prometheus haga scraping de istio.

      - job_name: 'istiod'
        kubernetes_sd_configs:
        - role: endpoints
          namespaces:
            names:
            - istio-system
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: istiod;http-monitoring
      - job_name: 'envoy-stats'
        metrics_path: /stats/prometheus
        kubernetes_sd_configs:
        - role: pod

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_container_port_name]
          action: keep
          regex: '.*-envoy-prom'

Con este stack puedes adicionar tableros a grafana de forma dinamica, en este orden de ideas debes aplicar los tableros como configmaps con un label:

  labels:
    grafana_dashboard: "1"

kubectl apply -f monitoring-stack-istio-request.yaml
kubectl apply -f monitoring-stack-istio-workload.yaml

los tableros que yo he adicionado son:

monitoring-stack-istio-request.yaml: este muestra el troughtput en tiempo real por namespace y muestra los pods health y los que no estan health


monitoring-stack-istio-workload.yaml: este es un tablero estandar de istio.


si deseas exponer grafana bajo un dominio, debes adicionar este dato en el configmap llamado monitoring-stack-grafana luego de instalar el stack

 [security]
    cookie_samesite = disabled

y reiniciar el pod de grafana, esto con el fin de que grafana no valide el dominio de origen y asi no tener que hacer configuraciones en el reverse proxy que estes usando.








