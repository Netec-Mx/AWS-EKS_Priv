---
layout: lab
title: "Práctica 18: Integración de Istio para seguridad mTLS entre microservicios"
permalink: /lab18/lab18/
images_base: /labs/lab18/img
duration: "75 minutos"
objective:
  - >
    Implementar y validar mTLS (mutual TLS) entre microservicios en un clúster Amazon EKS usando Istio (modo sidecar), demostrando el impacto de tráfico en claro (sin sidecar) vs tráfico en mTLS (con sidecar), reforzando la postura de seguridad con AuthorizationPolicy (defensa en profundidad) y ejecutando verificación/troubleshooting al estilo CKAD
    (apply/get/describe/logs/exec/events).
prerequisites:
  - "Windows + Visual Studio Code + Terminal GitBash"
  - "Herramientas: AWS CLI v2, kubectl, eksctl, curl"
  - "Permisos AWS: EKS (crear/describe cluster), EC2 (crear nodegroup), IAM (roles requeridos por eksctl), y acceso de red a internet"
  - "Permisos Kubernetes: crear namespaces, deployments, services y CRDs/políticas de Istio"
  - "(Recomendado) git"
introduction:
  - >
    Istio cifra y autentica el tráfico service-to-service mediante mTLS. Durante migraciones, suele usarse PERMISSIVE (acepta mTLS y tráfico en claro); el objetivo operacional es avanzar a STRICT (solo mTLS) y complementar con AuthorizationPolicy (quién puede hablar con quién). En este laboratorio construirás 3 escenarios (con sidecar en 2 namespaces y sin sidecar en 1 namespace legacy), forzarás mTLS de forma progresiva y validarás resultados con pruebas desde pods y diagnóstico con istioctl/kubectl estilo CKAD.
slug: lab18
lab_number: 18
final_result: >
  Al finalizar tendrás un entorno en EKS con Istio instalado, microservicios de prueba (httpbin/curl) en namespaces con y sin sidecar, mTLS aplicado (PERMISSIVE→STRICT) y reforzado con AuthorizationPolicy (defensa en profundidad y aislamiento por namespace). Habrás capturado evidencia práctica (HTTP codes, eventos, contenedores istio-proxy, peerauth/authorizationpolicy, proxy-status/analyze) y podrás diagnosticar fallas típicas (falta de inyección, políticas que deniegan, handshake mTLS) con comandos tipo CKAD.
notes:
  - "CKAD: práctica de diagnóstico y operación en Kubernetes (namespaces, deployments, labels, rollout, logs/exec/events, JSONPath)."
  - "CKAD: razonar fallos por capas (red/política/config) y validar hipótesis con evidencia (HTTP code, eventos, logs de istio-proxy)."
  - "Seguridad: mTLS (PeerAuthentication) + AuthorizationPolicy son defensa en profundidad; STRICT es el estado final recomendado cuando todo está inyectado."
references:
  - text: "Istio - Download and install (istioctl)"
    url: https://istio.io/latest/docs/setup/getting-started/
  - text: "Istio - Installation with istioctl"
    url: https://istio.io/latest/docs/setup/install/istioctl/
  - text: "Istio - Automatic sidecar injection (namespace label)"
    url: https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/
  - text: "Istio - PeerAuthentication (mTLS modes PERMISSIVE/STRICT)"
    url: https://istio.io/latest/docs/reference/config/security/peer_authentication/
  - text: "Istio - AuthorizationPolicy (ALLOW/DENY)"
    url: https://istio.io/latest/docs/reference/config/security/authorization-policy/
  - text: "Istio - istioctl proxy-status"
    url: https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-proxy-status
  - text: "Istio - istioctl analyze"
    url: https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-analyze
  - text: "eksctl - create cluster"
    url: https://eksctl.io/usage/creating-and-managing-clusters/
  - text: "AWS - Amazon EKS User Guide"
    url: https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html
prev: /lab17/lab17/
next: /lab19/lab19/
---

### Costo (resumen)

- **EKS** cobra por clúster por hora (control plane) y nodos (EC2/EBS).
- **Istio** es OSS, pero consume recursos (CPU/Mem) del clúster.
- Esta práctica NO requiere LoadBalancers externos para validar mTLS.

