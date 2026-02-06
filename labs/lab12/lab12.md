---
layout: lab
title: "Práctica 12: Configuración de un Service Mesh con Istio (canary release)"
permalink: /lab12/lab12/
images_base: /labs/lab12/img
duration: "60 minutos"
objective:
  - "Instalar Istio (modo sidecar) en un clúster Amazon EKS, habilitar inyección automática, desplegar 2 versiones (v1/v2) de un microservicio y ejecutar un canary release controlando el porcentaje de tráfico con VirtualService y DestinationRule; validando con pruebas de tráfico y troubleshooting tipo CKAD (describe/events/logs/rollout + istioctl)."
prerequisites:
  - "Amazon EKS accesible con kubectl"
  - "Windows + Visual Studio Code + Terminal GitBash"
  - "Herramientas: AWS CLI v2, kubectl, eksctl, git, curl"
  - "Permisos Kubernetes: crear Namespace, CRDs, webhooks, Deployments, Services"
  - "Acceso a Internet para descargar Istio (istioctl + samples)"
  - "(Opcional) unzip o PowerShell (Expand-Archive) para extraer ZIP en Windows"
introduction:
  - "Istio agrega una capa de red para microservicios mediante sidecars Envoy. Con DestinationRule defines subsets (v1/v2) basados en labels y con VirtualService aplicas reglas de enrutamiento (por pesos) para hacer canary releases progresivos (100/0 → 90/10 → 50/50 → 0/100) sin modificar el Service de Kubernetes. En este laboratorio realizarás el flujo completo y lo validarás con tráfico real y evidencia (logs/events/estado de recursos) al estilo CKAD."
slug: lab12
lab_number: 12
final_result: >
  Al finalizar tendrás Istio instalado en EKS, un microservicio helloworld con 2 versiones (v1/v2) dentro del mesh, y un canary release controlado por porcentajes usando DestinationRule + VirtualService; validado con pruebas desde un Pod curl dentro del clúster y diagnóstico con kubectl (describe/events/logs) + istioctl (analyze/proxy-status).
notes:
  - "CKAD: práctica altamente relevante (namespaces, labels/selectors, Deployments/Services, kubectl exec/logs/describe/events, troubleshooting por capas)."
  - "Este laboratorio NO expone LoadBalancers: todo se prueba dentro del clúster para evitar costos extra y reducir variables de red."
  - "Regla operativa: si un Pod NO tiene el contenedor istio-proxy, ese Pod NO está siendo controlado por las reglas de Istio."
references:
  - text: "Istio - Download the Istio release (istioctl + releases)"
    url: https://istio.io/latest/docs/setup/additional-setup/download-istio-release/
  - text: "Istio - Sidecar Injection (auto-injection logic y labels)"
    url: https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/
  - text: "Istio - DestinationRule (subsets y políticas por servicio)"
    url: https://istio.io/latest/docs/reference/config/networking/destination-rule/
  - text: "Istio - VirtualService (traffic shifting por weights)"
    url: https://istio.io/latest/docs/reference/config/networking/virtual-service/
  - text: "Istio - Release announcements (referencia de versión actual)"
    url: https://istio.io/latest/news/releases/
  - text: "AWS - Amazon EKS pricing (costo del control plane)"
    url: https://aws.amazon.com/eks/pricing/
prev: /lab11/lab11/
next: /lab13/lab13/
---

## Costo (resumen)

- **EKS** cobra por clúster por hora (estándar vs **Extended Support**). Revisa la página de pricing antes de iniciar la práctica.
- **Istio** no se cobra como servicio separado, pero consume recursos de cómputo en el clúster (CPU/RAM).

> **IMPORTANTE:** Para este laboratorio no se crean LoadBalancers; todas las pruebas son internas (Pod → Service) y por `kubectl exec`, reduciendo costos y variabilidad.
{: .lab-note .important .compact}

---

### Tarea 1. Preparación del workspace y baseline del clúster

Crearás la carpeta del laboratorio, definirás variables reutilizables y validarás contexto/permisos antes de instalar Istio.

