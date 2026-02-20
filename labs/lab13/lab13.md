---
layout: lab
title: "Práctica 13: Implementación de escalado automático con KEDA (EKS)"
permalink: /lab13/lab13/
images_base: /labs/lab13/img
duration: "60 minutos"
objective:
  - "Instalar KEDA en un clúster Amazon EKS y habilitar escalado event-driven (scale-to-zero incluido) para un microservicio tipo worker que procesa mensajes desde Amazon SQS, controlando el escalado mediante un ScaledObject y TriggerAuthentication (IRSA), y validando el comportamiento con troubleshooting estilo CKAD (HPA, eventos, logs, describe, etc.)."
prerequisites:
  - "Amazon EKS accesible con kubectl"
  - "Windows + Visual Studio Code + Terminal GitBash"
  - "Herramientas: AWS CLI v2, kubectl, eksctl, helm, docker, git, curl"
  - "Permisos AWS: crear SQS, IAM policy/role (IRSA) y (opcional) ECR"
  - "Permisos Kubernetes: crear namespaces, CRDs, Deployments, ServiceAccounts y leer logs/events"
  - "(Recomendado) OIDC provider habilitado para IRSA (se valida y configura en el laboratorio)"
introduction: |
  KEDA (Kubernetes Event-driven Autoscaling) complementa al HPA: consulta fuentes externas (colas, streams, métricas) y expone esas señales como métricas para que el HPA escale. A diferencia del HPA “clásico”, KEDA habilita **scale-to-zero** (0 réplicas) cuando no hay eventos.
  - En esta práctica, el evento será el backlog de una cola **Amazon SQS** y el workload será un **worker** que hace long polling, procesa mensajes y los elimina. Verás el ciclo completo: 0 → N réplicas cuando llegan mensajes y N → 0 cuando se vacía la cola.
slug: lab13
lab_number: 13
final_result: >
  Al finalizar tendrás KEDA instalado en EKS, una cola SQS como fuente de eventos, un worker desplegado con identidad IAM vía IRSA, y un flujo completo de autoescalado event-driven controlado por ScaledObject/TriggerAuthentication, validado con HPA, logs del worker, métricas de la cola y troubleshooting operativo (describe/events/logs) estilo CKAD.
notes:
  - "CKAD: práctica núcleo de troubleshooting operativo (Deployment/Pods/SA/env vars + HPA + describe/events/logs + pods efímeros para validar identidad)."
  - "Este laboratorio evita exponer endpoints con LoadBalancer. Todo se valida desde dentro del clúster o con CLI, minimizando costos y superficie."
  - "Principio clave: KEDA decide **cuándo activar** (0 → 1) y expone métricas; el HPA ejecuta el escalado **1 → N** sobre el Deployment."
references:
  - text: "KEDA - Install (Helm chart)"
    url: https://keda.sh/docs/latest/deploy/
  - text: "KEDA - AWS SQS Queue scaler (metadata, activationQueueLength, queueLength)"
    url: https://keda.sh/docs/latest/scalers/aws-sqs/
  - text: "KEDA - Authentication (TriggerAuthentication / podIdentity provider: aws)"
    url: https://keda.sh/docs/2.18/concepts/authentication/
  - text: "AWS - IAM roles for service accounts (IRSA) en EKS"
    url: https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html
  - text: "eksctl - associate OIDC provider"
    url: https://eksctl.io/usage/iamserviceaccounts/#associate-iam-oidc-provider
  - text: "eksctl - create iamserviceaccount (IRSA)"
    url: https://eksctl.io/usage/iamserviceaccounts/
  - text: "AWS - Amazon EKS pricing"
    url: https://aws.amazon.com/eks/pricing/
  - text: "AWS - Amazon SQS pricing"
    url: https://aws.amazon.com/sqs/pricing/
  - text: "AWS - Amazon ECR pricing"
    url: https://aws.amazon.com/ecr/pricing/
prev: /lab12/lab12/
next: /lab14/lab14/
---

## Costo (resumen)

- **KEDA** es OSS (sin costo de licencia), pero consume CPU/RAM en tu clúster.
- **EKS** cobra tarifa por clúster por hora y los **nodos** (EC2/EBS/transferencia) también generan costo.
- **SQS** cobra por requests (Send/Receive/Delete, etc.).
- **ECR** (si lo usas) cobra por almacenamiento/transferencia.

