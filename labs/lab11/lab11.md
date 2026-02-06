---
layout: lab
title: "Práctica 11: Monitoreo y métricas de costos con herramientas OSS"
permalink: /lab11/lab11/
images_base: /labs/lab11/img
duration: "60 minutos"
objective:
  - "Implementar en Amazon EKS un stack OSS para observabilidad + costos usando Prometheus + Grafana (kube-prometheus-stack) y OpenCost, para medir costos por namespace/labels, consultar métricas con PromQL y consumir reportes vía API (/allocation), validando cómo requests/limits, etiquetas y namespaces impactan la imputación de costos (showback/chargeback) con troubleshooting estilo CKAD."
prerequisites:
  - "Cuenta AWS con permisos para: EKS, IAM, EC2, VPC, CloudFormation (si creas clúster), ECR/Public ECR"
  - "Windows + Visual Studio Code + Terminal GitBash"
  - "Herramientas: AWS CLI v2, kubectl, eksctl, helm 3, git"
  - "(Opcional) jq para procesar JSON en CLI"
  - "Permisos Kubernetes: crear namespaces, CRDs (Prometheus Operator), Deployments, Services"
  - "Recomendado: mínimo 2 nodos (capacidad suficiente para Prometheus/Grafana/OpenCost)"
introduction:
  - "OpenCost es un proyecto open source para medir y asignar costos de infraestructura y workloads en Kubernetes. Se integra con Prometheus para almacenar/consultar métricas y expone una API (por ejemplo /allocation) y una UI para visualizar tendencias. La imputación por namespace/labels es especialmente útil cuando los workloads declaran requests/limits y usan etiquetas consistentes (showback/chargeback)."
slug: lab11
lab_number: 11
final_result: >
  Al finalizar tendrás un stack OSS funcional en EKS: kube-prometheus-stack (Prometheus Operator + Prometheus + Grafana) recolectando métricas del clúster, OpenCost calculando/exportando métricas de costo e integrándose con Prometheus, y un set de workloads de prueba para demostrar cómo requests/limits + namespaces + labels impactan la imputación; con evidencia por UI/PromQL/API y troubleshooting tipo CKAD.
notes:
  - "CKAD: práctica núcleo (Resource Requests/Limits, namespaces, labels/selectors, troubleshooting con kubectl describe/events/logs/rollout, validación de estado con métricas)."
  - "No asumas nada: valida nombres reales de Services/Secrets/Pods con kubectl antes de port-forward o de leer credenciales."
  - "Evita LoadBalancer en labs: usa port-forward para no crear recursos AWS adicionales con costo."
references:
  - text: "kube-prometheus-stack (Helm chart) - prometheus-community"
    url: https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack
  - text: "OpenCost - Managing with Helm"
    url: https://opencost.io/docs/installation/helm/
  - text: "OpenCost - Working with Prometheus (exporter + ejemplos PromQL)"
    url: https://opencost.io/docs/integrations/prometheus/
  - text: "OpenCost - Metrics reference"
    url: https://opencost.io/docs/integrations/metrics/
  - text: "OpenCost - API (puertos 9003/9090 y endpoints)"
    url: https://opencost.io/docs/integrations/api/
  - text: "OpenCost - API Examples (/allocation, aggregate, window, resolution)"
    url: https://opencost.io/docs/integrations/api-examples/
  - text: "Kubernetes - Resource Management for Pods (requests/limits)"
    url: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
prev: /lab10/lab10/
next: /lab12/lab12/
---

## Costo (resumen)

- **EKS** cobra por clúster por hora. Si tu clúster entra en **Extended Support** (según versión), el costo por hora puede ser mayor.
- **kube-prometheus-stack** y **OpenCost** son OSS (sin costo por licencia), pero consumen **CPU/Mem/Storage** en el clúster.
- Evita `Service type LoadBalancer` en el laboratorio para no crear **ELB/NLB**. Usa `port-forward`.

> **IMPORTANTE:** Este laboratorio instala componentes relativamente “pesados” (Prometheus + Grafana). Haz la práctica en **dev/stage** y elimina recursos al final si es un clúster de laboratorio.
{: .lab-note .important .compact}

