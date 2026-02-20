---
layout: lab
title: "Práctica 20: Implementación de políticas de ahorro energético"
permalink: /lab20/lab20/
images_base: /labs/lab20/img
duration: "75 minutos"
objective: |
    Implementar en Amazon EKS un conjunto de políticas prácticas de ahorro energético/costo enfocadas a Kubernetes:
    - (1) Gobernanza de recursos con LimitRange y ResourceQuota.
    - (2) Autoscaling eficiente con HPA (autoscaling/v2 + behavior
    para evitar flapping)
    - (3) Apagado programado de un entorno no productivo usando kube-green (OSS)
    - (4) De forma opcional, escalado programado del Auto Scaling Group (ASG) de un NodeGroup para reducir nodos fuera de horario.
prerequisites:
  - "Amazon EKS accesible con kubectl (contexto correcto)"
  - "Windows + Visual Studio Code + Terminal GitBash"
  - "Herramientas: AWS CLI v2, kubectl, git, curl"
  - "Helm 3 (helm) para instalar kube-green"
  - "(Recomendado) jq para inspección rápida (opcional)"
  - "Permisos Kubernetes: crear Namespace, Deployment, Service, LimitRange, ResourceQuota, HPA, Pods temporales"
  - "(Opcional AWS) autoscaling:* y eks:DescribeNodegroup para scheduled actions del ASG"
introduction:
  - >
    En Kubernetes, el consumo energético/costo suele dispararse por sobreaprovisionamiento: workloads sin requests/limits, entornos dev/stage “siempre encendidos”, autoscaling que sube pero no baja de forma estable, y nodos con baja utilización. En esta práctica aplicarás políticas declarativas (Kubernetes + OSS) para forzar disciplina de recursos (LimitRange/Quota), ajustar el escalado horizontal con HPA behavior (menos rebotes = menos consumo), y apagar automáticamente un namespace no productivo usando kube-green. Como extra opcional, programarás el escalado del ASG de un NodeGroup para apagar nodos fuera de horario.
slug: lab20
lab_number: 20
final_result: >
  Al finalizar tendrás un namespace no productivo “energy-dev” con políticas de gobernanza activas (LimitRange + ResourceQuota),
  un HPA autoscaling/v2 con behavior (scale up rápido y scale down estable), y un apagado programado con kube-green (SleepInfo)
  que escala workloads a 0 en ventanas fuera de horario. Opcionalmente, dejarás scheduled actions en el ASG del NodeGroup para reducir nodos fuera de horario. Todo respaldado con evidencia reproducible (describe/events/top) y troubleshooting estilo CKAD.
notes:
  - "CKAD: práctica núcleo (LimitRange/ResourceQuota/HPA) + troubleshooting con `kubectl describe`, `kubectl get events`, `kubectl top`, `kubectl rollout`, `jsonpath`."
  - "Importante: HPA (CPU/mem) normalmente NO escala a 0. Si quieres ‘sleep’ real a 0 réplicas, lo más simple en dev es pausar/eliminar HPA durante la ventana de sueño, o usar soluciones de scale-to-zero basadas en métricas externas (p.ej., KEDA)."
  - "Programar ASG scheduled actions es potente pero riesgoso si apagas el pool donde viven addons críticos. Úsalo solo con NodeGroups dedicados a dev/lab."
references:
  - text: "Kubernetes - LimitRange"
    url: https://kubernetes.io/docs/concepts/policy/limit-range/
  - text: "Kubernetes - ResourceQuota"
    url: https://kubernetes.io/docs/concepts/policy/resource-quotas/
  - text: "Kubernetes - Horizontal Pod Autoscaling (conceptos)"
    url: https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/
  - text: "Kubernetes - HPA Walkthrough (php-apache + load generator)"
    url: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/
  - text: "metrics-server (releases / manifest oficial)"
    url: https://github.com/kubernetes-sigs/metrics-server/releases
  - text: "kube-green - Installation (Helm)"
    url: https://kube-green.dev/docs/installation/
  - text: "kube-green - Configuration (SleepInfo ejemplos)"
    url: https://kube-green.dev/docs/configuration/
  - text: "AWS - EKS Kubernetes versions (lifecycle)"
    url: https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html
  - text: "AWS CLI - put-scheduled-update-group-action (EC2 Auto Scaling)"
    url: https://docs.aws.amazon.com/cli/latest/reference/autoscaling/put-scheduled-update-group-action.html