> **NOTA (CKAD):** Muchísimos fallos se explican con: contexto incorrecto, permisos, eventos y “por qué mi Pod no programa”. Practica `kubectl describe`, `kubectl get events` y `kubectl rollout status`.
{: .lab-note .info .compact}

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo del curso con un usuario con permisos administrativos.

- {% include step_label.html %} Abre **Visual Studio Code**.

- {% include step_label.html %} Abre la terminal integrada en VS Code.

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **Git Bash** como terminal.

  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de estar en la carpeta raíz de tus laboratorios (por ejemplo **`labs-eks-ckad`**).

  > **NOTA:** Si vienes de otra práctica, usa `cd ..` hasta volver a la raíz.
  {: .lab-note .info .compact}

- {% include step_label.html %} Crea el directorio del laboratorio y la estructura estándar.

  ```bash
  mkdir -p lab12 && cd lab12
  mkdir -p 00-prereqs 01-istio 02-app 03-traffic 04-validate outputs logs
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

  # Namespace del laboratorio (dentro del mesh)
  export NS="canary"

  # Ajusta a una versión existente. Consulta:
  # https://istio.io/latest/news/releases/
  export ISTIO_VERSION="1.28.2"
  EOF
  ```
  ```bash
  source .env
  ```

- {% include step_label.html %} Verifica CLIs (evidencia de herramientas disponibles).

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
  ```bash
  git --version
  ```
  ```bash
  curl --version
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica tu identidad en AWS (si falla, corrige credenciales antes de continuar).

  > **NOTA:** Verifica que no existan errores.
  {: .lab-note .info .compact}

  ```bash
  aws sts get-caller-identity | tee outputs/01_aws_identity.json
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

- {% include step_label.html %} Define variables del clúster (usa un nombre único).

  ```bash
  export AWS_REGION="${AWS_REGION}"
  export CLUSTER_NAME="eks-istio-lab"
  export NODEGROUP_NAME="mng-1"
  export K8S_VERSION="1.33"
  ```