---

### Tarea 1. Preparación del workspace y variables

Crearás la carpeta de la práctica, validarás herramientas, autenticarás AWS y dejarás listas variables de entorno reutilizables para operar EKS/Helm desde GitBash.

> **NOTA (CKAD):** Entrenas “baseline” real: contexto correcto, pods del sistema, eventos recientes y por qué un Pod queda `Pending`.
{: .lab-note .info .compact}

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo del curso con un usuario con permisos administrativos.

- {% include step_label.html %} Abre **Visual Studio Code**.

- {% include step_label.html %} Abre la terminal integrada en VS Code.

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **Git Bash** como terminal.

  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de estar en la carpeta raíz de tus laboratorios (por ejemplo **`labs-eks-ckad`**).

  > Si vienes de otra práctica, usa `cd ..` hasta volver a la raíz.
  {: .lab-note .info .compact}

- {% include step_label.html %} Crea la carpeta del laboratorio y la estructura estándar.

  ```bash
  mkdir lab11 && cd lab11
  mkdir -p k8s helm outputs logs scripts notes
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que la estructura de carpetas quedó creada.

  ```bash
  find . -maxdepth 2 -type d | sort
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea un archivo `.env` con variables base (recomendado para reproducibilidad).

  ```bash
  cat > .env <<'EOF'
  export AWS_REGION="us-west-2"

  # Si ya tienes clúster, pon su nombre aquí.
  # Si no tienes, lo crearás en la Tarea 2.
  export CLUSTER_NAME="eks-oss-cost-lab"

  # Namespaces plataforma
  export MON_NS="monitoring"
  export OC_NS="opencost"

  # Releases Helm
  export MON_RELEASE="monitoring"
  export OC_RELEASE="opencost"
  EOF
  ```
  ```bash
  source .env
  ```