> **IMPORTANTE:** Este laboratorio evita LoadBalancers. Todo se valida con CLI y recursos internos, minimizando costos y variabilidad.
{: .lab-note .important .compact}

---

### Tarea 1. Preparación del workspace y baseline del clúster (EKS + AWS)

Crearás la carpeta del laboratorio, definirás variables, validarás que estás en la **cuenta/región correctas** y que `kubectl` apunta al **clúster correcto** con permisos para instalar CRDs/controladores. Esto evita “misterios” al depurar.

> **NOTA (CKAD):** Entrena hábitos de examen: validar **context**, **namespace**, permisos con `kubectl auth can-i`, y capturar evidencia con `describe/events/logs`.
{: .lab-note .info .compact}

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
  mkdir -p lab13 && cd lab13
  mkdir -p 00-prereqs 01-keda 02-aws 03-app 04-keda 05-tests outputs logs
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que la estructura de carpetas quedó creada.

  ```bash
  find . -maxdepth 2 -type d | sort
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea un archivo `.env` con variables base (recomendado para reproducibilidad).

  > **IMPORTANTE:** No asumas valores. Ajusta explícitamente región y nombre de clúster.
  {: .lab-note .important .compact}

  ```bash
  cat > .env <<'EOF'
  # Región donde existe tu clúster EKS y donde crearás SQS/ECR
  export AWS_REGION="us-west-2"

  # Namespace del laboratorio
  export NS="keda-demo"

  # Identificador único para evitar colisiones de nombres (SQS/IAM/ECR tags)
  export LAB_ID="$(date +%Y%m%d%H%M%S)"

  # Recursos AWS del lab
  export QUEUE_NAME="keda-lab-queue-$LAB_ID"
  export ECR_REPO="keda-sqs-worker"
  export IMAGE_TAG="lab13-$LAB_ID"
  EOF
  ```
  ```bash
  source .env
  ```

- {% include step_label.html %} Guarda variables en evidencia.

  ```bash
  echo "AWS_REGION=$AWS_REGION NS=$NS LAB_ID=$LAB_ID QUEUE_NAME=$QUEUE_NAME ECR_REPO=$ECR_REPO IMAGE_TAG=$IMAGE_TAG" | tee outputs/01_vars.txt
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
  docker --version
  ```

  ```bash
  git --version
  ```

  ```bash
  curl --version
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida tu identidad en AWS (si falla, corrige credenciales antes de continuar).

  > **NOTA:** Verifica que no existan errores.
  {: .lab-note .info .compact}

  ```bash
  aws sts get-caller-identity | tee outputs/01_aws_identity.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Obligatorio si no estás 100% seguro del nombre) Lista clústeres en la región y confirma `CLUSTER_NAME`.

  > **NOTA:** Sino hay cluster activo avanza para crear uno.
  {: .lab-note .info .compact}

  ```bash
  aws eks list-clusters --region "$AWS_REGION" --output table | tee outputs/01_eks_list_clusters.txt
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
  export CLUSTER_NAME="eks-keda-lab"
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
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.status" --output text | tee outputs/02_cluster_status.txt
  ```
  {% include step_image.html %}
  ```bash
  aws eks list-nodegroups --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION" --output table | tee outputs/02_nodegroups_table.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Configura `kubeconfig` para el clúster indicado y valida conectividad.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION" | tee outputs/01_eks_update_kubeconfig.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl config current-context | tee outputs/01_kubectl_current_context.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl cluster-info | tee outputs/01_kubectl_cluster_info.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get nodes -o wide | tee outputs/01_kubectl_nodes.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asocia OIDC provider (recomendado para IRSA/controladores). Este paso es seguro de repetir.

  ```bash
  eksctl utils associate-iam-oidc-provider --cluster "$CLUSTER_NAME" --region "$AWS_REGION" --approve | tee outputs/02_associate_oidc.txt
  ```
  {% include step_image.html %}

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.identity.oidc.issuer" --output text | tee outputs/02_oidc_issuer.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica permisos mínimos para instalar KEDA (CRDs + recursos cluster-scope).

  ```bash
  kubectl auth can-i create namespace | tee outputs/01_rbac_can_i_ns.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl auth can-i create customresourcedefinitions.apiextensions.k8s.io | tee outputs/01_rbac_can_i_crd.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl auth can-i create clusterrole | tee outputs/01_rbac_can_i_clusterrole.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Instalar KEDA en EKS con Helm

Instalarás KEDA en el namespace `keda` con el chart oficial y validarás que queden listos el **operator** y el **metrics apiserver** (KEDA sirve métricas al HPA).

> **IMPORTANTE:** Si `keda-operator-metrics-apiserver` no está Ready, el HPA no recibirá métricas y el escalado fallará.
{: .lab-note .important .compact}

#### Tarea 3.1

- {% include step_label.html %} Agrega el repositorio oficial de KEDA y actualiza índices.

  ```bash
  helm repo add kedacore https://kedacore.github.io/charts
  helm repo update
  ```
  {% include step_image.html %}

- {% include step_label.html %} Instala/actualiza KEDA (idempotente) en `keda`.

  ```bash
  helm upgrade --install keda kedacore/keda --namespace keda --create-namespace | tee outputs/03_helm_keda_install.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera a que los pods de KEDA queden **Ready** (sin “adivinar tiempos”).

  > **NOTA:** Puede ser normal ver 1 restart del keda-operator justo después de instalarlo (primer minuto), sobre todo si:
  - El Pod arrancó antes de que estuvieran listos CRDs / APIService / webhooks
  - Hubo un readiness/liveness probe fallando al inicio
  - El contenedor recibió una señal (rollout inicial / “warm-up”)
  {: .lab-note .info .compact}

  ```bash
  kubectl -n keda get pods -o wide | tee outputs/03_keda_pods_initial.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n keda wait --for=condition=Available deploy/keda-operator --timeout=600s
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n keda wait --for=condition=Available deploy/keda-operator-metrics-apiserver --timeout=600s
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n keda get pods -o wide | tee outputs/03_keda_pods_ready.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida CRDs de KEDA instalados.

  ```bash
  kubectl get crd | egrep -i "scaledobjects.keda.sh|triggerauthentications.keda.sh|scaledjobs.keda.sh" | tee outputs/03_keda_crds.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Crear SQS y configurar IRSA (OIDC + IAM Policy/Role + ServiceAccount)