prev: /lab19/lab19/
next: /lab21/lab21/
---

---

### Costo (resumen)

- **EKS** cobra por clúster por hora (control plane) + **EC2/EBS** por los nodos.
- **LimitRange/ResourceQuota/HPA** no tienen costo directo.
- **kube-green** no tiene costo directo, pero consume recursos mínimos del clúster.
- **Scheduled actions** del ASG no tienen costo directo (pagas por nodos si están encendidos).

> **IMPORTANTE:** El ahorro real sucede cuando reduces réplicas y/o nodos. Si creaste un clúster solo para el lab,
> elimínalo al final.
{: .lab-note .important .compact}

### Convenciones del laboratorio

- **Todo** se ejecuta en **GitBash** dentro de **VS Code**.
- Guarda evidencia en `outputs/` con `tee` para demostrar resultados.
- Mantén manifiestos por carpeta (políticas / HPA / kube-green) para que sea reproducible.

> **NOTA (CKAD):** Loop recomendado: `apply` → `get/describe` → `events/logs` → corregir YAML → repetir.
{: .lab-note .info .compact}

---

### Tarea 1. Preparación del workspace y baseline

Crearás la carpeta del laboratorio, definirás variables reutilizables y validarás identidad AWS, herramientas y contexto.

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo del curso con un usuario con permisos administrativos.

- {% include step_label.html %} Abre **Visual Studio Code** y la terminal integrada.

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **Git Bash** como terminal.

  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de estar en la carpeta raíz de tus laboratorios (por ejemplo **`labs-eks-ckad`**).

  > **NOTA:** Si vienes de otra práctica, usa `cd ..` hasta volver a la raíz.
  {: .lab-note .info .compact}

- {% include step_label.html %} Crea el directorio del laboratorio y la estructura estándar.

  ```bash
  mkdir -p lab20 && cd lab20
  mkdir -p 00-prereqs 01-eks 02-k8s 03-policies 04-hpa 05-kubegreen 06-asg outputs logs
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que la estructura quedó creada.

  ```bash
  find . -maxdepth 2 -type d | sort | tee outputs/00_dirs.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables base (ajusta región y nombre de clúster si reusas uno existente).

  ```bash
  export AWS_REGION="us-west-2"
  export CLUSTER_NAME="eks-energy-policies-lab"
  export NODEGROUP_NAME="mng-dev"

  export NS="energy-dev"
  export TIME_ZONE="America/Mexico_City"

  export K8S_VERSION="1.33"
  export NODE_TYPE="t3.medium"
  export NODES="2"

  export LAB_ID="$(date +%Y%m%d%H%M%S)"
  ```