- {% include step_label.html %} Verifica que tus CLIs responden correctamente.

  ```bash
  aws --version 
  ```
  ```bash
  kubectl version --client=true
  ```
  ```bash
  eksctl version
  ```
  ```bash
  helm version --short
  ```
  ```bash
  jq --version
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica tu identidad en AWS (si falla, corrige credenciales antes de continuar).

  ```bash
  aws sts get-caller-identity | tee outputs/06_aws_identity.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda tu `AWS_ACCOUNT_ID` (se reutiliza en diagnósticos).

  ```bash
  export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  echo "AWS_ACCOUNT_ID=$AWS_ACCOUNT_ID" | tee outputs/07_aws_account_id.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. (Opcional) Crear un clúster EKS para esta práctica

Crea un clúster EKS con Managed Node Group usando `eksctl`. **Omite esta tarea** si ya tienes un clúster y conoces su `CLUSTER_NAME`.

> **IMPORTANTE:** Crear clúster genera costos. Úsalo solo si no cuentas con un clúster de laboratorio.
{: .lab-note .important .compact}

#### Tarea 2.1

- {% include step_label.html %} Define variables del clúster de laboratorio (ajusta si deseas).

  ```bash
  export NODEGROUP_NAME="mng-1"
  export K8S_VERSION="1.33"
  ```

- {% include step_label.html %} (Recomendado) Elige un tamaño de nodo con suficiente memoria para Prometheus/Grafana (por ejemplo `t3.large` o superior).

  > **IMPORTANTE:** Si usas nodos muy pequeños, es común ver `Pending` por falta de CPU/Mem.
  {: .lab-note .important .compact}

- {% include step_label.html %} Crea el clúster con un Managed Node Group (mínimo 2 nodos).

  > **IMPORTANTE:** El cluster tardara aproximadamente **15 minutos** en crearse. Espera el proceso
  {: .lab-note .important .compact}

  ```bash
  eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --version "$K8S_VERSION" \
    --managed \
    --nodegroup-name "$NODEGROUP_NAME" \
    --node-type t3.large \
    --nodes 2 --nodes-min 2 --nodes-max 3
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el clúster quedó `ACTIVE`.

  ```bash
  aws eks describe-cluster \
    --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query "cluster.status" --output text | tee outputs/08_cluster_status.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---

### Tarea 3. Conectar kubectl al clúster y validar baseline

Actualizarás tu kubeconfig y verificarás que puedes listar nodos, namespaces y eventos recientes.

#### Tarea 3.1

- {% include step_label.html %} Actualiza tu kubeconfig hacia el clúster objetivo.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma el contexto actual.

  ```bash
  kubectl config current-context | tee outputs/09_kubectl_context.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el control plane responde.

  ```bash
  kubectl cluster-info | tee outputs/10_cluster_info.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista nodos y confirma que están `Ready`.

  ```bash
  kubectl get nodes -o wide | tee outputs/11_nodes.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Baseline rápido: pods y eventos recientes.

  ```bash
  kubectl get pods -A | head -n 40 | tee outputs/12_pods_head.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get events -A --sort-by=.lastTimestamp | tail -n 40 | tee outputs/13_events_tail.txt
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

### Tarea 4. Instalar Prometheus + Grafana (kube-prometheus-stack)

Instalarás Prometheus Operator + Prometheus + Grafana + exporters usando Helm. Este stack facilita integrar OpenCost vía **ServiceMonitor** sin escribir configs manuales.

> **IMPORTANTE:** kube-prometheus-stack instala CRDs del Prometheus Operator. Si ya tienes otro Prometheus Operator, revisa duplicidad/compatibilidad.
{: .lab-note .important .compact}

#### Tarea 4.1 — Namespace + Helm repo

- {% include step_label.html %} Crea el namespace de monitoreo de forma idempotente.

  ```bash
  kubectl create ns "$MON_NS" --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}
  ```bash
  kubectl get ns "$MON_NS" -o wide | tee outputs/14_mon_ns.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Agrega y actualiza el repo Helm de Prometheus Community.

  ```bash
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm repo update
  ```
  {% include step_image.html %}
  ```bash
  helm search repo prometheus-community/kube-prometheus-stack | head -n 10 | tee outputs/15_helm_search_kps.txt
  ```
  {% include step_image.html %}

#### Tarea 4.2 — Values (selectores abiertos para ServiceMonitor/PodMonitor)

- {% include step_label.html %} Crea `helm/values-monitoring.yaml`.

  > **NOTA:** Esto permite descubrir ServiceMonitors/PodMonitors externos sin requerir labels “del chart”.
  {: .lab-note .info .compact}

  ```bash
  cat > helm/values-monitoring.yaml <<'EOF'
  prometheus:
    prometheusSpec:
      serviceMonitorSelectorNilUsesHelmValues: false
      podMonitorSelectorNilUsesHelmValues: false
      ruleSelectorNilUsesHelmValues: false
      probeSelectorNilUsesHelmValues: false

  grafana:
    enabled: true

  alertmanager:
    enabled: false
  EOF
  ```

- {% include step_label.html %} Instala/actualiza el release de monitoreo.

  ```bash
  helm upgrade --install "$MON_RELEASE" prometheus-community/kube-prometheus-stack \
    -n "$MON_NS" -f helm/values-monitoring.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida CRDs del Prometheus Operator (evidencia).

  ```bash
  kubectl get crd | egrep "monitoring.coreos.com" | tee outputs/16_prom_operator_crds.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get crd | egrep "servicemonitors|podmonitors|prometheuses|alertmanagers" | tee outputs/17_monitor_crds.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica pods/servicios y espera readiness con `kubectl wait`.

  > **Tip (no asumir labels):** Si necesitas validar Prometheus por nombre, lista pods y filtra: `kubectl -n "$MON_NS" get pods | grep -i prometheus`
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$MON_NS" get pods -o wide | tee outputs/18_mon_pods.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$MON_NS" get svc -o wide | tee outputs/19_mon_svc.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$MON_NS" wait --for=condition=Ready pod \
    -l app.kubernetes.io/name=grafana --timeout=600s
  ```
  {% include step_image.html %}

#### Tarea 4.3 — Acceso local (port-forward)

> **IMPORTANTE:** Para evitar costos AWS, usa `port-forward`. No uses LoadBalancer.
{: .lab-note .important .compact}