Crearás una cola SQS (fuente de eventos) y configurarás un ServiceAccount con IAM Role (IRSA). La identidad IAM se materializa con la anotación `eks.amazonaws.com/role-arn`. Luego validarás la identidad con un pod efímero de AWS CLI.

> **NOTA (CKAD):** Practicarás `ServiceAccount`, variables de entorno y validación con pods efímeros (`kubectl run --rm -it`), además de capturar evidencia para depuración.
{: .lab-note .info .compact}

#### Tarea 4.1 — Crear namespace del lab y cola SQS

- {% include step_label.html %} Crea el namespace del laboratorio (idempotente).

  ```bash
  kubectl create ns "$NS" --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}
  ```bash
  kubectl get ns "$NS" -o wide | tee outputs/04_ns_created.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea (o recupera) la URL de la cola SQS.

  ```bash
  QUEUE_URL="$(aws sqs get-queue-url --queue-name "$QUEUE_NAME" --region "$AWS_REGION" --query QueueUrl --output text 2>/dev/null || true)"
  ```
  ```bash
  if [ -z "$QUEUE_URL" ]; then
    QUEUE_URL="$(aws sqs create-queue --queue-name "$QUEUE_NAME" --region "$AWS_REGION" --query QueueUrl --output text)"
  fi
  ```
  ```bash
  echo "QUEUE_URL=$QUEUE_URL" | tee outputs/04_queue_url.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén el ARN de la cola (se usa para mínimo privilegio).

  ```bash
  QUEUE_ARN="$(aws sqs get-queue-attributes --queue-url "$QUEUE_URL" --attribute-names QueueArn --region "$AWS_REGION" --query Attributes.QueueArn --output text)"
  ```
  ```bash
  echo "QUEUE_ARN=$QUEUE_ARN" | tee outputs/04_queue_arn.txt
  ```
  {% include step_image.html %}

#### Tarea 4.2 — Crear IAM Policy mínima (escala + consumo)

- {% include step_label.html %} Crea el documento de policy (mínimo privilegio) en `02-aws/iam-policy-sqs.json`.

  ```bash
  cat > 02-aws/iam-policy-sqs.json <<EOF
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "KedaScaleSignals",
        "Effect": "Allow",
        "Action": [
          "sqs:GetQueueAttributes"
        ],
        "Resource": "$QUEUE_ARN"
      },
      {
        "Sid": "WorkerConsume",
        "Effect": "Allow",
        "Action": [
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:ChangeMessageVisibility"
        ],
        "Resource": "$QUEUE_ARN"
      }
    ]
  }
  EOF 
  ```

- {% include step_label.html %} Crea (o reutiliza) la IAM Policy y guarda su ARN.

  ```bash
  export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  export POLICY_NAME="keda-sqs-policy-$LAB_ID"
  ```
  ```bash
  POLICY_ARN="$(aws iam list-policies --scope Local --query "Policies[?PolicyName=='$POLICY_NAME'].Arn | [0]" --output text)"
  ```
  ```bash
  if [ "$POLICY_ARN" = "None" ] || [ -z "$POLICY_ARN" ]; then
    POLICY_ARN="$(aws iam create-policy --policy-name "$POLICY_NAME" --policy-document file://02-aws/iam-policy-sqs.json --query Policy.Arn --output text)"
  fi
  ```
  ```bash
  echo "POLICY_ARN=$POLICY_ARN" | tee outputs/04_policy_arn.txt
  ```
  {% include step_image.html %}

#### Tarea 4.3 — Habilitar OIDC provider (IRSA) y crear ServiceAccount con eksctl

- {% include step_label.html %} Asocia el OIDC provider al clúster (requisito de IRSA). **Verifica que ya este asociado**

  ```bash
  eksctl utils associate-iam-oidc-provider --cluster "$CLUSTER_NAME" --region "$AWS_REGION" --approve | tee outputs/04_eksctl_associate_oidc.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén el nombre real del ServiceAccount del KEDA Operator (desde el Deployment).

  ```bash
  export KEDA_NS="keda"
  export KEDA_OPERATOR_SA="$(kubectl -n "$KEDA_NS" get deploy keda-operator -o jsonpath='{.spec.template.spec.serviceAccountName}')"
  echo "KEDA_OPERATOR_SA=$KEDA_OPERATOR_SA" | tee outputs/04_keda_operator_sa.txt
  ```