> **IMPORTANTE:** crear un clúster EKS genera costo si no lo eliminas.
{: .lab-note .important .compact}

### Convenciones del laboratorio

- **Todo** se ejecuta en **GitBash** dentro de **VS Code**.
- Guarda evidencia en `outputs/` para demostrar resultados.
- Si ya tienes clúster, **omite** Tarea 2.

> **NOTA (CKAD):** disciplina de troubleshooting:
> `get` → `describe` → `events` → `logs` → `exec`.
{: .lab-note .info .compact}

---

### Tarea 1. Preparación del workspace y baseline (variables + evidencias)

Crearás la carpeta del lab, estructura estándar, variables reutilizables y evidencias de herramientas e identidad AWS.

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
  mkdir -p lab18 && cd lab18
  mkdir -p 00-prereqs 01-eks 02-istio 03-k8s 04-tests outputs logs
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que la estructura quedó creada.

  ```bash
  find . -maxdepth 2 -type d | sort | tee outputs/00_dirs.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables base (ajusta región y cluster).

  ```bash
  export AWS_REGION="us-west-2"
  export CLUSTER_NAME="eks-istio-mtls-lab"
  export LAB_ID="$(date +%Y%m%d%H%M%S)"
  export NS_MTLS="mtls-demo"
  export NS_OTHER="other-demo"
  export NS_LEGACY="legacy-demo"
  export ISTIO_VERSION="1.28.2"
  ```

- {% include step_label.html %} Captura identidad AWS y guarda variables para reuso (reproducible).

  ```bash
  export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  export CALLER_ARN="$(aws sts get-caller-identity --query Arn --output text)"
  ```
  ```bash
  cat > outputs/vars.env <<EOF
  AWS_REGION=$AWS_REGION
  CLUSTER_NAME=$CLUSTER_NAME
  LAB_ID=$LAB_ID
  NS_MTLS=$NS_MTLS
  NS_OTHER=$NS_OTHER
  NS_LEGACY=$NS_LEGACY
  ISTIO_VERSION=$ISTIO_VERSION
  AWS_ACCOUNT_ID=$AWS_ACCOUNT_ID
  CALLER_ARN=$CALLER_ARN
  EOF
  ```
  ```bash
  aws sts get-caller-identity --output json | tee outputs/01_sts_identity.json
  ```
  {% include step_image.html %}
  ```bash
  cat outputs/vars.env | tee outputs/01_vars_echo.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica herramientas (evidencia).

  ```bash
  aws --version | tee outputs/01_aws_version.txt
  kubectl version --client=true | tee outputs/01_kubectl_version.txt
  eksctl version | tee outputs/01_eksctl_version.txt
  curl --version | head -n 2 | tee outputs/01_curl_version.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear un clúster EKS para el laboratorio (opcional)

Si ya tienes un clúster funcional y `kubectl` apunta al contexto correcto, **puedes omitir** la creación.

> **IMPORTANTE:** elimina recursos al final si este clúster es solo para el lab.
{: .lab-note .important .compact}

#### Tarea 2.1

- {% include step_label.html %} (Reuso) Valida si ya tienes un clúster accesible.

  ```bash
  kubectl config current-context 2>/dev/null | tee outputs/02_current_context.txt || true
  kubectl get nodes -o wide 2>/dev/null | tee outputs/02_nodes_if_any.txt || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Si NO reusas) Lista versiones soportadas y elige una (no adivines).

  ```bash
  eksctl utils describe-cluster-versions strings --region "$AWS_REGION" \
    | jq -r '.clusterVersions[].ClusterVersion'
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Si NO reusas) Define versión y crea el clúster con 2 nodos.

  > **IMPORTANTE:** El cluster tardara aproximadamente **15 minutos** en crearse. Espera el proceso
  {: .lab-note .important .compact}

  ```bash
  export K8S_VERSION="${K8S_VERSION:-1.33}"
  export NODEGROUP_NAME="mng-istio"

  {
    echo "K8S_VERSION=$K8S_VERSION"
    echo "NODEGROUP_NAME=$NODEGROUP_NAME"
  } | tee outputs/02_k8s_nodegroup_vars.txt

  eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --version "$K8S_VERSION" \
    --managed \
    --nodegroup-name "$NODEGROUP_NAME" \
    --node-type t3.medium \
    --nodes 2 --nodes-min 2 --nodes-max 3 \
    | tee outputs/02_eksctl_create_cluster.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Configura kubeconfig y valida contexto.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION" | tee outputs/02_update_kubeconfig.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl config current-context | tee outputs/02_kube_context.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica nodos Ready.

  ```bash
  kubectl get nodes -o wide | tee outputs/02_nodes_wide.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl wait --for=condition=Ready node --all --timeout=600s | tee outputs/02_wait_nodes_ready.txt
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

- {% include step_label.html %} Baseline de eventos recientes.

  ```bash
  kubectl get events -A --sort-by=.lastTimestamp | tail -n 30 | tee outputs/02_events_tail_baseline.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Descargar e Instalar Istio y validar el control plane