- {% include step_label.html %} Obtén el password admin de Grafana (no asumas secretos; valida primero). **Guarda la contraseña en un bloc de notas**

  ```bash
  kubectl -n "$MON_NS" get secret | grep -i grafana | tee outputs/20_grafana_secrets.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$MON_NS" get secret "${MON_RELEASE}-grafana" \
    -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
  ```
  {% include step_image.html %}

- {% include step_label.html %} Abre una **segunda terminal** Port-forward a Grafana.

  ```bash
  cd lab11
  source .env
  kubectl -n "$MON_NS" get svc | grep -i grafana | tee outputs/21_grafana_svc_candidates.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$MON_NS" port-forward svc/${MON_RELEASE}-grafana 3000:80
  ```
  {% include step_image.html %}

- {% include step_label.html %} Abre el navegador **Chrome/Edge** y escribe la siguiente URL para abrir Grafana.

  - Recuerda que el usuario es: **`admin`**
  - La contraseña es la que guardaste en el bloc de notas.
  
  **UI**
  ```bash
  http://localhost:3000
  ```
  {% include step_image.html %}

- {% include step_label.html %} En una **tercera terminal**, realiza port-forward a Prometheus.

  ```bash
  cd lab11
  source .env
  kubectl -n "$MON_NS" get svc | grep -i prometheus | tee outputs/22_prom_svc_candidates.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$MON_NS" port-forward svc/${MON_RELEASE}-kube-prometheus-prometheus 9090:9090
  ```
  {% include step_image.html %}

- {% include step_label.html %} Abre el navegador **Chrome/Edge** en otra pestaña y escribe la siguiente URL para abrir Prometheus.
 
  **UI**
  ```bash
  http://localhost:9090
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Instalar OpenCost e integrarlo con Prometheus

Instalarás OpenCost con Helm y lo conectarás al Prometheus del stack. Habilitarás scraping (ServiceMonitor) y validarás targets UP, métricas presentes y acceso a UI/API.

#### Tarea 5.1 — Namespace + Helm repo

- {% include step_label.html %} Ahora **regresa a la primera terminal** y crea el namespace de OpenCost de forma idempotente.

  ```bash
  kubectl create ns "$OC_NS" --dry-run=client -o yaml | kubectl apply -f -
  ```
  ```bash
  kubectl get ns "$OC_NS" -o wide | tee outputs/23_oc_ns.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Agrega el repo Helm de OpenCost y actualiza.

  ```bash
  helm repo add opencost-charts https://opencost.github.io/opencost-helm-chart
  helm repo update
  ```
  {% include step_image.html %}
  ```bash
  helm search repo opencost-charts/opencost | head -n 10 | tee outputs/24_helm_search_opencost.txt
  ```
  {% include step_image.html %}

#### Tarea 5.2 — Descubrir el Service real de Prometheus (sin asumir nombres)

- {% include step_label.html %} Identifica el Service de Prometheus dentro del namespace de monitoring.

  ```bash
  kubectl -n "$MON_NS" get svc -o wide | tee outputs/25_mon_svc_again.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define una variable `PROM_SVC` (elige el Service correcto de la lista).

  > **NOTA:** Normalmente es: `${MON_RELEASE}-kube-prometheus-prometheus`
  {: .lab-note .info .compact}

  ```bash
  export PROM_SVC="${MON_RELEASE}-kube-prometheus-prometheus"
  echo "PROM_SVC=$PROM_SVC" | tee outputs/26_prom_svc_selected.txt
  ```
  {% include step_image.html %}

#### Tarea 5.3 — Values OpenCost (Prometheus externo + ServiceMonitor)

- {% include step_label.html %} Crea `helm/values-opencost.yaml` apuntando a Prometheus (service DNS dentro del clúster).

  ```bash
  cat > helm/values-opencost.yaml <<EOF
  opencost:
    prometheus:
      internal:
        enabled: false
      external:
        enabled: true
        url: "http://${PROM_SVC}.${MON_NS}:9090"

    metrics:
      serviceMonitor:
        enabled: true
        namespace: "${MON_NS}"
        additionalLabels:
          release: "${MON_RELEASE}"

    ui:
      enabled: true
  EOF
  ```

- {% include step_label.html %} Instala/actualiza OpenCost.

  ```bash
  helm upgrade --install "$OC_RELEASE" opencost-charts/opencost \
    -n "$OC_NS" --create-namespace \
    -f helm/values-opencost.yaml \
    --wait
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida pods, servicios y ServiceMonitor (si se creó).

  ```bash
  kubectl -n "$OC_NS" get pods -o wide | tee outputs/27_oc_pods.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$OC_NS" get svc -o wide | tee outputs/28_oc_svc.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$MON_NS" get servicemonitors | grep -i opencost | tee outputs/29_sm_opencost.txt || true
  ```
  {% include step_image.html %}