- {% include step_label.html %} Crea IRSA para el KEDA Operator (necesario para que el scaler haga GetQueueAttributes en SQS).

  > **NOTA:** Este proceso puede tardar 2 minutos en terminar.
  {: .lab-note .info .compact}

  ```bash
  eksctl create iamserviceaccount \
    --cluster "$CLUSTER_NAME" --region "$AWS_REGION" \
    --namespace "$KEDA_NS" --name "$KEDA_OPERATOR_SA" \
    --role-name "${CLUSTER_NAME}-keda-operator-irsa" \
    --attach-policy-arn "$POLICY_ARN" \
    --override-existing-serviceaccounts \
    --approve | tee outputs/04_eksctl_irsa_keda_operator.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida que el ServiceAccount tiene la anotación `eks.amazonaws.com/role-arn`.

  ```bash
  kubectl -n "$KEDA_NS" get sa "$KEDA_OPERATOR_SA" -o yaml | sed -n '1,140p' | tee outputs/04_keda_operator_sa_yaml.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Reinicia el KEDA Operator para que tome la anotación IRSA y valida el rollout.

  ```bash
  kubectl -n "$KEDA_NS" rollout restart deploy/keda-operator
  kubectl -n "$KEDA_NS" rollout status deploy/keda-operator --timeout=300s
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea IRSA para el worker (sqs-worker-sa) en el namespace del laboratorio ($NS) adjuntando la policy (IRSA).

  > **NOTA:** Este proceso puede tardar 2 minutos en terminar.
  {: .lab-note .info .compact}

  ```bash
  eksctl create iamserviceaccount \
    --cluster "$CLUSTER_NAME" --region "$AWS_REGION" \
    --namespace "$NS" --name "sqs-worker-sa" \
    --role-name "${CLUSTER_NAME}-sqs-worker-irsa" \
    --attach-policy-arn "$POLICY_ARN" \
    --override-existing-serviceaccounts \
    --approve | tee outputs/04_eksctl_irsa_sqs_worker.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida que el ServiceAccount del worker tiene la anotación `eks.amazonaws.com/role-arn`.

  ```bash
  kubectl -n "$NS" get sa "sqs-worker-sa" -o yaml | sed -n '1,120p' | tee outputs/04_sqs_worker_sa_yaml.txt
  ```
  {% include step_image.html %}