- {% include step_label.html %} Captura identidad AWS (evidencia) y guarda el Account ID.

  ```bash
  aws sts get-caller-identity --output json | tee outputs/01_sts_identity.json
  ```
  {% include step_image.html %}
  ```bash
  export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  echo "AWS_ACCOUNT_ID=$AWS_ACCOUNT_ID" | tee outputs/01_account_id.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda variables para reuso (si reinicias terminal).

  ```bash
  cat > outputs/vars.env <<EOF
  AWS_REGION=$AWS_REGION
  CLUSTER_NAME=$CLUSTER_NAME
  NODEGROUP_NAME=$NODEGROUP_NAME
  NS=$NS
  TIME_ZONE=$TIME_ZONE
  K8S_VERSION=$K8S_VERSION
  NODE_TYPE=$NODE_TYPE
  NODES=$NODES
  LAB_ID=$LAB_ID
  AWS_ACCOUNT_ID=$AWS_ACCOUNT_ID
  EOF
  ```
  ```bash
  cat outputs/vars.env | tee outputs/01_vars_echo.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica herramientas (evidencia).

  ```bash
  aws --version | tee outputs/01_aws_version.txt
  kubectl version --client=true | tee outputs/01_kubectl_version.txt
  helm version | tee outputs/01_helm_version.txt
  git --version | tee outputs/01_git_version.txt
  curl --version | head -n 2 | tee outputs/01_curl_version.txt
  jq --version 2>/dev/null | tee outputs/01_jq_version.txt || true
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear un clúster EKS para el laboratorio

Si ya tienes un clúster EKS funcional y `kubectl` apunta al contexto correcto, **puedes omitir esta tarea**.

> **IMPORTANTE:** Crear clúster genera costos. Elimínalo en la si fue creado solo para el lab.
{: .lab-note .important .compact}

#### Tarea 2.1

- {% include step_label.html %} (Creación) Crea el clúster con Managed Node Group (2 nodos).

  > **NOTA:** Si falla por “unsupported Kubernetes version”, vuelve a ejecutar **sin** `--version` o cambia `K8S_VERSION`.
  {: .lab-note .info .compact}

  > **IMPORTANTE:** El cluster tardara aproximadamente **15 minutos** en crearse. Espera el proceso
  {: .lab-note .important .compact}

  ```bash
  eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --version "$K8S_VERSION" \
    --managed \
    --nodegroup-name "$NODEGROUP_NAME" \
    --node-type "$NODE_TYPE" \
    --nodes "$NODES" --nodes-min 1 --nodes-max 3 \
    | tee outputs/02_eksctl_create_cluster.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Configura kubeconfig y valida contexto.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    | tee outputs/02_update_kubeconfig.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl config current-context | tee outputs/02_kube_context.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get nodes -o wide | tee outputs/02_nodes_wide.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl wait --for=condition=Ready node --all --timeout=600s | tee outputs/02_wait_nodes_ready.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Evidencia rápida del estado del clúster desde AWS.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query "cluster.status" --output text | tee outputs/02_cluster_status.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Crear namespace no productivo y desplegar app base (php-apache)

Crearás el namespace `energy-dev` y desplegarás una app base CPU-bound (php-apache) usada para pruebas de HPA.

#### Tarea 3.1 — Namespace + etiquetas

- {% include step_label.html %} Crea el namespace (idempotente) y etiqueta para gobernanza.

  ```bash
  kubectl create ns "$NS" --dry-run=client -o yaml | kubectl apply -f -
  ```
  ```bash
  kubectl label ns "$NS" env=dev sustainability=enabled --overwrite
  ```
  ```bash
  kubectl get ns "$NS" --show-labels | tee outputs/03_ns_labels.txt
  ```
  {% include step_image.html %}

#### Tarea 3.2 — Deployment + Service (php-apache)

- {% include step_label.html %} Crea el manifiesto `02-k8s/01_php-apache.yaml`.

  ```bash
  cat > 02-k8s/01_php-apache.yaml <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: php-apache
    namespace: $NS
    labels:
      app: php-apache
      kube-green.dev/include: "true"
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: php-apache
    template:
      metadata:
        labels:
          app: php-apache
          kube-green.dev/include: "true"
      spec:
        containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports:
          - containerPort: 80
          resources:
            requests:
              cpu: 200m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: php-apache
    namespace: $NS
    labels:
      app: php-apache
  spec:
    selector:
      app: php-apache
    ports:
    - name: http
      port: 80
      targetPort: 80
  EOF
  ```