#### Tarea 5.4 — Acceso local a OpenCost (UI y API)

> **NOTA:** OpenCost expone API en **9003** y UI en **9090**. Como Prometheus usa 9090 local, mapearemos OpenCost UI a **9091** local.
{: .lab-note .info .compact}

- {% include step_label.html %} En una **cuarta terminal** realiza port-forward a OpenCost (deployment).

  ```bash
  cd lab11
  source .env
  kubectl -n "$OC_NS" get deploy | tee outputs/30_oc_deploys.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$OC_NS" port-forward deployment/opencost 9003:9003 9091:9090
  ```
  {% include step_image.html %}

 - {% include step_label.html %} Abre el navegador **Chrome/Edge** en otra pestaña y escribe la siguiente URL para abrir OpenCost.

    - **UI**
  ```bash
  http://localhost:9091
  ```
  {% include step_image.html %} 

- {% include step_label.html %} Regresa a la **primera terminal** y valida salud de OpenCost (API).

  ```bash
  curl -i http://localhost:9003/healthz | tee outputs/31_oc_healthz.txt || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida los targets UP en Prometheus. Abre la URL en otra pestaña del navegador que elegiste.

  > **IMPORTANTE:** Si no aparece: revisa ServiceMonitor, namespace del ServiceMonitor, selectors de Prometheus (Tarea 4) y puertos del Service de OpenCost.
  {: .lab-note .important .compact}

  1) Abre **`http://localhost:9090/targets`**  
  2) Busca **opencost** y confirma **`UP`**

  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Desplegar workloads de prueba (requests/labels)

Crearás tres namespaces y desplegarás workloads con distintos requests/limits y labels para demostrar imputación de costos por entidad (namespace/labels).

> **NOTA (CKAD):** Requests/limits son tema central: scheduling, QoS y troubleshooting cuando un Pod no programa por falta de recursos.
{: .lab-note .info .compact}

#### Tarea 6.1 — Namespaces + etiquetas