#### Tarea 4.4 — Validación práctica de IRSA (pod efímero AWS CLI)

- {% include step_label.html %} Lanza un pod efímero con la SA y consulta atributos de la cola.

  ```bash
  kubectl run awscli -n "$NS" \
    --image=public.ecr.aws/aws-cli/aws-cli:latest \
    --restart=Never -it --rm \
    --overrides='{
      "apiVersion":"v1",
      "spec":{"serviceAccountName":"sqs-worker-sa"}
    }' \
    -- sqs get-queue-attributes \
      --queue-url "$QUEUE_URL" \
      --attribute-names ApproximateNumberOfMessages \
      --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda un “snapshot” de atributos para evidencia (desde tu máquina).

  ```bash
  aws sqs get-queue-attributes --queue-url "$QUEUE_URL" --attribute-names ApproximateNumberOfMessages --region "$AWS_REGION" | tee outputs/04_sqs_attributes_baseline.json
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Construir imagen del worker (Docker) y publicarla en ECR

Construirás una imagen Docker local para un worker que hace long polling en SQS y procesa mensajes (simulado con `sleep`). La publicarás en ECR para que el clúster EKS la ejecute.

> **IMPORTANTE:** Esta práctica usa IRSA: NO uses access keys en Secrets. El contenedor debe usar la cadena de credenciales por defecto del SDK (boto3), lo que permitirá tomar credenciales desde IRSA.
{: .lab-note .important .compact}

#### Tarea 5.1 — Código del worker (Python)

- {% include step_label.html %} Crea el archivo `03-app/worker.py`.

  ```bash
  cat > 03-app/worker.py <<'EOF'
  import os, time
  import boto3

  QUEUE_URL = os.environ["QUEUE_URL"]
  REGION = os.environ.get("AWS_REGION", "us-east-1")
  SLEEP_SECS = int(os.environ.get("PROCESS_SECONDS", "2"))

  sqs = boto3.client("sqs", region_name=REGION)
  print(f"[worker] starting. region={REGION} queue={QUEUE_URL} process_seconds={SLEEP_SECS}")

  while True:
      resp = sqs.receive_message(
          QueueUrl=QUEUE_URL,
          MaxNumberOfMessages=10,
          WaitTimeSeconds=20,         # long polling
          VisibilityTimeout=30
      )
      msgs = resp.get("Messages", [])
      if not msgs:
          print("[worker] no messages")
          continue

      for m in msgs:
          body = (m.get("Body") or "")[:200]
          rh = m["ReceiptHandle"]
          print(f"[worker] got message: {body}")
          time.sleep(SLEEP_SECS)
          sqs.delete_message(QueueUrl=QUEUE_URL, ReceiptHandle=rh)
          print("[worker] deleted message")
  EOF
  ```

- {% include step_label.html %} Crea el archivo `03-app/requirements.txt`.

  ```bash
  cat > 03-app/requirements.txt <<'EOF'
  boto3==1.35.0
  EOF
  ```

- {% include step_label.html %} Crea el archivo `03-app/Dockerfile`.

  ```bash
  cat > 03-app/Dockerfile <<'EOF'
  FROM python:3.11-slim
  WORKDIR /app
  COPY requirements.txt .
  RUN pip install --no-cache-dir -r requirements.txt
  COPY worker.py .
  CMD ["python", "worker.py"]
  EOF
  ```

- {% include step_label.html %} (Recomendado) Crea `.dockerignore` para builds más rápidos.

  ```bash
  cat > 03-app/.dockerignore <<'EOF'
  __pycache__/
  *.pyc
  .pytest_cache/
  .venv/
  EOF
  ```

#### Tarea 5.2 — Build + Push a ECR

- {% include step_label.html %} Crea el repositorio ECR si no existe.

  ```bash
  aws ecr describe-repositories --repository-names "$ECR_REPO" --region "$AWS_REGION" >/dev/null 2>&1 || aws ecr create-repository --repository-name "$ECR_REPO" --region "$AWS_REGION" >/dev/null
  ```
  ```bash
  aws ecr describe-repositories --repository-names "$ECR_REPO" --region "$AWS_REGION" --output table | tee outputs/05_ecr_repo.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Inicia sesión en ECR.

  ```bash
  aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Compila la imagen.

  ```bash
  export IMAGE_URI="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG"
  echo "IMAGE_URI=$IMAGE_URI" | tee outputs/05_image_uri.txt
  ```
  ```bash
  docker build -t keda-sqs-worker:"$IMAGE_TAG" 03-app
  ```
  {% include step_image.html %}