- {% include step_label.html %} Aplica y verifica rollout.

  ```bash
  kubectl apply -f 02-k8s/01_php-apache.yaml | tee outputs/03_apply_php_apache.txt
  ```
  ```bash
  kubectl -n "$NS" rollout status deploy/php-apache | tee outputs/03_rollout_php_apache.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get deploy,po,svc -o wide | tee outputs/03_app_resources.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida la imagen y los resource requests/limits del Deployment php-apache.

  ```bash
  kubectl -n "$NS" describe deploy php-apache | egrep -n "Image:|Requests:|Limits:" -A2 -B1 | tee outputs/03_deploy_resources_grep.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica la configuración del Service php-apache (tipo, puertos y selector).

  ```bash
  kubectl -n "$NS" get svc php-apache | tee outputs/03_svc_php_apache.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Gobernanza de recursos: LimitRange + ResourceQuota + pruebas negativas

Aplicarás políticas para forzar disciplina: defaults/mínimos/máximos por contenedor (LimitRange) y un “presupuesto” total por namespace (ResourceQuota). Confirmarás enforcement con pruebas negativas.

> **NOTA (CKAD):** LimitRange y ResourceQuota son temas clásicos: valida en `describe` y confirma rechazos en salida + `events`.
{: .lab-note .info .compact}

#### Tarea 4.1 — LimitRange (defaults + min/max)

- {% include step_label.html %} Crea `03-policies/01_limitrange.yaml` y aplícalo.

  ```bash
  cat > 03-policies/01_limitrange.yaml <<EOF
  apiVersion: v1
  kind: LimitRange
  metadata:
    name: energy-defaults
    namespace: $NS
  spec:
    limits:
    - type: Container
      max:
        cpu: "600m"
        memory: "512Mi"
      min:
        cpu: "50m"
        memory: "64Mi"
      default:
        cpu: "300m"
        memory: "256Mi"
      defaultRequest:
        cpu: "150m"
        memory: "128Mi"
  EOF
  ```
  ```bash
  kubectl apply -f 03-policies/01_limitrange.yaml | tee outputs/04_apply_limitrange.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" describe limitrange energy-defaults | tee outputs/04_limitrange_describe.txt
  ```
  {% include step_image.html %}

#### Tarea 4.2 — ResourceQuota (presupuesto del namespace)

- {% include step_label.html %} Crea `03-policies/02_resourcequota.yaml` y aplícalo.

  ```bash
  cat > 03-policies/02_resourcequota.yaml <<EOF
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: energy-quota
    namespace: $NS
  spec:
    hard:
      requests.cpu: "1"
      limits.cpu: "2"
      requests.memory: "1Gi"
      limits.memory: "2Gi"
      pods: "10"
  EOF
  ```
  ```bash
  kubectl apply -f 03-policies/02_resourcequota.yaml | tee outputs/04_apply_quota.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" describe quota energy-quota | tee outputs/04_quota_describe.txt
  ```
  {% include step_image.html %}

#### Tarea 4.3 — Prueba negativa A: Pod “caro” debe ser rechazado (LimitRange)

- {% include step_label.html %} Crea `03-policies/03_bad_pod.yaml` y aplícalo (espera **ERROR**).

  ```bash
  cat > 03-policies/03_bad_pod.yaml <<EOF
  apiVersion: v1
  kind: Pod
  metadata:
    name: bad-pod
    namespace: $NS
  spec:
    containers:
    - name: c
      image: busybox:1.36
      command: ["sh","-c","sleep 3600"]
      resources:
        requests:
          cpu: "800m"
          memory: "700Mi"
  EOF
  ```
  ```bash
  kubectl apply -f 03-policies/03_bad_pod.yaml 2>&1 | tee outputs/04_bad_pod_apply.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -n 30 | tee outputs/04_events_tail.txt
  ```
  {% include step_image.html %}

#### Tarea 4.4 — (Opcional) Defaults inyectados por LimitRange

- {% include step_label.html %} Crea un deployment sin recursos y observa defaults en `describe pod`.

  ```bash
  cat > 03-policies/04_defaults_demo.yaml <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: defaults-demo
    namespace: $NS
    labels:
      app: defaults-demo
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: defaults-demo
    template:
      metadata:
        labels:
          app: defaults-demo
      spec:
        containers:
        - name: nginx
          image: nginx:1.27
          ports:
          - containerPort: 80
  EOF
  ```
  ```bash
  kubectl apply -f 03-policies/04_defaults_demo.yaml | tee outputs/04_apply_defaults_demo.txt
  ```
  ```bash
  kubectl -n "$NS" rollout status deploy/defaults-demo | tee outputs/04_rollout_defaults_demo.txt
  ```
  {% include step_image.html %}
  ```bash
  POD_DEMO="$(kubectl -n "$NS" get pod -l app=defaults-demo -o jsonpath='{.items[0].metadata.name}')"
  ```
  ```bash
  kubectl -n "$NS" describe pod "$POD_DEMO" | egrep -n "Requests:|Limits:" -A6 -B2 | tee outputs/04_defaults_injected.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa LimitRange y ResourceQuota aplicados en el namespace.

  ```bash
  kubectl -n "$NS" get limitrange,quota | tee outputs/04_get_limitrange_quota.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Muestra el resultado/error capturado del intento de aplicar el “bad pod”.

  ```bash
  cat outputs/04_bad_pod_apply.txt | tee outputs/04_bad_pod_apply_echo.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. HPA eficiente (autoscaling/v2 + behavior) y validación con carga