Descargarás Istio, habilitarás `istioctl` en PATH de la sesión y validarás `istiod` + CRDs.

> **NOTA (CKAD):** después de instalar/aplicar: `get pods` → `describe` si falla → `events`.
{: .lab-note .info .compact}

#### Tarea 3.1

- {% include step_label.html %} Descarga Istio (crea `istio-<version>/`).

  ```bash
  cd 02-istio
  ```
  ```bash
  export ISTIO_VERSION="${ISTIO_VERSION}"
  OS="$(uname -s | tr '[:upper:]' '[:lower:]')"
  echo "OS=$OS ISTIO_VERSION=$ISTIO_VERSION" | tee ../outputs/03_os_istio.txt
  ```
  ```bash
  if [[ "$OS" == mingw* || "$OS" == msys* || "$OS" == cygwin* ]]; then
    ZIP="istio-${ISTIO_VERSION}-win.zip"
    curl -fsSL -o "$ZIP" "https://github.com/istio/istio/releases/download/${ISTIO_VERSION}/${ZIP}"

    if command -v unzip >/dev/null 2>&1; then
      unzip -q -o "$ZIP"
    else
      powershell.exe -NoProfile -Command "Expand-Archive -Force '$ZIP' '.'"
    fi
  else
    curl -fsSL https://istio.io/downloadIstio | ISTIO_VERSION="$ISTIO_VERSION" sh -
  fi
  ```
  ```bash
  ls -1 | grep -E '^istio-' | tail -n 5 | tee ../outputs/03_istio_dirs.txt
  ```
  {% include step_image.html %}

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

#### Tarea 3.2

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

### Tarea 4. Desplegar workloads de prueba y validar sidecar (con vs sin inyección)

Crearás 3 namespaces: dos con inyección (`enabled`) y uno legacy sin sidecar.
Luego desplegarás `httpbin` (server) y `curl` (client) y confirmarás contenedores.

#### Tarea 4.1