- {% include step_label.html %} Continua en la **primera terminal** y crea los namespaces idempotentes.

  ```bash
  kubectl create ns team-a --dry-run=client -o yaml | kubectl apply -f -
  kubectl create ns team-b --dry-run=client -o yaml | kubectl apply -f -
  kubectl create ns team-c --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}

- {% include step_label.html %} Etiqueta los namespaces.

  ```bash
  kubectl label ns team-a team=a env=dev   --overwrite
  kubectl label ns team-b team=b env=stage --overwrite
  kubectl label ns team-c team=c env=prod  --overwrite
  ```
  ```bash
  kubectl get ns --show-labels | egrep "team-a|team-b|team-c" | tee outputs/32_team_ns_labels.txt
  ```
  {% include step_image.html %}

#### Tarea 6.2 — Manifiestos de workloads

- {% include step_label.html %} Crea el archivo `k8s/workloads.yaml`.

  ```bash
  cat > k8s/workloads.yaml <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: web-a
    namespace: team-a
    labels:
      app: web
      team: a
      env: dev
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: web-a
    template:
      metadata:
        labels:
          app: web-a
          team: a
          env: dev
      spec:
        containers:
        - name: nginx
          image: nginx:1.27
          ports:
          - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: api-b
    namespace: team-b
    labels:
      app: api
      team: b
      env: stage
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: api-b
    template:
      metadata:
        labels:
          app: api-b
          team: b
          env: stage
      spec:
        containers:
        - name: api
          image: ghcr.io/stefanprodan/podinfo:6.7.1
          ports:
          - containerPort: 9898
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: batch-c
    namespace: team-c
    labels:
      app: batch
      team: c
      env: prod
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: batch-c
    template:
      metadata:
        labels:
          app: batch-c
          team: c
          env: prod
      spec:
        containers:
        - name: worker
          image: busybox:1.36
          command: ["sh","-c","while true; do echo hello; sleep 5; done"]
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "100m"
              memory: "128Mi"
  EOF
  ```

- {% include step_label.html %} Aplica el manifiesto.

  ```bash
  kubectl apply -f k8s/workloads.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida los rollouts.

  ```bash
  kubectl -n team-a rollout status deploy/web-a | tee outputs/33_rollout_team_a.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n team-b rollout status deploy/api-b | tee outputs/34_rollout_team_b.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n team-c rollout status deploy/batch-c | tee outputs/35_rollout_team_c.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida los recursos y labels.

  ```bash
  kubectl -n team-a get deploy,pods -o wide --show-labels | tee outputs/36_team_a_resources_labels.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n team-b get deploy,pods -o wide --show-labels | tee outputs/37_team_b_resources_labels.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n team-c get deploy,pods -o wide --show-labels | tee outputs/38_team_c_resources_labels.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Troubleshooting rápido si algún Pod no programa.

  > **IMPORTANTE:** Si ves `Insufficient cpu/memory`: reduce requests/replicas o aumenta nodos. **No asumas** tamaño de clúster.
  {: .lab-note .important .compact}

  > **NOTA:** Si no hay errores continua con las tareas y pasos
  {: .lab-note .info .compact}


  ```bash
  kubectl -n team-b get pods | tee outputs/39_team_b_pods.txt
  ```
  ```bash
  kubectl -n team-b describe pod -l app=api-b | sed -n '1,220p' | tee outputs/40_team_b_describe_pod_head.txt
  ```
  ```bash
  kubectl get events -A --sort-by=.lastTimestamp | tail -n 40 | tee outputs/41_events_tail_2.txt
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Consultas de costos (UI, PromQL, API) + evidencias

Usarás OpenCost UI, Prometheus (PromQL) y OpenCost API para obtener costos por namespace y por label, confirmando que los workloads aparecen con imputación de costos.

> **IMPORTANTE:** Algunas métricas/reportes pueden tardar unos minutos en reflejarse. Si tu ventana es muy corta, prueba `window=6h` o `window=24h`.
{: .lab-note .important .compact}

#### Tarea 7.1 — OpenCost UI

- {% include step_label.html %} Abre la UI de OpenCost.

  > **NOTA:** Omite este paso si ya tienes la aplicacion abierta.
  {: .lab-note .info .compact}

  - UI: **`http://localhost:9091`**

#### Tarea 7.2 — PromQL (Prometheus)