Verificarás (o instalarás) Metrics Server, crearás un HPA `autoscaling/v2` con `behavior` anti-flapping y validarás scale up/down con un generador de carga.

> **IMPORTANTE:** Si `kubectl top` no funciona, tu HPA tampoco tendrá métricas. Primero arregla métricas.
{: .lab-note .important .compact}

#### Tarea 5.1 — Verificar/instalar Metrics Server

- {% include step_label.html %} Verifica si ya existe metrics-server.

  > **IMPORTANTE:** Si metrics-server existe, **NO** hagas la instalacion salta ese paso. `aws eks update-addon --cluster-name "$CLUSTER_NAME" --addon-name metrics-server --region "$AWS_REGION" --resolve-conflicts OVERWRITE`
  {: .lab-note .important .compact}

  ```bash
  kubectl get deployment -n kube-system metrics-server >/dev/null 2>&1 \
    && echo "OK: metrics-server existe" | tee outputs/05_metrics_server_exists.txt \
    || echo "NO: metrics-server no existe" | tee outputs/05_metrics_server_missing.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si no existe, instálalo desde el manifest oficial del proyecto (release latest).

  ```bash
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml \
    | tee outputs/05_apply_metrics_server.txt
  kubectl -n kube-system rollout status deploy/metrics-server | tee outputs/05_rollout_metrics_server.txt
  ```

- {% include step_label.html %} Valida métricas (evidencia).

  ```bash
  kubectl top nodes | tee outputs/05_top_nodes.txt
  ```
  ```bash
  kubectl -n "$NS" top pods | tee outputs/05_top_pods_before.txt || true
  ```
  {% include step_image.html %}

#### Tarea 5.2 — Crear HPA con behavior

- {% include step_label.html %} Crea `04-hpa/01_hpa.yaml` y aplícalo.

  ```bash
  cat > 04-hpa/01_hpa.yaml <<EOF
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: php-apache
    namespace: $NS
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: php-apache
    minReplicas: 1
    maxReplicas: 6
    metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
    behavior:
      scaleUp:
        stabilizationWindowSeconds: 0
        policies:
        - type: Pods
          value: 2
          periodSeconds: 15
        selectPolicy: Max
      scaleDown:
        stabilizationWindowSeconds: 60
        policies:
        - type: Percent
          value: 50
          periodSeconds: 60
        selectPolicy: Min
  EOF
  ```
  ```bash
  kubectl apply -f 04-hpa/01_hpa.yaml | tee outputs/05_apply_hpa.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get hpa | tee outputs/05_hpa_get.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" describe hpa php-apache | tee outputs/05_hpa_describe_initial.txt
  ```
  {% include step_image.html %}

#### Tarea 5.3 — Generar carga y observar scale up/down

- {% include step_label.html %} En tu **terminal principal**, observa el HPA (detén el watch cuando termines).

  ```bash
  kubectl -n "$NS" get hpa php-apache --watch
  ```
  {% include step_image.html %}

- {% include step_label.html %} Inicia el generador de carga **(en una segunda terminal)**.

  ```bash
  cd lab20
  source outputs/vars.env
  kubectl -n "$NS" run load-generator \
    --image=busybox:1.36 --restart=Never -- \
    sh -c 'while true; do wget -q -O- http://php-apache >/dev/null; sleep 0.01; done'
  ```
  {% include step_image.html %} 

- {% include step_label.html %} Verifica que está corriendo

  ```bash
  kubectl -n "$NS" get pod load-generator -w
  ```
  {% include step_image.html %} 

- {% include step_label.html %} **En tu primera terminal** verifica el escalado. Cuanto estes listo corta el proceso con **`CTRL + C`**

  {% include step_image.html %} 

- {% include step_label.html %} **En tu primera terminal** captura evidencia del estado después de carga y durante scale down (puede tardar por estabilización).

  > **NOTA:** Si el HPA intenta escalar y falla por **ResourceQuota**, eso es *gobernanza funcionando*. Verás eventos del tipo “exceeded quota”. Ajusta el quota si quieres permitir una demo más grande.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$NS" get hpa php-apache | tee outputs/05_hpa_after_load.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get deploy php-apache -o wide | tee outputs/05_deploy_php_apache_after_load.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" top pods | tee outputs/05_top_pods_after.txt || true
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -n 30 | tee outputs/05_events_tail.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Apagado programado del entorno dev con kube-green

Instalarás kube-green y crearás un `SleepInfo` para “dormir” el namespace fuera de horario (escala a 0 Deployments/StatefulSets).

> **IMPORTANTE:** Para que el “sleep” mantenga `replicas=0`, **pausa o elimina el HPA**. HPA con CPU normalmente mantiene `minReplicas >= 1`.
{: .lab-note .important .compact}

#### Tarea 6.1 — Instalar kube-green (Helm)

- {% include step_label.html %} Instala kube-green (idempotente con upgrade/install).

  ```bash
  helm repo add jetstack https://charts.jetstack.io | tee outputs/06_helm_repo_add_jetstack.txt
  helm repo update | tee outputs/06_helm_repo_update_jetstack.txt
  ```
  {% include step_image.html %}
  ```bash
  helm upgrade --install cert-manager jetstack/cert-manager \
    --namespace cert-manager --create-namespace \
    --set installCRDs=true \
    | tee outputs/06_helm_install_cert_manager.txt
  ```
  {% include step_image.html %}
  ```bash
  helm repo add kube-green https://kube-green.github.io/helm-charts/
  helm upgrade --install kube-green kube-green/kube-green \
    --namespace kube-green --create-namespace \
    | tee outputs/06_helm_install_kubegreen.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n kube-green rollout status deploy/kube-green-controller-manager \
    | tee outputs/06_rollout_kubegreen.txt
  ```
  ```bash
  kubectl -n kube-green get pods -o wide | tee outputs/06_kubegreen_pods.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get crd | grep -i sleepinfo | tee outputs/06_crd_sleepinfo.txt
  ```
  {% include step_image.html %}

#### Tarea 6.2 — Pausar/eliminar HPA (para permitir sleep a 0)

- {% include step_label.html %} Elimina el HPA (lo puedes recrear reaplicando `04-hpa/01_hpa.yaml`).

  ```bash
  kubectl -n "$NS" delete hpa php-apache --ignore-not-found | tee outputs/06_delete_hpa.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get hpa 2>/dev/null | tee outputs/06_hpa_list_after_delete.txt || true
  ```
  {% include step_image.html %}

#### Tarea 6.3 — Crear SleepInfo (ventana corta para demo)

- {% include step_label.html %} Calcula una ventana corta de prueba: sleep en ~2 min y wake en ~6 min.

  > **NOTA:** Si tu `date -d` no funciona en GitBash, define `SLEEP_AT` y `WAKE_AT` manualmente (HH:MM).
  {: .lab-note .info .compact}

  ```bash
  NOW="$(date +'%H:%M')"
  SLEEP_AT="$(date -d '+2 minutes' +'%H:%M' 2>/dev/null || echo '20:30')"
  WAKE_AT="$(date -d '+6 minutes' +'%H:%M' 2>/dev/null || echo '20:35')"
  ```
  ```bash
  echo "NOW=$NOW SLEEP_AT=$SLEEP_AT WAKE_AT=$WAKE_AT TIME_ZONE=$TIME_ZONE" | tee outputs/06_times.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea `05-kubegreen/01_sleepinfo.yaml` y aplícalo.

  ```bash
  cat > 05-kubegreen/01_sleepinfo.yaml <<EOF
  apiVersion: kube-green.com/v1alpha1
  kind: SleepInfo
  metadata:
    name: dev-working-hours
    namespace: $NS
  spec:
    weekdays: "*"
    sleepAt: "$SLEEP_AT"
    wakeUpAt: "$WAKE_AT"
    timeZone: "$TIME_ZONE"
    suspendCronJobs: true
    includeRef:
      - matchLabels:
          kube-green.dev/include: "true"
  EOF
  ```
  ```bash
  kubectl apply -f 05-kubegreen/01_sleepinfo.yaml | tee outputs/06_apply_sleepinfo.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" describe sleepinfo dev-working-hours | tee outputs/06_sleepinfo_describe.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get sleepinfo | tee outputs/06_sleepinfo_get.txt
  ```
  {% include step_image.html %}