- {% include step_label.html %} Etiqueta la imagen.

  ```bash
  docker tag keda-sqs-worker:"$IMAGE_TAG" "$IMAGE_URI"
  ```

- {% include step_label.html %} Finalmente publica la imagen.

  ```bash
  docker push "$IMAGE_URI" | tee outputs/05_docker_push.txt
  ```

- {% include step_label.html %} Valida que la imagen existe en ECR.

  ```bash
  aws ecr describe-images --repository-name "$ECR_REPO" --region "$AWS_REGION" --output table | tee outputs/05_ecr_images.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Configurar KEDA (TriggerAuthentication + ScaledObject) y validar escalado (scale-to-zero)

Desplegarás el worker con **réplicas = 0**, crearás `TriggerAuthentication` (pod identity provider `aws`) y un `ScaledObject` que escala con base en la longitud de la cola SQS. Validarás que KEDA crea un **HPA** y que el escalado reacciona al enviar mensajes.

> **NOTA (CKAD):** Diagnóstico por capas: Deployment → Pods → HPA → KEDA CRDs → eventos/logs.
{: .lab-note .info .compact}

#### Tarea 6.1 — Deployment del worker (inicia en 0)

- {% include step_label.html %} Crea el archivo `04-keda/deployment-worker.yaml` (réplicas iniciales 0).

  ```bash
  cat > 04-keda/deployment-worker.yaml <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: sqs-worker
    namespace: $NS
    labels:
      app: sqs-worker
  spec:
    replicas: 0
    selector:
      matchLabels:
        app: sqs-worker
    template:
      metadata:
        labels:
          app: sqs-worker
      spec:
        serviceAccountName: sqs-worker-sa
        containers:
        - name: worker
          image: $IMAGE_URI
          env:
          - name: QUEUE_URL
            value: "$QUEUE_URL"
          - name: AWS_REGION
            value: "$AWS_REGION"
          - name: PROCESS_SECONDS
            value: "2"
          resources:
            requests:
              cpu: "50m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
  EOF
  ```

- {% include step_label.html %} Aplica el Deployment y valida que queda en 0 réplicas.

  ```bash
  kubectl apply -f 04-keda/deployment-worker.yaml
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get deploy sqs-worker -o wide | tee outputs/06_deploy_worker_initial.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get pods -l app=sqs-worker -o wide | tee outputs/06_pods_worker_initial.txt
  ```
  {% include step_image.html %}

#### Tarea 6.2 — TriggerAuthentication (pod identity provider aws)

- {% include step_label.html %} Crea el archivo `04-keda/triggerauth-aws.yaml`.

  > **IMPORTANTE:** El namespace debe ser el del lab (`$NS`). No dejes hardcode como `keda-demo` si cambiaste `NS`.
  {: .lab-note .important .compact}

  ```bash
  cat > 04-keda/triggerauth-aws.yaml <<EOF
  apiVersion: keda.sh/v1alpha1
  kind: TriggerAuthentication
  metadata:
    name: aws-pod-identity
    namespace: $NS
  spec:
    podIdentity:
      provider: aws
      identityOwner: keda
  EOF
  ```

- {% include step_label.html %} Aplica y valida el TriggerAuthentication.

  ```bash
  kubectl apply -f 04-keda/triggerauth-aws.yaml
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get triggerauthentication aws-pod-identity -o yaml | sed -n '1,160p' | tee outputs/06_triggerauth.yaml
  ```
  {% include step_image.html %}

#### Tarea 6.3 — ScaledObject para SQS

- {% include step_label.html %} Crea el archivo `04-keda/scaledobject-sqs.yaml`.

  ```bash
  cat > 04-keda/scaledobject-sqs.yaml <<EOF
  apiVersion: keda.sh/v1alpha1
  kind: ScaledObject
  metadata:
    name: sqs-worker-scaledobject
    namespace: $NS
  spec:
    scaleTargetRef:
      name: sqs-worker
    pollingInterval: 10
    cooldownPeriod: 30
    minReplicaCount: 0
    maxReplicaCount: 10
    triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: aws-pod-identity
      metadata:
        queueURL: "$QUEUE_URL"
        awsRegion: "$AWS_REGION"
        queueLength: "5"
        identityOwner: operator
        activationQueueLength: "1"
  EOF
  ```

- {% include step_label.html %} Aplica ScaledObject y valida que KEDA crea un HPA.

  ```bash
  kubectl apply -f 04-keda/scaledobject-sqs.yaml
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get scaledobject sqs-worker-scaledobject -o wide | tee outputs/06_scaledobject_get.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get hpa -o wide | tee outputs/06_hpa_get_initial.txt
  ```
  {% include step_image.html %}

#### Tarea 6.4 — Prueba de escalado (enviar mensajes y observar 0 → N)

- {% include step_label.html %} Abre una **segunda terminal** para observar (watch) el escalado.

  ```bash
  cd lab13
  source .env
  kubectl -n "$NS" get deploy -o wide -w
  ```
  {% include step_image.html %}

- {% include step_label.html %} En la **primera terminal**, envía 50 mensajes a SQS.

  ```bash
  for i in $(seq 1 50); do
    aws sqs send-message --queue-url "$QUEUE_URL" --message-body "msg-$i" --region "$AWS_REGION" >/dev/null
  done
  echo "Sent 50 messages to $QUEUE_NAME"
  ```
  {% include step_image.html %}
  {% include step_image.html %}

- {% include step_label.html %} (Evidencia) Captura el backlog aproximado en SQS.

  > **IMPORTANTE:** Espera a que termine de enviar los 50 mensajes y ejecuta el comando en la **primera terminal**
  {: .lab-note .important .compact}

  ```bash
  aws sqs get-queue-attributes --queue-url "$QUEUE_URL" --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible --region "$AWS_REGION" | tee outputs/06_sqs_attributes_after_send.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Cuando existan pods, sigue los logs del worker (confirmación de consumo y borrado).

  ```bash
  kubectl -n "$NS" logs deploy/sqs-worker -c worker --tail=50 -f
  ```
  {% include step_image.html %}