- {% include step_label.html %} Crea el clúster con un Managed Node Group.

  > **IMPORTANTE:** El cluster tardara aproximadamente **15 minutos** en crearse. Espera el proceso
  {: .lab-note .important .compact}

  ```bash
  eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --version "$K8S_VERSION" \
    --managed \
    --nodegroup-name "$NODEGROUP_NAME" \
    --node-type t3.medium \
    --nodes 2 --nodes-min 2 --nodes-max 3
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el clúster esté `ACTIVE` y que existan NodeGroups.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"     --query "cluster.status" --output text | tee outputs/02_cluster_status.txt
  ```
  {% include step_image.html %}
  ```bash
  aws eks list-nodegroups --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION" --output table | tee outputs/02_nodegroups_table.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asocia OIDC provider (recomendado para controladores/IRSA).

  ```bash
  eksctl utils associate-iam-oidc-provider --cluster "$CLUSTER_NAME" --region "$AWS_REGION" --approve
  ```
  {% include step_image.html %}
  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.identity.oidc.issuer" --output text | tee outputs/02_oidc_issuer.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Validación de los permisos (RBAC) necesarios para instalar Istio (debe responder `yes`).

  ```bash
  kubectl auth can-i create namespace | tee outputs/01_rbac_ns.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl auth can-i create customresourcedefinitions.apiextensions.k8s.io | tee outputs/01_rbac_crd.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl auth can-i create mutatingwebhookconfigurations.admissionregistration.k8s.io | tee outputs/01_rbac_mwh.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl auth can-i create validatingwebhookconfigurations.admissionregistration.k8s.io | tee outputs/01_rbac_vwh.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Baseline: eventos recientes del clúster (detecta problemas previos).

  ```bash
  kubectl get events -A --sort-by=.lastTimestamp | tail -n 50 | tee outputs/01_events_tail.txt
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Descargar e instalar Istio

Descargarás Istio (incluye `istioctl` y `samples/`), lo agregarás al PATH y lo instalarás con perfil **minimal** (control plane sin gateways). También habilitarás access logs del proxy para observar tráfico.

> **IMPORTANTE:** Esta práctica instala componentes cluster-scope (CRDs + webhooks). Si hay errores aquí, la inyección de sidecars fallará después.
{: .lab-note .important .compact}

#### Tarea 3.1 — Descargar Istio

- {% include step_label.html %} Descarga Istio dentro de `01-istio/`.

  ```bash
  cd 01-istio
  ```
  ```bash
  export ISTIO_VERSION="${ISTIO_VERSION}"
  OS="$(uname -s | tr '[:upper:]' '[:lower:]')"
  echo "OS=$OS ISTIO_VERSION=$ISTIO_VERSION" | tee ../outputs/03_os_istio.txt
  ```
  ```bash
  if [[ "$OS" == mingw* || "$OS" == msys* || "$OS" == cygwin* ]]; then
    curl -fsSL -o "istio-${ISTIO_VERSION}-win.zip" "https://github.com/istio/istio/releases/download/${ISTIO_VERSION}/istio-${ISTIO_VERSION}-win.zip"

    if command -v unzip >/dev/null 2>&1; then
      unzip -q "istio-${ISTIO_VERSION}-win.zip"
    else
      powershell.exe -NoProfile -Command "Expand-Archive -Force 'istio-${ISTIO_VERSION}-win.zip' '.'"
    fi
  else
    curl -fsSL https://istio.io/downloadIstio | ISTIO_VERSION="$ISTIO_VERSION" sh -
  fi
  ```
  {% include step_image.html %}
  ```bash
  ls -1 | grep -E '^istio-' | tail -n 5 | tee ../outputs/03_istio_dirs.txt
  ```

- {% include step_label.html %} Define `ISTIO_HOME` apuntando a la carpeta extraída y valida `istioctl` local.

  ```bash
  export ISTIO_HOME="$PWD/istio-${ISTIO_VERSION}"
  echo "ISTIO_HOME=$ISTIO_HOME" | tee ../outputs/03_istio_home.txt
  ```
  {% include step_image.html %}
  ```bash
  export PATH="$ISTIO_HOME/bin:$PATH"
  which istioctl | tee ../outputs/03_which_istioctl.txt
  ```
  {% include step_image.html %}
  ```bash
  istioctl version --remote=false | tee ../outputs/03_istioctl_version_local.txt
  ```
  {% include step_image.html %}

#### Tarea 3.2 — Instalar Istio

- {% include step_label.html %} Instala Istio con perfil `minimal` y habilita access logs del proxy (se verán en `kubectl logs ... -c istio-proxy`).

  ```bash
  istioctl install --set profile=minimal --set meshConfig.accessLogFile=/dev/stdout -y | tee ../outputs/03_istio_install.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica el control plane en `istio-system` (debe existir `istiod` Ready).

  ```bash
  kubectl get ns istio-system -o wide | tee ../outputs/03_ns_istio_system.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n istio-system get pods -o wide | tee ../outputs/03_istio_pods.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n istio-system wait --for=condition=Ready pod -l app=istiod --timeout=600s
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n istio-system get pods -o wide | tee ../outputs/03_istio_pods_ready.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma CRDs y webhooks instalados (evidencia).

  ```bash
  kubectl get crd | egrep "istio\.io|networking\.istio\.io|security\.istio\.io" | head -n 60 | tee ../outputs/03_istio_crds_head.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get mutatingwebhookconfigurations | grep -i istio | tee ../outputs/03_mutating_webhooks.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get validatingwebhookconfigurations | grep -i istio | tee ../outputs/03_validating_webhooks.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida instalación desde el punto de vista del control plane y del análisis estático.

  > **IMPORTANTE:** Es normal que aparezca el mensaje **The namespace is not enabled for Istio injection.** se configurara en la siguiente tarea.
  {: .lab-note .important .compact}

  ```bash
  istioctl version --remote=true | tee ../outputs/03_istioctl_version_remote.txt
  ```
  {% include step_image.html %}
  ```bash
  istioctl analyze -A | tee ../outputs/03_istioctl_analyze_all.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n istio-system logs deploy/istiod --tail=120 | tee ../outputs/03_istiod_logs_tail.txt
  ```