#### Tarea 6.4 — Evidenciar sleep (replicas→0) y wake up (replicas→1)

- {% include step_label.html %} Observa réplicas del deployment (antes).

  ```bash
  kubectl -n "$NS" get deploy php-apache -o wide | tee outputs/06_deploy_before_sleep.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Inicia watch del deployment y espera a `sleepAt` (debe bajar a 0).

  ```bash
  kubectl -n "$NS" get deploy php-apache -w
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si no cambia, revisa logs del controlador y eventos.

  ```bash
  kubectl -n kube-green logs deploy/kube-green --tail=160 | tee outputs/06_kubegreen_logs_tail.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -n 40 | tee outputs/06_events_tail.txt
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Escalado programado de nodos con ASG scheduled actions

Cuando “duermes” el namespace, aún puedes tener nodos encendidos. Si tu clúster tiene NodeGroups separados (p.ej., uno dedicado a dev), puedes programar que el ASG reduzca capacidad fuera de horario.

> **IMPORTANTE:** Haz esto solo si tienes un NodeGroup que puedas apagar sin tumbar componentes críticos.
{: .lab-note .important .compact}

#### Tarea 7.1 — Identificar el ASG del Managed NodeGroup

- {% include step_label.html %} Obtén el ASG asociado al NodeGroup.

  ```bash
  export ASG_NAME="$(aws eks describe-nodegroup \
    --region "$AWS_REGION" \
    --cluster-name "$CLUSTER_NAME" \
    --nodegroup-name "$NODEGROUP_NAME" \
    --query 'nodegroup.resources.autoScalingGroups[0].name' \
    --output text)"
  ```
  ```bash
  echo "ASG_NAME=$ASG_NAME" | tee outputs/07_asg_name.txt
  ```
  {% include step_image.html %}

#### Tarea 7.2 — Crear scheduled actions (bajar de noche / subir por la mañana)

  > **IMPORTANTE:** Esta tarea es un ejemplo dado los horarios para la escalabilidad. no se pueden esperar a las fechas correspondientes.
  {: .lab-note .important .compact}

- {% include step_label.html %} Programa acciones (ejemplo: bajar 20:30 y subir 07:30, horario CDMX).

  ```bash
  aws autoscaling put-scheduled-update-group-action \
    --region "$AWS_REGION" \
    --auto-scaling-group-name "$ASG_NAME" \
    --scheduled-action-name "dev-scale-down-night" \
    --recurrence "30 20 * * *" \
    --min-size 0 --max-size 0 --desired-capacity 0 \
    --time-zone "$TIME_ZONE" \
    | tee outputs/07_put_scheduled_scale_down.txt
  ```
  ```bash
  aws autoscaling put-scheduled-update-group-action \
    --region "$AWS_REGION" \
    --auto-scaling-group-name "$ASG_NAME" \
    --scheduled-action-name "dev-scale-up-morning" \
    --recurrence "30 7 * * *" \
    --min-size 1 --max-size 3 --desired-capacity 1 \
    --time-zone "$TIME_ZONE" \
    | tee outputs/07_put_scheduled_scale_up.txt
  ```

- {% include step_label.html %} Verifica acciones programadas y capacidad actual del ASG (evidencia).

  ```bash
  aws autoscaling describe-scheduled-actions \
    --region "$AWS_REGION" \
    --auto-scaling-group-name "$ASG_NAME" \
    --query 'ScheduledUpdateGroupActions[].{Action:ScheduledActionName,Recurrence:Recurrence,Min:MinSize,Desired:DesiredCapacity,Max:MaxSize,TimeZone:TimeZone}' \
    --output table | tee outputs/07_scheduled_actions_compact.txt
  ```
  {% include step_image.html %}
  ```bash
  aws autoscaling describe-auto-scaling-groups \
    --region "$AWS_REGION" \
    --auto-scaling-group-names "$ASG_NAME" \
    --query 'AutoScalingGroups[0].{Desired:DesiredCapacity,Min:MinSize,Max:MaxSize}' \
    --output table | tee outputs/07_asg_capacity.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Checklist final