#### Tarea 6.5 — Validar retorno a 0 (N → 0) y checklist HPA

- {% include step_label.html %} Espera a que el backlog llegue a ~0 y captura evidencia.

  ```bash
  aws sqs get-queue-attributes --queue-url "$QUEUE_URL" --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible --region "$AWS_REGION" | tee outputs/06_sqs_attributes_after_processing.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura estado final de recursos (debe tender a 0 réplicas tras `cooldownPeriod`).

  ```bash
  kubectl -n "$NS" get deploy -o wide | tee outputs/06_final_get_resources.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" describe hpa | sed -n '1,260p' | tee outputs/06_final_describe_hpa.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" describe scaledobject sqs-worker-scaledobject | sed -n '1,260p' | tee outputs/06_final_describe_scaledobject.txt
  ```
  {% include step_image.html %}

#### Tarea 6.6 — Troubleshooting (si NO escala o NO consume)

  > **IMPORTANTE:** Sino hubo errores en las tareas pasadas solo ejecuta los comandos para revisar que todo este bien.
  {: .lab-note .important .compact}

- {% include step_label.html %} Revisa eventos del namespace (admission/webhook, image pull, crash loops).

  ```bash
  kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -n 60 | tee outputs/06_troubleshoot_events_tail.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el Deployment tiene SA/ENV/imagen correctas.

  ```bash
  kubectl -n "$NS" get deploy sqs-worker -o yaml | egrep -n "serviceAccountName|QUEUE_URL|AWS_REGION|image:" -A2 -B2 | tee outputs/06_troubleshoot_deploy_snippet.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Logs de KEDA (operator y metrics apiserver).

  ```bash
  kubectl -n keda logs deploy/keda-operator --tail=220 | tee outputs/06_keda_operator_logs.txt
  ```
  ```bash
  kubectl -n keda logs deploy/keda-operator-metrics-apiserver --tail=220 | tee outputs/06_keda_metrics_apiserver_logs.txt
  ```

- {% include step_label.html %} Describe HPA y ScaledObject para ver errores de autenticación, métricas o trigger.

  ```bash
  kubectl -n "$NS" get hpa -o wide | tee outputs/06_troubleshoot_hpa_get.txt
  ```
  ```bash
  kubectl -n "$NS" describe hpa | sed -n '1,320p' | tee outputs/06_troubleshoot_describe_hpa.txt
  ```
  ```bash
  kubectl -n "$NS" describe scaledobject sqs-worker-scaledobject | sed -n '1,320p' | tee outputs/06_troubleshoot_describe_scaledobject.txt
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Checklist final (CKAD) y evidencia