- {% include step_label.html %} Crea namespaces (idempotente) y etiqueta inyección en 2.

  ```bash
  kubectl create ns "$NS_MTLS"   --dry-run=client -o yaml | kubectl apply -f -
  kubectl create ns "$NS_OTHER"  --dry-run=client -o yaml | kubectl apply -f -
  kubectl create ns "$NS_LEGACY" --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}
  ```bash
  kubectl label ns "$NS_MTLS"  istio-injection=enabled --overwrite
  kubectl label ns "$NS_OTHER" istio-injection=enabled --overwrite
  ```
  {% include step_image.html %}
  ```bash
  kubectl get ns --show-labels | egrep "($NS_MTLS|$NS_OTHER|$NS_LEGACY)" | tee outputs/04_ns_labels.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Copia manifiestos de samples a tu directorio (reproducible).

  ```bash
  export ISTIO_DIR="$(ls -1d 02-istio/istio-*/ 2>/dev/null | tail -n 1)"
  echo "ISTIO_DIR=$ISTIO_DIR" | tee outputs/03_istio_dir_fixed.txt
  ```
  ```bash
  cp "$ISTIO_DIR"/samples/httpbin/httpbin.yaml 03-k8s/10_httpbin.yaml
  cp "$ISTIO_DIR"/samples/curl/curl.yaml 03-k8s/11_curl.yaml
  ```
  ```bash
  ls -la 03-k8s | tee outputs/04_ls_03k8s.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Despliega `httpbin` en `mtls-demo` y `curl` en los 3 namespaces.

  ```bash
  kubectl apply -n "$NS_MTLS" -f 03-k8s/10_httpbin.yaml | tee outputs/04_apply_httpbin.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl apply -n "$NS_MTLS" -f 03-k8s/11_curl.yaml | tee outputs/04_apply_curl_mtls.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl apply -n "$NS_OTHER" -f 03-k8s/11_curl.yaml | tee outputs/04_apply_curl_other.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl apply -n "$NS_LEGACY" -f 03-k8s/11_curl.yaml | tee outputs/04_apply_curl_legacy.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera a que estén listos.

  ```bash
  kubectl -n "$NS_MTLS" rollout status deploy/httpbin | tee outputs/04_rollout_httpbin.txt
  ```
  ```bash
  kubectl -n "$NS_MTLS" wait --for=condition=Ready pod -l app=curl --timeout=240s | tee outputs/04_wait_curl_mtls.txt
  ```
  ```bash
  kubectl -n "$NS_OTHER" wait --for=condition=Ready pod -l app=curl --timeout=240s | tee outputs/04_wait_curl_other.txt
  ```
  ```bash
  kubectl -n "$NS_LEGACY" wait --for=condition=Ready pod -l app=curl --timeout=240s | tee outputs/04_wait_curl_legacy.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Evidencia: sidecar presente vs ausente (contenedores).

  ```bash
  { kubectl -n "$NS_MTLS" get pod -l app=curl -o jsonpath='{.items[0].spec.containers[*].name}'; echo; } \
    | tee outputs/04_containers_mtls.txt
  ```
  {% include step_image.html %}
  ```bash
  { kubectl -n "$NS_LEGACY" get pod -l app=curl -o jsonpath='{.items[0].spec.containers[*].name}'; echo; } \
    | tee outputs/04_containers_legacy.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda nombres de pods de curl (para pruebas).

  ```bash
  export POD_MTLS="$(kubectl -n "$NS_MTLS" get pod -l app=curl -o jsonpath='{.items[0].metadata.name}')"
  export POD_OTHER="$(kubectl -n "$NS_OTHER" get pod -l app=curl -o jsonpath='{.items[0].metadata.name}')"
  export POD_LEGACY="$(kubectl -n "$NS_LEGACY" get pod -l app=curl -o jsonpath='{.items[0].metadata.name}')"

  {
    echo "POD_MTLS=$POD_MTLS"
    echo "POD_OTHER=$POD_OTHER"
    echo "POD_LEGACY=$POD_LEGACY"
  } | tee outputs/04_curl_pods.txt
  ```
  {% include step_image.html %}

{% include step_label.html %} Verifica recursos y eventos en namespaces clave (evidencia).

  ```bash
  kubectl -n "$NS_MTLS" get deploy,svc,pods -o wide | tee outputs/04_mtls_resources.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_OTHER" get pods -o wide | tee outputs/04_other_pods.txt 
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_LEGACY" get pods -o wide | tee outputs/04_legacy_pods.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get events -A --sort-by=.lastTimestamp | tail -n 20 | tee outputs/04_events_tail.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Baseline de conectividad (antes de políticas)

Confirmas que, sin forzar mTLS aún, el tráfico HTTP suele funcionar desde los tres clientes.

#### Tarea 5.1

- {% include step_label.html %} Ejecuta pruebas baseline (captura HTTP codes).

  ```bash
  kubectl -n "$NS_MTLS" exec "$POD_MTLS" -c curl -- \
    curl -sS -o /dev/null -w "mtls-demo -> httpbin: %{http_code}\n" http://httpbin.mtls-demo:8000/ip \
    | tee outputs/05_baseline_mtls.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_OTHER" exec "$POD_OTHER" -c curl -- \
    curl -sS -o /dev/null -w "other-demo -> httpbin: %{http_code}\n" http://httpbin.mtls-demo:8000/ip \
    | tee outputs/05_baseline_other.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_LEGACY" exec "$POD_LEGACY" -c curl -- \
    curl -sS -o /dev/null -w "legacy-demo -> httpbin: %{http_code}\n" http://httpbin.mtls-demo:8000/ip \
    | tee outputs/05_baseline_legacy.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Forzar mTLS (PERMISSIVE → STRICT) y bloquear plaintext (defense in depth)