Consolidarás evidencia final del estado del entorno (sin limpiar todavía). Esto es útil para “entregar” resultados antes de borrar recursos.

#### Tarea 8.1 

- {% include step_label.html %} Estado final de recursos, políticas y eventos.

  ```bash
  kubectl -n "$NS" get all -o wide | tee outputs/08_get_all_ns.txt
  ```
  ```bash
  kubectl -n "$NS" get limitrange,quota,hpa,sleepinfo 2>/dev/null | tee outputs/08_policy_objects.txt || true
  ```
  ```bash
  kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -n 60 | tee outputs/08_events_tail.txt
  ```
  ```bash
  kubectl top nodes | tee outputs/08_top_nodes.txt || true
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r8 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r8 %}

---

### Tarea 9. Limpieza del laboratorio

Eliminarás el namespace del lab y (opcionalmente) kube-green si fue solo para esta práctica.

> **IMPORTANTE:** Si programaste scheduled actions del ASG (Tarea 7), considera eliminarlas también (Tarea 10).
{: .lab-note .important .compact}

#### Tarea 9.1 — Eliminar namespace del lab

- {% include step_label.html %} Elimina el namespace del lab y verifica.

  ```bash
  kubectl delete ns "$NS" | tee outputs/09_delete_ns.txt
  ```
  ```bash
  kubectl get ns | grep -w "$NS" && echo "Namespace aún existe (espera terminación)" \
    || echo "OK: namespace eliminado" | tee outputs/09_verify_ns_deleted.txt
  ```
  {% include step_image.html %}

#### Tarea 9.2 — Desinstalar kube-green

- {% include step_label.html %} Si kube-green se instaló solo para el lab, desinstálalo y elimina su namespace.

  ```bash
  helm uninstall kube-green -n kube-green | tee outputs/09_helm_uninstall_kubegreen.txt || true
  ```
  {% include step_image.html %}
  ```bash
  kubectl delete ns kube-green --ignore-not-found | tee outputs/09_delete_ns_kubegreen.txt || true
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r8 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r8 %}