Ejecutarás un checklist final para confirmar que el sistema quedó bien configurado y dejarás evidencia reproducible del estado.

#### Tarea 7.1

- {% include step_label.html %} Checklist final de recursos del lab.

  ```bash
  kubectl -n "$NS" get deploy,rs,pods -o wide --show-labels | tee outputs/07_check_deploy_rs_pods.txt
  ```
  ```bash
  kubectl -n "$NS" get scaledobject,triggerauthentication -o wide | tee outputs/07_check_keda_objects.txt
  ```
  ```bash
  kubectl -n "$NS" get hpa -o wide | tee outputs/07_check_hpa.txt
  ```
  ```bash
  kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -n 60 | tee outputs/07_check_events_tail.txt
  ```

- {% include step_label.html %} Guarda variables clave en `outputs/env.sh` para limpieza reproducible.

  ```bash
  cat > outputs/env.sh <<EOF
  export AWS_REGION="$AWS_REGION"
  export CLUSTER_NAME="$CLUSTER_NAME"
  export NS="$NS"
  export LAB_ID="$LAB_ID"
  export QUEUE_NAME="$QUEUE_NAME"
  export QUEUE_URL="$QUEUE_URL"
  export QUEUE_ARN="$QUEUE_ARN"
  export POLICY_NAME="$POLICY_NAME"
  export POLICY_ARN="$POLICY_ARN"
  export ECR_REPO="$ECR_REPO"
  export IMAGE_TAG="$IMAGE_TAG"
  export IMAGE_URI="$IMAGE_URI"
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Limpieza del laboratorio

Eliminarás el namespace del lab, la cola SQS y el IAM ServiceAccount/Policy creados. Esto evita costos residuales.

> **IMPORTANTE:** Esta tarea elimina recursos. Asegúrate de estar en la cuenta/región correctas (ya quedó validado en la Tarea 1).
{: .lab-note .important .compact}

#### Tarea 8.1

- {% include step_label.html %} (Recomendado) Carga variables guardadas si cambiaste de terminal.

  ```bash
  test -f outputs/env.sh && source outputs/env.sh || true
  ```

- {% include step_label.html %} Elimina el namespace del laboratorio.

  ```bash
  kubectl delete ns "$NS" --ignore-not-found | tee outputs/08_cleanup_delete_ns.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina la cola SQS.

  ```bash
  aws sqs delete-queue --queue-url "$QUEUE_URL" --region "$AWS_REGION" | tee outputs/08_cleanup_delete_queue.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el iamserviceaccount (rol/stack) creado por eksctl.

  > **NOTA:** Si el namespace ya no existe, este comando puede fallar. Es aceptable; lo importante es borrar el stack/role.
  {: .lab-note .info .compact}

  ```bash
  eksctl delete iamserviceaccount --cluster "$CLUSTER_NAME" --region "$AWS_REGION" --namespace "$NS" --name sqs-worker-sa | tee outputs/08_cleanup_delete_iamserviceaccount.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina la IAM Policy creada (si es exclusiva del lab).

  ```bash
  for v in $(aws iam list-policy-versions --policy-arn "$POLICY_ARN" \
    --query 'Versions[?IsDefaultVersion==`false`].VersionId' --output text); do
    aws iam delete-policy-version --policy-arn "$POLICY_ARN" --version-id "$v"
  done
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el repositorio ECR (si es exclusivo del lab).

  > **IMPORTANTE:** Esto borra el repositorio y puede fallar si hay imágenes. Si quieres borrar imágenes también, usa `--force` (AWS CLI reciente).
  {: .lab-note .important .compact}

  ```bash
  # aws ecr delete-repository --repository-name "$ECR_REPO" --region "$AWS_REGION" --force
  aws ecr delete-repository --repository-name "$ECR_REPO" --region "$AWS_REGION" >/dev/null 2>&1 || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el namespace ya no existe.

  ```bash
  kubectl get ns | grep -w "$NS" && echo "Namespace aún existe (espera terminación)" || echo "OK: namespace eliminado" | tee outputs/08_cleanup_ns_check.txt
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