Primero aplicas PERMISSIVE (migración), pero bloqueas tráfico sin identidad (plaintext) con AuthorizationPolicy.
Luego cambias a STRICT (estado final).

> **NOTA (CKAD):** Diferencia de síntomas:
- **AuthorizationPolicy** típicamente → `403`
- **STRICT mTLS** contra legacy sin sidecar → fallas de conexión/handshake (código 000, reset, 503)
{: .lab-note .info .compact}

#### Tarea 6.1

- {% include step_label.html %} PeerAuthentication PERMISSIVE en `mtls-demo`.

  ```bash
  cat > 03-k8s/20_peerauth_permissive.yaml <<EOF
  apiVersion: security.istio.io/v1
  kind: PeerAuthentication
  metadata:
    name: default
    namespace: $NS_MTLS
  spec:
    mtls:
      mode: PERMISSIVE
  EOF
  ```
  ```bash
  kubectl apply -f 03-k8s/20_peerauth_permissive.yaml | tee outputs/06_apply_peerauth_permissive.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} AuthorizationPolicy DENY si no hay principal (bloquea plaintext).

  ```bash
  cat > 03-k8s/21_authz_require_mtls_deny_plaintext.yaml <<EOF
  apiVersion: security.istio.io/v1
  kind: AuthorizationPolicy
  metadata:
    name: require-mtls
    namespace: $NS_MTLS
  spec:
    action: DENY
    rules:
    - from:
      - source:
          notPrincipals: ["*"]
  EOF
  ```
  ```bash
  kubectl apply -f 03-k8s/21_authz_require_mtls_deny_plaintext.yaml | tee outputs/06_apply_authz_require_mtls.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba en PERMISSIVE + DENY plaintext (legacy debe quedar bloqueado).

  > **NOTA:** Los resultados deberian ser:
  - `mtls-demo` y `other-demo`: `200`
  - `legacy-demo`: típicamente `403`
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$NS_MTLS" exec "$POD_MTLS" -c curl -- \
    curl -sS -o /dev/null -w "mtls-demo -> httpbin: %{http_code}\n" http://httpbin.mtls-demo:8000/ip \
    | tee outputs/06_test_perm_mtls.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_OTHER" exec "$POD_OTHER" -c curl -- \
    curl -sS -o /dev/null -w "other-demo -> httpbin: %{http_code}\n" http://httpbin.mtls-demo:8000/ip \
    | tee outputs/06_test_perm_other.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_LEGACY" exec "$POD_LEGACY" -c curl -- \
    sh -lc 'curl -sS -o /dev/null -w "legacy-demo -> httpbin: %{http_code}\n" http://httpbin.mtls-demo:8000/ip || true' \
    | tee outputs/06_test_perm_legacy.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Cambia a mTLS STRICT (estado final recomendado).

  ```bash
  cat > 03-k8s/22_peerauth_strict.yaml <<EOF
  apiVersion: security.istio.io/v1
  kind: PeerAuthentication
  metadata:
    name: default
    namespace: $NS_MTLS
  spec:
    mtls:
      mode: STRICT
  EOF
  ```
  ```bash
  kubectl apply -f 03-k8s/22_peerauth_strict.yaml | tee outputs/06_apply_peerauth_strict.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba en STRICT (legacy debe fallar sí o sí; sidecars deben seguir OK).

  ```bash
  kubectl -n "$NS_MTLS" exec "$POD_MTLS" -c curl -- \
    curl -sS -o /dev/null -w "mtls-demo -> httpbin: %{http_code}\n" http://httpbin.mtls-demo:8000/ip \
    | tee outputs/06_test_strict_mtls.txt
  ```
  ```bash
  kubectl -n "$NS_OTHER" exec "$POD_OTHER" -c curl -- \
    curl -sS -o /dev/null -w "other-demo -> httpbin: %{http_code}\n" http://httpbin.mtls-demo:8000/ip \
    | tee outputs/06_test_strict_other.txt
  ```
  ```bash
  kubectl -n "$NS_LEGACY" exec "$POD_LEGACY" -c curl -- \
    sh -lc 'curl -sS -o /dev/null -w "legacy-demo -> httpbin: %{http_code}\n" http://httpbin.mtls-demo:8000/ip || echo "legacy-demo -> httpbin: FAILED (expected)"' \
    | tee outputs/06_test_strict_legacy.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida políticas de seguridad y captura logs del sidecar (evidencia).

  ```bash
  kubectl -n "$NS_MTLS" get peerauthentication,authorizationpolicy -o wide | tee outputs/06_get_policies.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_OTHER" logs "$POD_OTHER" -c istio-proxy --tail=80 2>&1 | tee outputs/06_istio_proxy_logs_other_tail.txt || true
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Aislar acceso por namespace con AuthorizationPolicy (ALLOW solo desde mtls-demo)

Ahora haces control de acceso: aunque `other-demo` tenga sidecar (mTLS), **ya no debe poder** hablar con `httpbin`.

#### Tarea 7.1

- {% include step_label.html %} Crea AuthorizationPolicy ALLOW solo desde `mtls-demo`.

  ```bash
  cat > 03-k8s/30_authz_allow_same_namespace.yaml <<EOF
  apiVersion: security.istio.io/v1
  kind: AuthorizationPolicy
  metadata:
    name: mtls-demo-isolation
    namespace: $NS_MTLS
  spec:
    action: ALLOW
    rules:
    - from:
      - source:
          namespaces: ["$NS_MTLS"]
  EOF
  ```
  ```bash
  kubectl apply -f 03-k8s/30_authz_allow_same_namespace.yaml | tee outputs/07_apply_authz_isolation.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba accesos: mtls-demo permitido / other-demo denegado.

  > **NOTA:** Los resultados deberian ser:
  - `mtls-demo`: `200`
  - `other-demo`: típicamente `403`
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$NS_MTLS" exec "$POD_MTLS" -c curl -- \
    curl -sS -o /dev/null -w "mtls-demo -> httpbin: %{http_code}\n" http://httpbin.mtls-demo:8000/ip \
    | tee outputs/07_test_allow_mtls.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_OTHER" exec "$POD_OTHER" -c curl -- \
    curl -sS -o /dev/null -w "other-demo -> httpbin: %{http_code}\n" http://httpbin.mtls-demo:8000/ip \
    | tee outputs/07_test_deny_other.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Inspecciona políticas y salud de la malla Istio (evidencia).

  ```bash
  kubectl -n "$NS_MTLS" get authorizationpolicy -o wide | tee outputs/07_get_authz.txt
  ```
  {% include step_image.html %}
  ```bash
  istioctl analyze --namespace "$NS_MTLS" 2>&1 | tee outputs/07_analyze_mtls_ns.txt || true
  ```
  {% include step_image.html %}
  ```bash
  istioctl proxy-status 2>&1 | tee outputs/07_proxy_status.txt || true
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Limpieza del laboratorio (recomendada)

Elimina namespaces del lab y, opcionalmente, desinstala Istio y/o elimina el clúster si fue creado solo para esta práctica.

#### Tarea 8.1

- {% include step_label.html %} (Si reiniciaste terminal) Recarga variables guardadas.

  ```bash
  set -a
  source outputs/vars.env
  set +a
  ```

- {% include step_label.html %} Borra namespaces del lab (incluye workloads/policies).

  ```bash
  kubectl delete ns "$NS_MTLS" "$NS_OTHER" "$NS_LEGACY" --ignore-not-found | tee outputs/08_delete_namespaces.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Desinstala Istio.

  ```bash
  istioctl uninstall --purge -y 2>&1 | tee outputs/08_istio_uninstall.txt || true
  ```
  {% include step_image.html %}
  ```bash
  kubectl delete ns istio-system --ignore-not-found | tee outputs/08_delete_istio_system_ns.txt
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