---

### Tarea 10. Limpieza AWS (scheduled actions / clúster)

Si aplicaste Tarea 7, elimina las scheduled actions. Si creaste el clúster solo para esta práctica, elimínalo.

#### Tarea 10.1 — Eliminar scheduled actions del ASG

- {% include step_label.html %} Borra las acciones programadas (si existen).

  ```bash
  # Requiere ASG_NAME. Si no lo tienes, re-ejecuta Tarea 7.1.
  aws autoscaling delete-scheduled-action \
    --region "$AWS_REGION" \
    --auto-scaling-group-name "$ASG_NAME" \
    --scheduled-action-name "dev-scale-down-night" \
    | tee outputs/10_delete_scheduled_down.txt || true
  ```
  {% include step_image.html %}
  ```bash
  aws autoscaling delete-scheduled-action \
    --region "$AWS_REGION" \
    --auto-scaling-group-name "$ASG_NAME" \
    --scheduled-action-name "dev-scale-up-morning" \
    | tee outputs/10_delete_scheduled_up.txt || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que ya no estén listadas.

  ```bash
  aws autoscaling describe-scheduled-actions \
    --region "$AWS_REGION" \
    --auto-scaling-group-name "$ASG_NAME" \
    --query 'ScheduledUpdateGroupActions[].{Action:ScheduledActionName,Recurrence:Recurrence,Min:MinSize,Desired:DesiredCapacity,Max:MaxSize,TimeZone:TimeZone}' \
    --output table | tee outputs/07_scheduled_actions_compact.txt
  ```
  {% include step_image.html %}

#### Tarea 10.2 — Elimina el clúster EKS

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
{% capture r1 %}{{ results[9] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}