- {% include step_label.html %} En Prometheus (http://localhost:9090), prueba queries base.

  > **NOTA:** Si no ves métricas, revisa Targets (`/targets`) y que OpenCost esté `UP`.
  {: .lab-note .info .compact}

  - **Costo horario total (nodos):**
    - **Por Nodo**
    ```promql
    sum by (instance) (node_total_hourly_cost{job="opencost"})
    ```
    - **Por Región**
    ```promql
    sum by (region) (node_total_hourly_cost{job="opencost"})
    ```
    - **Por instance_type**
    ```promql
    sum by (instance_type) (node_total_hourly_cost{job="opencost"})
    ```
    {% include step_image.html %}

  - **Estimación mensual (730h aprox):**
    - **Por Instancia**
    ```promql
    sum by (instance) (node_total_hourly_cost{job="opencost"}) * 730
    ```
    - **Por Región**
    ```promql
    sum by (region) (node_total_hourly_cost{job="opencost"}) * 730
    ```
    {% include step_image.html %}

  - **Costo RAM por hora (nodos):**
    - **Por Instancia**
    ```promql
    sum by (instance) (node_ram_hourly_cost{job="opencost"})
    ```
    - **Por Región**
    ```promql
    sum by (region) (node_ram_hourly_cost{job="opencost"})
    ```
    {% include step_image.html %}

  - **Sanity check de asignación:**
    - **Por Namespace**
    ```promql
    sum by (namespace) (container_cpu_allocation)
    ```
    {% include step_image.html %}

#### Tarea 7.3 — API `/allocation`

- {% include step_label.html %} Ve a la **primera terminal** y consulta por namespace (última hora).

  ```bash
  curl -sG "http://localhost:9003/allocation" \
    -d window=60m \
    -d aggregate=namespace \
    -d resolution=5m \
  | jq -r '
    .data[0]
    | to_entries[]
    | [.key,
      (.value.totalCost // 0),
      (.value.cpuCost // 0),
      (.value.ramCost // 0),
      (.value.cpuCoreUsageAverage // 0),
      (.value.ramByteUsageAverage // 0)
      ]
    | @tsv
  ' | (echo -e "namespace\ttotalCost\tcpuCost\tramCost\tcpuUsageAvg\tramUsageAvg"; cat) \
  | column -t | tee outputs/42_allocation_by_namespace_table.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Consulta por namespace + label `team`.

  ```bash
  curl -sG "http://localhost:9003/allocation" \
    -d window=60m \
    -d aggregate="namespace,label:team" \
    -d resolution=5m \
  | jq -r '
    .data[0]
    | to_entries
    | sort_by(-.value.totalCost)
    | .[]
    | .value as $v
    | [
        ($v.properties.namespace // "-"),
        ($v.properties.namespaceLabels.team // $v.properties.labels.team // "-"),
        ($v.totalCost // 0),
        ($v.cpuCost   // 0),
        ($v.ramCost   // 0)
      ]
    | @tsv
  ' \
  | (echo -e "namespace\tteam\ttotalCost\tcpuCost\tramCost"; cat) \
  | column -t -s $'\t' \
  | tee outputs/43_allocation_by_ns_team_table.txt
  ```
  {% include step_image.html %}

#### Tarea 7.4 — Checklist final (tipo CKAD)

- {% include step_label.html %} Captura evidencia de pods/servicemonitors/salud.

  ```bash
  kubectl -n "$MON_NS" get pods -o wide | tee outputs/45_final_mon_pods.txt
  ```
  ```bash
  kubectl -n "$OC_NS" get pods -o wide | tee outputs/46_final_oc_pods.txt
  ```
  ```bash
  kubectl -n "$MON_NS" get servicemonitors | tee outputs/47_final_servicemonitors.txt
  ```
  ```bash
  curl -sG "http://localhost:9003/allocation" -d window=15m -d aggregate=namespace -d resolution=5m \
    | head -n 120 | tee outputs/49_final_allocation_15m_head.json
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## Tarea 8. Limpieza y control de costos

Eliminarás la carga de prueba y, si este clúster es solo de laboratorio, podrás desinstalar OpenCost y kube-prometheus-stack.

> **IMPORTANTE:** Si tu clúster ya usa Prometheus/Grafana para otras cosas, **NO** desinstales kube-prometheus-stack.
{: .lab-note .important .compact}

### Tarea 8.1

- {% include step_label.html %} Elimina workloads de prueba y namespaces de equipos.

  ```bash
  kubectl delete -f k8s/workloads.yaml --ignore-not-found
  kubectl delete ns team-a team-b team-c --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} Desinstala OpenCost.

  ```bash
  helm -n "$OC_NS" uninstall "$OC_RELEASE" || true
  kubectl delete ns "$OC_NS" --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} Desinstala kube-prometheus-stack.

  ```bash
  helm -n "$MON_NS" uninstall "$MON_RELEASE" || true
  kubectl delete ns "$MON_NS" --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que no queden namespaces de práctica.

  ```bash
  kubectl get ns | egrep "team-a|team-b|team-c|${MON_NS}|${OC_NS}" || echo "OK: namespaces de lab eliminados"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el clúster creado por `eksctl`.

  > **NOTA:** El cluster tardara **9 minutos** aproximadamente en eliminarse.
  {: .lab-note .info .compact}

  ```bash
  eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el cluster se haya eliminado correctamente.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" >/dev/null 2>&1 || echo "Cluster eliminado"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}