- {% include step_label.html %} Vuelve a la carpeta raíz del lab.

  ```bash
  cd ..
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Desplegar microservicio (v1/v2) y cliente curl dentro del mesh

Crearás un namespace con inyección automática, desplegarás `helloworld` en dos versiones (v1/v2) y un Deployment `curl` para generar tráfico interno.

> **NOTA (CKAD):** Aquí practicas directo: namespaces, labels/selectors, Deployments/Services, `kubectl exec`, `rollout status`, `describe` y eventos.
{: .lab-note .info .compact}

#### Tarea 4.1 — Namespace + auto-injection

- {% include step_label.html %} Crea el namespace del laboratorio y habilita inyección automática de sidecar.

  ```bash
  kubectl create namespace "$NS" --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}
  ```bash
  kubectl label namespace "$NS" istio-injection=enabled --overwrite
  ```
  ```bash
  kubectl get ns "$NS" -L istio-injection | tee outputs/04_ns_labeled.txt
  ```
  {% include step_image.html %}

#### Tarea 4.2 — Manifiestos (autocontenidos)

- {% include step_label.html %} Copia los manifests `samples` a tu carpeta `02-app/` (para que el lab sea autocontenido).

  ```bash
  test -d "$ISTIO_HOME/samples" || (echo "ERROR: ISTIO_HOME/samples no existe. Revisa Tarea 3." && exit 1)
  ```
  ```bash
  cp "$ISTIO_HOME/samples/helloworld/helloworld.yaml" 02-app/helloworld.yaml
  cp "$ISTIO_HOME/samples/curl/curl.yaml" 02-app/curl.yaml
  ls -la 02-app | tee outputs/04_02app_ls.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Aplica el **Service** de `helloworld` y luego las versiones v1 y v2 (por labels del YAML).

  ```bash
  kubectl apply -n "$NS" -f 02-app/helloworld.yaml -l service=helloworld
  kubectl apply -n "$NS" -f 02-app/helloworld.yaml -l version=v1
  kubectl apply -n "$NS" -f 02-app/helloworld.yaml -l version=v2
  ```
  {% include step_image.html %}

- {% include step_label.html %} Aplica el cliente `curl` dentro del namespace (también recibirá sidecar por auto-injection).

  ```bash
  kubectl apply -n "$NS" -f 02-app/curl.yaml
  ```
  {% include step_image.html %}

#### Tarea 4.3 — Esperar Ready + pruebas base

- {% include step_label.html %} Revisa recursos y espera rollouts.

  ```bash
  kubectl -n "$NS" get deploy,svc,pods -o wide | tee outputs/04_resources_initial.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" rollout status deploy/helloworld-v1 | tee outputs/04_rollout_v1.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" rollout status deploy/helloworld-v2 | tee outputs/04_rollout_v2.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" rollout status deploy/curl | tee outputs/04_rollout_curl.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica inyección de sidecar: cada Pod debe tener `istio-proxy`.

  ```bash
  kubectl get pod -n "$NS" -l app=helloworld -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.containers[*].name}{"\n"}{end}' | tee outputs/04_helloworld_containers.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get pod -n "$NS" -l app=curl -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.containers[*].name}{"\n"}{end}' | tee outputs/04_curl_containers.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba base desde curl hacia el Service (debe responder v1 o v2).

  ```bash
  kubectl exec -n "$NS" deploy/curl -c curl -- curl -sS helloworld:5000/hello | tee outputs/04_single_request.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Evidencia de endpoints del Service (debe incluir Pods v1 y v2).

  ```bash
  kubectl -n "$NS" get svc helloworld -o wide | tee outputs/04_svc_helloworld.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get endpoints helloworld -o wide | tee outputs/04_endpoints_helloworld.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get pods -l app=helloworld --show-labels | tee outputs/04_pods_labels.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Canary release con DestinationRule + VirtualService (weights)

Definirás subsets por versión (v1/v2) y controlarás el porcentaje de tráfico con enrutamiento por pesos.

> **NOTA (CKAD):** Este patrón depende 100% de **labels consistentes**. Si el selector/labels no cuadran, el tráfico no se comportará como esperas.
{: .lab-note .info .compact}

#### Tarea 5.1 — DestinationRule (subsets)

- {% include step_label.html %} Crea y aplica `DestinationRule` para definir subsets `v1` y `v2` (mapeados a labels reales).

  ```bash
  cat > 03-traffic/destinationrule-helloworld.yaml <<EOF
  apiVersion: networking.istio.io/v1
  kind: DestinationRule
  metadata:
    name: helloworld
    namespace: ${NS}
  spec:
    host: helloworld
    subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
  EOF
  ```
  ```bash
  kubectl apply -f 03-traffic/destinationrule-helloworld.yaml
  ```
  {% include step_image.html %}

#### Tarea 5.2 — VirtualServices por etapa (100/0 → 90/10 → 50/50 → 0/100)

- {% include step_label.html %} Crea y aplica `VirtualService` con **100% v1 / 0% v2**.

  ```bash
  cat > 03-traffic/virtualservice-100v1.yaml <<EOF
  apiVersion: networking.istio.io/v1
  kind: VirtualService
  metadata:
    name: helloworld
    namespace: ${NS}
  spec:
    hosts:
    - helloworld
    http:
    - route:
      - destination:
          host: helloworld
          subset: v1
        weight: 100
      - destination:
          host: helloworld
          subset: v2
        weight: 0
  EOF
  ```
  ```bash
  kubectl apply -f 03-traffic/virtualservice-100v1.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea y aplica **90/10**.

  ```bash
  cat > 03-traffic/virtualservice-90v1-10v2.yaml <<EOF
  apiVersion: networking.istio.io/v1
  kind: VirtualService
  metadata:
    name: helloworld
    namespace: ${NS}
  spec:
    hosts:
    - helloworld
    http:
    - route:
      - destination:
          host: helloworld
          subset: v1
        weight: 90
      - destination:
          host: helloworld
          subset: v2
        weight: 10
  EOF
  ```
  ```bash
  kubectl apply -f 03-traffic/virtualservice-90v1-10v2.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea y aplica **50/50**.

  ```bash
  cat > 03-traffic/virtualservice-50v1-50v2.yaml <<EOF
  apiVersion: networking.istio.io/v1
  kind: VirtualService
  metadata:
    name: helloworld
    namespace: ${NS}
  spec:
    hosts:
    - helloworld
    http:
    - route:
      - destination:
          host: helloworld
          subset: v1
        weight: 50
      - destination:
          host: helloworld
          subset: v2
        weight: 50
  EOF
  ```
  ```bash
  kubectl apply -f 03-traffic/virtualservice-50v1-50v2.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea y aplica **0/100 (todo v2)**.

  ```bash
  cat > 03-traffic/virtualservice-0v1-100v2.yaml <<EOF
  apiVersion: networking.istio.io/v1
  kind: VirtualService
  metadata:
    name: helloworld
    namespace: ${NS}
  spec:
    hosts:
    - helloworld
    http:
    - route:
      - destination:
          host: helloworld
          subset: v1
        weight: 0
      - destination:
          host: helloworld
          subset: v2
        weight: 100
  EOF
  ```
  ```bash
  kubectl apply -f 03-traffic/virtualservice-0v1-100v2.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que existan `DestinationRule` y `VirtualService` y captura evidencia.

  ```bash
  kubectl get destinationrule,virtualservice -n "$NS" -o wide | tee outputs/05_istio_resources.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl describe destinationrule helloworld -n "$NS" | sed -n '1,220p' | tee outputs/05_describe_destinationrule.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl describe virtualservice helloworld -n "$NS" | sed -n '1,260p' | tee outputs/05_describe_virtualservice.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Pruebas de tráfico y troubleshooting (estilo CKAD)

Generarás tráfico repetido y verificarás que el porcentaje cambia. Si el tráfico no cambia, ejecutarás un checklist de diagnóstico (eventos, sidecar, config y estado del proxy).

> **Nota:** El reparto por pesos es probabilístico. Con pocas peticiones, el conteo puede variar. Usa más requests para ver tendencias.
{: .lab-note .info .compact}

#### Tarea 6.1 — Pruebas de tráfico con conteo por versión

- {% include step_label.html %} Prueba **100% v1** (aplica VS 100v1, genera requests y cuenta).

  ```bash
  kubectl apply -f 03-traffic/virtualservice-100v1.yaml
  ```
  ```bash
  kubectl exec -n "$NS" deploy/curl -c curl -- sh -lc     'for i in $(seq 1 80); do curl -sS helloworld:5000/hello; done' | tee outputs/06_100v1_raw.txt
  ```
  {% include step_image.html %}
  ```bash
  cat outputs/06_100v1_raw.txt | grep -oE 'v1|v2' | sort | uniq -c | tee outputs/06_100v1_count.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba **90/10** (más requests para observar mejor el reparto).

  ```bash
  kubectl apply -f 03-traffic/virtualservice-90v1-10v2.yaml
  ```
  ```bash
  kubectl exec -n "$NS" deploy/curl -c curl -- sh -lc 'for i in $(seq 1 300); do curl -sS helloworld:5000/hello; done' | tee outputs/06_90_10_raw.txt
  ```
  {% include step_image.html %}
  ```bash
  cat outputs/06_90_10_raw.txt | grep -oE 'v1|v2' | sort | uniq -c | tee outputs/06_90_10_count.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba **50/50**.

  > **IMPORTANTE:** Recordar que `VirtualService` es un peso estadístico, no una garantía exacta en una muestra finita.
  {: .lab-note .important .compact}

  ```bash
  kubectl apply -f 03-traffic/virtualservice-50v1-50v2.yaml
  ```
  ```bash
  kubectl exec -n "$NS" deploy/curl -c curl -- sh -lc 'for i in $(seq 1 300); do curl -sS helloworld:5000/hello; done' | tee outputs/06_50_50_raw.txt
  ```
  {% include step_image.html %}
  ```bash
  cat outputs/06_50_50_raw.txt | grep -oE 'v1|v2' | sort | uniq -c | tee outputs/06_50_50_count.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba **0/100 (todo v2)**.

  ```bash
  kubectl apply -f 03-traffic/virtualservice-0v1-100v2.yaml
  ```
  ```bash
  kubectl exec -n "$NS" deploy/curl -c curl -- sh -lc 'for i in $(seq 1 80); do curl -sS helloworld:5000/hello; done' | tee outputs/06_0_100_raw.txt
  ```
  {% include step_image.html %}
  ```bash
  cat outputs/06_0_100_raw.txt | grep -oE 'v1|v2' | sort | uniq -c | tee outputs/06_0_100_count.txt
  ```
  {% include step_image.html %}

#### Tarea 6.2 — Troubleshooting si “no obedece”

- {% include step_label.html %} Revisa eventos del namespace (errores de webhook, crashloop, image pull).

  ```bash
  kubectl get events -n "$NS" --sort-by=.lastTimestamp | tail -n 60 | tee outputs/06_events_tail.txt
  ```

- {% include step_label.html %} Confirma que los Pods objetivo tienen sidecar `istio-proxy` (si falta, no hay control de Istio).

  ```bash
  kubectl get pod -n "$NS" -l app=helloworld -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.containers[*].name}{"\n"}{end}' | tee outputs/06_sidecar_check.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que los recursos Istio existen y están en el namespace correcto.

  ```bash
  kubectl get destinationrule,virtualservice -n "$NS" -o wide | tee outputs/06_get_istio_resources.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get pod -n "$NS" -l app=helloworld --show-labels | tee outputs/06_pods_labels_again.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida que `istiod` está listo (sin control plane sano, no hay config efectiva).

  ```bash
  kubectl -n istio-system get pods -o wide | tee outputs/06_istiod_status.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n istio-system logs deploy/istiod --tail=120 | tee outputs/06_istiod_logs_tail.txt
  ```

- {% include step_label.html %} Diagnóstico con `istioctl` sobre el namespace.

  > **IMPORTANTE:** Puedes ignorar los mensajes de error. Son mensajes del analizador interno de istioctl cuando intenta “traducir” (convertir) recursos de configuración de malla (MeshConfig/MeshNetworks) de una API vieja (core/v1alpha1) y no encuentra el traductor en esa versión/binario.
  {: .lab-note .important .compact}

  ```bash
  istioctl analyze -n "$NS" | tee outputs/06_istioctl_analyze_ns.txt
  ```
  ```bash
  istioctl proxy-status | tee outputs/06_istioctl_proxy_status.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Logs del proxy (access logs) para evidencia de tráfico (toma un Pod v1 y un Pod v2).

  ```bash
  POD_V1="$(kubectl get pod -n "$NS" -l app=helloworld,version=v1 -o jsonpath='{.items[0].metadata.name}')"
  POD_V2="$(kubectl get pod -n "$NS" -l app=helloworld,version=v2 -o jsonpath='{.items[0].metadata.name}')"
  ```
  ```bash
  echo "POD_V1=$POD_V1" | tee outputs/06_pod_v1.txt
  echo "POD_V2=$POD_V2" | tee outputs/06_pod_v2.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl logs -n "$NS" "$POD_V1" -c istio-proxy --tail=120 | tee outputs/06_proxy_logs_v1_tail.txt
  ```
  ```bash
  kubectl logs -n "$NS" "$POD_V2" -c istio-proxy --tail=120 | tee outputs/06_proxy_logs_v2_tail.txt
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Checklist final

Harás un checklist final tipo CKAD (estado de recursos, labels, sidecar, eventos).

#### Tarea 7.1 — Checklist (evidencia)

- {% include step_label.html %} Estado final de recursos del lab.

  ```bash
  kubectl -n "$NS" get deploy,svc,pods -o wide --show-labels | tee outputs/07_final_resources.txt
  ```
  ```bash
  kubectl -n "$NS" get destinationrule,virtualservice -o wide | tee outputs/07_final_istio_resources.txt
  ```
  ```bash
  kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -n 50 | tee outputs/07_final_events_tail.txt
  ```

- {% include step_label.html %} Validación rápida de “mesh membership”: confirma `istio-proxy` en Pods.

  ```bash
  kubectl get pod -n "$NS" -l app=helloworld     -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.containers[*].name}{"\n"}{end}'     | tee outputs/07_final_sidecar_check.txt
  ```

- {% include step_label.html %} Estado del control plane y del plano de datos (proxy-status).

  ```bash
  kubectl -n istio-system get pods -o wide | tee outputs/07_final_istio_system_pods.txt
  ```
  ```bash
  istioctl proxy-status | tee outputs/07_final_istioctl_proxy_status.txt
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Limpieza

Limpiarás los recursos creados por la practica

> **IMPORTANTE:** `istioctl uninstall --purge` elimina recursos de Istio del clúster. Ejecuta ese paso solo si tu clúster NO usa Istio para otras prácticas/ambientes.
{: .lab-note .important .compact}

#### Tarea 8.1

- {% include step_label.html %} Elimina el namespace del laboratorio (remueve todo lo namespaced: Deployments/Services/VS/DR).

  ```bash
  kubectl delete namespace "$NS" --ignore-not-found
  ```
  ```bash
  kubectl get ns | grep -w "$NS" && echo "Namespace aún existe (espera su terminación)" || echo "OK: namespace eliminado" | tee outputs/07_ns_deleted.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Desinstala Istio.

  ```bash
  istioctl uninstall --purge -y | tee outputs/07_istio_uninstall.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl delete namespace istio-system --ignore-not-found
  ```
  {% include step_image.html %}
  ```bash
  kubectl get ns | grep -w istio-system && echo "istio-system aún existe (espera su terminación)" || echo "OK: istio-system eliminado" | tee outputs/07_istio_system_deleted.txt
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