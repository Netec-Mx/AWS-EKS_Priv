---
layout: lab
title: "Práctica 8: Creación de NodePools dedicados con reglas de afinidad"
permalink: /lab8/lab8/
images_base: /labs/lab8/img
duration: "50 minutos"
objective:
  - "Crear NodePools dedicados (por ejemplo **critical** y **batch**) usando Karpenter, y dirigir workloads a esos pools mediante reglas de afinidad (**nodeAffinity / podAntiAffinity**), reforzadas con **taints/tolerations**. Validarás el comportamiento observando Pods **Pending/Unschedulable**, creación de nodos por pool, colocación correcta por etiquetas y distribución (anti-affinity), usando troubleshooting estilo CKAD con `kubectl describe`, `events`, `logs` y verificación de `labels/taints`."
prerequisites:
  - "Cuenta AWS con permisos para: EKS, IAM, EC2, CloudFormation y SQS."
  - "Windows + Visual Studio Code + Terminal GitBash."
  - "Herramientas: AWS CLI v2, kubectl, eksctl, helm 3, jq, git."
  - "Conectividad a internet para bajar charts/manifiestos."
introduction:
  - "Un **NodePool** define la “intención” de los nodos (labels, taints, requisitos de instancias, consolidación). Los Pods se colocan en esos nodos usando reglas de scheduling: **nodeAffinity** (selecciona nodos por labels) y **podAntiAffinity** (distribuye réplicas evitando co‑locación). **Taints/tolerations** son el “candado” que evita que cualquier Pod use un pool dedicado."
slug: lab8
lab_number: 8
final_result: >
  Al finalizar habrás definido NodePools dedicados (critical/batch) con labels + taints y dirigido workloads usando nodeAffinity y tolerations; además, habrás aplicado podAntiAffinity para distribuir réplicas entre nodos, observando el comportamiento real de scheduling (incluyendo un caso Pending intencional por falta de toleration) y el scale‑up de Karpenter para cumplir reglas duras. Todo respaldado con evidencia en describe/events/logs y verificación de labels/taints.
notes:
  - "CKAD: práctica fuerte de scheduling (nodeAffinity, podAntiAffinity, taints/tolerations) + troubleshooting de Pods Pending con kubectl describe pod y Events."
  - "Costos: este lab puede crear nodos EC2 (on‑demand/spot) y EBS/Network. Karpenter no se cobra aparte, pero sí los recursos que aprovisiona."
  - "Regla práctica: afinidad selecciona, taint/toleration bloquea/autoriza. Anti‑affinity required puede forzar scale‑up (y costo): úsala con intención."
references:
  - text: "Karpenter - NodePools (labels, taints, requirements, disruption, limits)"
    url: https://karpenter.sh/docs/concepts/nodepools/
  - text: "Karpenter - NodeClasses / EC2NodeClass (selectors por tags, role/AMI, status.*)"
    url: https://karpenter.sh/docs/concepts/nodeclasses/
  - text: "Kubernetes - Assign Pods to Nodes (nodeSelector/nodeAffinity)"
    url: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
  - text: "Kubernetes - Taints and Tolerations (NoSchedule, matching)"
    url: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
  - text: "Kubernetes - Pod Affinity and Anti-Affinity"
    url: https://www.strsistemas.com/blog/pod-affinity-antiaffinity-kubernetes
  - text: "AWS - Amazon EKS pricing (costo del control plane)"
    url: https://aws.amazon.com/eks/pricing/
prev: /lab7/lab7/
next: /lab9/lab9/
---

## Costo (resumen)

- **EKS** cobra por clúster por hora (estándar vs **Extended Support**).
- **Karpenter** no se cobra como servicio separado, pero **sí** generará recursos como **EC2 instances**, **EBS**, y (por arquitectura recomendada) **SQS** para interrupciones.

> **IMPORTANTE:** En esta práctica, las reglas `required` (afinidad/anti‑afinidad) pueden forzar a Karpenter a crear nodos adicionales. Mantén el demo corto y realiza cleanup al final.
{: .lab-note .important .compact}

---

### Tarea 1. Preparación del workspace y variables

Crearás la carpeta del laboratorio, abrirás GitBash en VS Code, validarás herramientas y dejarás variables listas para que los comandos sean reproducibles.

> **NOTA (CKAD):** Cuando un Pod queda Pending, tu secuencia base es: `kubectl describe pod` → leer **Events** (FailedScheduling) → validar **labels/affinity**, **taints/tolerations** y **requests**.
{: .lab-note .info .compact}

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo del curso con un usuario con permisos administrativos.

- {% include step_label.html %} Abre **Visual Studio Code**.

- {% include step_label.html %} Abre la terminal integrada en VS Code (ícono de terminal en la parte superior derecha).

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **Git Bash** como terminal.

  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de estar en la carpeta raíz de tus laboratorios (ej. `labs-eks-ckad`).

  > Si vienes de otro lab, usa `cd ..` hasta volver a la raíz.
  {: .lab-note .info .compact}

- {% include step_label.html %} Crea el directorio de trabajo del laboratorio y entra en él.

  ```bash
  mkdir lab08 && cd lab08
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea la estructura de carpetas estándar (evita errores de rutas).

  ```bash
  mkdir -p manifests iam logs outputs scripts notes
  ```

- {% include step_label.html %} Confirma que la estructura quedó creada.

  ```bash
  find . -maxdepth 2 -type d | sort
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea un archivo `.env` con variables base (sin credenciales), luego cárgalo.

  ```bash
  cat > .env <<'EOF'
  export AWS_REGION="us-west-2"

  # Karpenter
  export KARPENTER_NAMESPACE="karpenter"
  export KARPENTER_VERSION="1.7.4"

  # Demo
  export DEMO_NS="dedicated-pools"
  export NODECLASS_NAME="default"

  # NodePools dedicados
  export POOL_CRITICAL="critical"
  export POOL_BATCH="batch"
  EOF
  ```
  ```bash
  source .env
  ```

- {% include step_label.html %} Verifica herramientas (si alguna falla, corrige PATH/instalación antes de continuar).

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
  jq --version
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica identidad AWS (si falla, corrige credenciales/perfil antes de avanzar).

  ```bash
  aws sts get-caller-identity
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda tu `AWS_ACCOUNT_ID` para reutilizarlo (evita repetir comandos).

  ```bash
  export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  ```
  ```bash
  echo "AWS_ACCOUNT_ID=$AWS_ACCOUNT_ID"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. (Opcional) Crear un clúster EKS para esta práctica

Crea un clúster EKS con Managed Node Group usando `eksctl`. **Omite esta tarea** si ya tienes clúster y usarás su `CLUSTER_NAME`.

> **IMPORTANTE:** Crear clúster genera costos. Úsalo solo si no cuentas con uno de laboratorio.
{: .lab-note .important .compact}

#### Tarea 2.1

- {% include step_label.html %} (Opcional) Lista versiones soportadas en tu región para elegir `K8S_VERSION`.

  ```bash
  export AWS_REGION="us-west-2"
  ```
  ```bash
  eksctl utils describe-cluster-versions strings --region "$AWS_REGION" \
    | jq -r '.clusterVersions[].ClusterVersion'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de que la versión elegida aparece en la lista, usaremos la version **`1.33`**.

- {% include step_label.html %} Define variables del clúster.

  ```bash
  export CLUSTER_NAME="eks-nodepool-lab"
  export NODEGROUP_NAME="mng-1"
  export K8S_VERSION="1.33"
  ```

- {% include step_label.html %} Crea el clúster con un Managed Node Group.

  > **NOTA:** `eksctl` crea VPC/subnets/SG/roles y el clúster con un Managed Node Group listo para workloads.
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
    --node-type t3.medium \
    --nodes 2 --nodes-min 2 --nodes-max 3
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma desde AWS que el clúster quedó `ACTIVE`.

  ```bash
  aws eks describe-cluster \
    --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query "cluster.status" --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que existe al menos un NodeGroup asociado al clúster (esto indica que habrá nodos para el sistema).

  ```bash
  aws eks list-nodegroups \
    --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --output table
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asegura que el OIDC provider esté asociado (requisito para IRSA del controller de Karpenter).

  ```bash
  eksctl utils associate-iam-oidc-provider \
    --cluster "$CLUSTER_NAME" --region "$AWS_REGION" --approve
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el issuer OIDC exista.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
  --query "cluster.identity.oidc.issuer" --output text
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Conectar kubectl al clúster y validar conectividad

Esta tarea es obligatoria tanto si usas un clúster existente como si lo creaste en la Tarea 2.

#### Tarea 3.1

- {% include step_label.html %} Actualiza kubeconfig hacia el clúster objetivo.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma el contexto actual (evita trabajar en el clúster equivocado).

  ```bash
  kubectl config current-context
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el control plane responde.

  ```bash
  kubectl cluster-info
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista nodos y confirma que están `Ready`.

  ```bash
  kubectl get nodes -o wide
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Instalar y validar Karpenter (prerrequisitos + Helm)

Esta práctica es independiente: en esta tarea dejas Karpenter operativo (IAM/SQS + IRSA + controller + CRDs). Si tu clúster ya tiene Karpenter funcionando, podrás usar estos pasos como verificación (son idempotentes en gran medida).

> **IMPORTANTE:** El template de CloudFormation puede cambiar de ruta por versión. Si el `curl` falla, abre la referencia de CloudFormation en la documentación de Karpenter y toma el template correspondiente a tu versión.
{: .lab-note .important .compact}

#### Tarea 4.1 — Verificar modo de autenticación (Access Entries)

- {% include step_label.html %} Consulta el modo de autenticación del clúster (necesario para Access Entries).

  ```bash
  aws eks describe-cluster \
    --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query 'cluster.accessConfig.authenticationMode' --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si el modo es `CONFIG_MAP`, actualízalo a `API_AND_CONFIG_MAP` para poder usar Access Entries (recomendado).

  > **IMPORTANTE:** Es normal que aparezca el mensaje de error, porque ya existe el metodo de autenticación correcto. avanza al siguiente paso.
  {: .lab-note .important .compact}

  ```bash
  aws eks update-cluster-config \
    --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --access-config authenticationMode=API_AND_CONFIG_MAP
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que el clúster regresó a estado `ACTIVE` tras el cambio (si está `UPDATING`, espera).

  ```bash
  aws eks describe-cluster \
    --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query 'cluster.status' --output text
  ```
  {% include step_image.html %}

#### Tarea 4.2 — OIDC (IRSA) + discovery tags en subnets/SG

- {% include step_label.html %} Taggea subnets del clúster con `karpenter.sh/discovery=$CLUSTER_NAME` (para selectors por tags en EC2NodeClass).

  > **Tip:** En entornos serios, **taggea solo subnets privadas**. Para el laboratorio, taggear los subnets del clúster es suficiente.
  {: .lab-note .info .compact}

  ```bash
  SUBNETS="$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query "cluster.resourcesVpcConfig.subnetIds[]" --output text)"
  ```
  ```bash
  aws ec2 create-tags --region "$AWS_REGION" \
    --resources $SUBNETS \
    --tags Key=karpenter.sh/discovery,Value="$CLUSTER_NAME"
  ```
  {% include step_image.html %}

- {% include step_label.html %} **Taggea el Cluster Security Group** con el mismo discovery tag.

  ```bash
  CLUSTER_SG="$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)"
  ```
  ```bash
  aws ec2 create-tags --region "$AWS_REGION" \
    --resources "$CLUSTER_SG" \
    --tags Key=karpenter.sh/discovery,Value="$CLUSTER_NAME"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que los tags existen (si no aparecen, Karpenter no podrá descubrir subnets/SG).

  ```bash
  aws ec2 describe-tags --region "$AWS_REGION" \
    --filters "Name=key,Values=karpenter.sh/discovery" "Name=value,Values=$CLUSTER_NAME" \
    --query "Tags[].{ResourceId:ResourceId,Key:Key,Value:Value}" --output table
  ```
  {% include step_image.html %}

#### Tarea 4.3 — CloudFormation (IAM + SQS interruption queue)

- {% include step_label.html %} Descarga el template oficial de CloudFormation para tu versión.

  ```bash
  export CFN_TEMPLATE="iam/karpenter-cloudformation.yaml"
  ```
  ```bash
  curl -fsSL \
    "https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml" \
    > "$CFN_TEMPLATE"
  ```
  ```bash
  wc -l "$CFN_TEMPLATE"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Despliega el stack (crea/actualiza roles/policies + SQS).

  > **NOTA:** La creación del stack puede durar **2 minutos**
  {: .lab-note .info .compact}

  ```bash
  aws cloudformation deploy \
    --stack-name "Karpenter-${CLUSTER_NAME}" \
    --template-file "$CFN_TEMPLATE" \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameter-overrides "ClusterName=${CLUSTER_NAME}"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que el stack quedó en estado correcto.

  ```bash
  aws cloudformation describe-stacks \
    --stack-name "Karpenter-${CLUSTER_NAME}" \
    --query "Stacks[0].StackStatus" --output text
  ```
  {% include step_image.html %}

#### Tarea 4.4 — Access Entry para Node Role (nodos creados por Karpenter)

- {% include step_label.html %} Define el ARN del Node Role creado por CloudFormation.

  ```bash
  export NODE_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}"
  ```
  ```bash
  echo "NODE_ROLE_ARN=$NODE_ROLE_ARN"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el Access Entry para el Node Role (si ya existe, es aceptable).

  ```bash
  aws eks create-access-entry \
    --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --principal-arn "$NODE_ROLE_ARN" \
    --type EC2_LINUX || echo "Access entry ya existe o no es necesario (verifica con list-access-entries)."
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el Access Entry aparece listado (sin esto, los nodos nuevos pueden no unirse).

  ```bash
  aws eks list-access-entries \
    --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --output table
  ```
  {% include step_image.html %}

#### Tarea 4.5 — Obtener ARN/IDs (rol del controller + cola SQS)

- {% include step_label.html %} Exporta los outputs del stack para identificar el rol IAM del controller y el nombre de la SQS.

  ```bash
  export STACK="Karpenter-${CLUSTER_NAME}"
  echo $STACK
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén la SQS Queue URL del stack y deriva el QueueName.

  ```bash
  KARPENTER_SQS_QUEUE_URL="$(aws cloudformation list-stack-resources --stack-name "$STACK" \
    --query "StackResourceSummaries[?ResourceType=='AWS::SQS::Queue'].PhysicalResourceId | [0]" \
    --output text)"
  ```
  ```bash
  export KARPENTER_SQS_QUEUE="$(echo "$KARPENTER_SQS_QUEUE_URL" | awk -F/ '{print $NF}')"
  ```
  ```bash
  echo "KARPENTER_SQS_QUEUE_URL=$KARPENTER_SQS_QUEUE_URL"
  echo "KARPENTER_SQS_QUEUE=$KARPENTER_SQS_QUEUE"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén el ARN de la ManagedPolicy creada por el stack (controller policy).

  ```bash
  export KARPENTER_CONTROLLER_POLICY_ARN="$(aws cloudformation list-stack-resources --stack-name "$STACK" \
    --query "StackResourceSummaries[?ResourceType=='AWS::IAM::ManagedPolicy'].PhysicalResourceId | [0]" \
    --output text)"
  ```
  ```bash
  echo "KARPENTER_CONTROLLER_POLICY_ARN=$KARPENTER_CONTROLLER_POLICY_ARN"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén el endpoint del clúster (valor recomendado para Helm).

  ```bash
  export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
  --query "cluster.endpoint" --output text)"
  ```
  ```bash
  echo "CLUSTER_ENDPOINT=$CLUSTER_ENDPOINT"
  ```
  {% include step_image.html %}

#### Tarea 4.6 — Crear ServiceAccount con IRSA (rol del controller)

- {% include step_label.html %} Crea/actualiza el ServiceAccount karpenter con IRSA (eksctl crea el role y deja la anotación).

  > **NOTA:** La creación puede tardar **2 minutos**
  {: .lab-note .info .compact}

  ```bash
  eksctl create iamserviceaccount \
  --cluster "$CLUSTER_NAME" --region "$AWS_REGION" \
  --namespace "$KARPENTER_NAMESPACE" \
  --name karpenter \
  --attach-policy-arn "$KARPENTER_CONTROLLER_POLICY_ARN" \
  --approve \
  --override-existing-serviceaccounts
  ```
  {% include step_image.html %}

- {% include step_label.html %} Extrae el Role ARN desde la anotación del ServiceAccount.

  ```bash
  export KARPENTER_IAM_ROLE_ARN="$(kubectl -n "$KARPENTER_NAMESPACE" get sa karpenter \
  -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}')"
  ```
  ```bash
  echo "KARPENTER_IAM_ROLE_ARN=$KARPENTER_IAM_ROLE_ARN"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el SA quedó anotado (prueba de IRSA).

  ```bash
  kubectl -n "$KARPENTER_NAMESPACE" get sa karpenter -o yaml | sed -n '1,120p'
  ```
  {% include step_image.html %}

#### Tarea 4.7 — Instalar CRDs y controlador

- {% include step_label.html %} Asegura que Helm no tenga sesión autenticada contra `public.ecr.aws` (pull público).

  ```bash
  helm registry logout public.ecr.aws >/dev/null 2>&1 || true
  ```

- {% include step_label.html %} Instala/actualiza el chart de **CRDs** (esto evita errores por CRDs faltantes).

  ```bash
  helm upgrade --install karpenter-crd oci://public.ecr.aws/karpenter/karpenter-crd \
    --version "${KARPENTER_VERSION}" \
    --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
    --wait
  ```
  {% include step_image.html %}

- {% include step_label.html %} Instala/actualiza el controlador Karpenter con IRSA (anota el ServiceAccount con el rol IAM).

  ```bash
  helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
    --version "${KARPENTER_VERSION}" \
    --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
    --set "settings.clusterName=${CLUSTER_NAME}" \
    --set "settings.clusterEndpoint=${CLUSTER_ENDPOINT}" \
    --set "settings.interruptionQueue=${KARPENTER_SQS_QUEUE}" \
    --set serviceAccount.create=false \
    --set serviceAccount.name=karpenter \
    --set controller.resources.requests.cpu=1 \
    --set controller.resources.requests.memory=1Gi \
    --set controller.resources.limits.cpu=1 \
    --set controller.resources.limits.memory=1Gi \
    --wait
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el deployment esté Ready **(si no está Ready, aún no continúes con NodePool/NodeClass).**

  ```bash
  kubectl get deploy -n "${KARPENTER_NAMESPACE}"
  ```
  ```bash
  kubectl rollout status deploy/karpenter -n "${KARPENTER_NAMESPACE}"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el ServiceAccount tiene la anotación del rol IAM (esto prueba que IRSA quedó aplicado).

  ```bash
  kubectl get sa -n "${KARPENTER_NAMESPACE}" karpenter -o yaml | sed -n '1,120p'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa logs recientes buscando errores típicos (AccessDenied, STS, errores de SQS).

  ```bash
  kubectl logs -n "${KARPENTER_NAMESPACE}" deploy/karpenter --tail=200 | tee logs/karpenter_tail_install.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Crear EC2NodeClass funcional y NodePools dedicados (critical/batch)

Crearás (si no existe) un **EC2NodeClass** funcional (`default`) y luego dos NodePools dedicados:

- **critical**: nodos **on‑demand** dedicados para cargas sensibles.
- **batch**: nodos **spot** (más económicos) dedicados para trabajos tolerantes (si no quieres Spot, cámbialo a on-demand).

Ambos tendrán:
- **labels** (`dedicated=...`) para que la afinidad los seleccione, y
- **taints** (`dedicated=...:NoSchedule`) para que *solo* entren Pods con tolerations.

> **IMPORTANTE (Spot):** Si tu cuenta/región no tiene capacidad Spot o quieres simplificar, cambia el pool `batch` a `on-demand`.
{: .lab-note .important .compact}

#### Tarea 5.1 — Namespace del demo

- {% include step_label.html %} Crea el namespace del demo.

  ```bash
  kubectl create ns "$DEMO_NS" || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el namespace existe (evita aplicar manifiestos a un namespace inexistente).

  ```bash
  kubectl get ns "$DEMO_NS"
  ```
  {% include step_image.html %}

#### Tarea 5.2 — EC2NodeClass (si no existe)

- {% include step_label.html %} Verifica si ya existe un EC2NodeClass con el nombre definido en `NODECLASS_NAME`.

  ```bash
  kubectl get ec2nodeclass "$NODECLASS_NAME" 2>/dev/null || echo "No existe EC2NodeClass $NODECLASS_NAME (se creará)."
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea `manifests/ec2nodeclass-default.yaml` (solo si no existe) usando selectors por tags de discovery.

  ```bash
  cat > manifests/ec2nodeclass-default.yaml <<EOF
  apiVersion: karpenter.k8s.aws/v1
  kind: EC2NodeClass
  metadata:
    name: ${NODECLASS_NAME}
  spec:
    role: "KarpenterNodeRole-${CLUSTER_NAME}"
    amiSelectorTerms:
      - alias: "al2023@latest"
    subnetSelectorTerms:
      - tags:
          karpenter.sh/discovery: "${CLUSTER_NAME}"
    securityGroupSelectorTerms:
      - tags:
          karpenter.sh/discovery: "${CLUSTER_NAME}"
  EOF
  ```

- {% include step_label.html %} Aplica el EC2NodeClass (si ya existía, este apply es idempotente).

  ```bash
  kubectl apply -f manifests/ec2nodeclass-default.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Describe el EC2NodeClass para detectar errores de selectors/tags.

  ```bash
  kubectl describe ec2nodeclass "$NODECLASS_NAME" | sed -n '1,220p' | tee outputs/ec2nodeclass_describe.txt
  ```
  {% include step_image.html %}

#### Tarea 5.3 — NodePools dedicados

- {% include step_label.html %} Crea el manifiesto `manifests/nodepools-dedicated.yaml` (NodePools `critical` y `batch` referenciando tu EC2NodeClass).

  ```bash
  cat > manifests/nodepools-dedicated.yaml <<EOF
  apiVersion: karpenter.sh/v1
  kind: NodePool
  metadata:
    name: ${POOL_CRITICAL}
  spec:
    template:
      metadata:
        labels:
          dedicated: "${POOL_CRITICAL}"
      spec:
        taints:
          - key: dedicated
            value: ${POOL_CRITICAL}
            effect: NoSchedule
        requirements:
          - key: karpenter.sh/capacity-type
            operator: In
            values: ["on-demand"]
        nodeClassRef:
          group: karpenter.k8s.aws
          kind: EC2NodeClass
          name: ${NODECLASS_NAME}
    disruption:
      consolidationPolicy: WhenEmptyOrUnderutilized
      consolidateAfter: 1m
  ---
  apiVersion: karpenter.sh/v1
  kind: NodePool
  metadata:
    name: ${POOL_BATCH}
  spec:
    template:
      metadata:
        labels:
          dedicated: "${POOL_BATCH}"
      spec:
        taints:
          - key: dedicated
            value: ${POOL_BATCH}
            effect: NoSchedule
        requirements:
          - key: karpenter.sh/capacity-type
            operator: In
            values: ["on-demand"]
        nodeClassRef:
          group: karpenter.k8s.aws
          kind: EC2NodeClass
          name: ${NODECLASS_NAME}
    disruption:
      consolidationPolicy: WhenEmptyOrUnderutilized
      consolidateAfter: 1m
  EOF
  ```

- {% include step_label.html %} Aplica los NodePools.

  ```bash
  kubectl apply -f manifests/nodepools-dedicated.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que los NodePools existen (aún no crearán nodos si no hay Pods que lo requieran).

  ```bash
  kubectl get nodepool
  ```
  {% include step_image.html %}

- {% include step_label.html %} Describe ambos NodePools para confirmar labels/taints y requisitos.

  ```bash
  kubectl describe nodepool "$POOL_CRITICAL" | sed -n '1,220p' | tee outputs/nodepool_critical_describe.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl describe nodepool "$POOL_BATCH" | sed -n '1,220p' | tee outputs/nodepool_batch_describe.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Workloads con nodeAffinity + tolerations (y un Pending intencional)

Crearás 3 Deployments:

- `api-critical` → debe caer en **critical** (afinidad + toleration).
- `job-batch` → debe caer en **batch** (afinidad + toleration).
- `oops-critical` → **mal configurado** (afinidad sin toleration) para provocar **Pending** y practicar diagnóstico.

#### Tarea 6.1 — Crear y aplicar workloads

- {% include step_label.html %} Crea `manifests/workloads-affinity.yaml` (usa tu `DEMO_NS` y pools definidos).

  ```bash
  cat > manifests/workloads-affinity.yaml <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: api-critical
    namespace: ${DEMO_NS}
  spec:
    replicas: 2
    selector:
      matchLabels: { app: api-critical }
    template:
      metadata:
        labels: { app: api-critical }
      spec:
        tolerations:
          - key: "dedicated"
            operator: "Equal"
            value: "${POOL_CRITICAL}"
            effect: "NoSchedule"
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: dedicated
                  operator: In
                  values: ["${POOL_CRITICAL}"]
        containers:
        - name: app
          image: traefik/whoami:v1.10.3
          ports: [{containerPort: 80}]
          resources:
            requests:
              cpu: "250m"
              memory: "128Mi"
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: job-batch
    namespace: ${DEMO_NS}
  spec:
    replicas: 3
    selector:
      matchLabels: { app: job-batch }
    template:
      metadata:
        labels: { app: job-batch }
      spec:
        tolerations:
          - key: "dedicated"
            operator: "Equal"
            value: "${POOL_BATCH}"
            effect: "NoSchedule"
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: dedicated
                  operator: In
                  values: ["${POOL_BATCH}"]
        containers:
        - name: app
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.9
          resources:
            requests:
              cpu: "200m"
              memory: "128Mi"
  ---
  # ERROR INTENCIONAL: Afinidad a critical, pero NO tolera el taint -> quedará Pending
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: oops-critical
    namespace: ${DEMO_NS}
  spec:
    replicas: 1
    selector:
      matchLabels: { app: oops-critical }
    template:
      metadata:
        labels: { app: oops-critical }
      spec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: dedicated
                  operator: In
                  values: ["${POOL_CRITICAL}"]
        containers:
        - name: app
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.9
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
  EOF
  ```

- {% include step_label.html %} Aplica los workloads.

  ```bash
  kubectl apply -f manifests/workloads-affinity.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa Pods (verás `Pending` al inicio si no existe capacidad dedicada aún).

  ```bash
  kubectl -n "$DEMO_NS" get pods -o wide -w
  ```
  {% include step_image.html %}

- {% include step_label.html %} **En otra terminal**, observa nodos mostrando el label `dedicated` (te ayuda a identificar nodos por pool).

  ```bash
  kubectl get nodes -L dedicated -o wide -w
  ```
  {% include step_image.html %}

#### Tarea 6.2 — Diagnóstico y fix del error intencional

- {% include step_label.html %} Identifica el Pod de `oops-critical` y confirma que quedó Pending.

  > **IMPORTANTE:** Regresa a la primera terminal, ejecuta **`CRL+C`** para aplicar los siguientes comandos.
  {: .lab-note .important .compact}

  ```bash
  kubectl -n "$DEMO_NS" get pods -l app=oops-critical
  ```
  ```bash
  POD_OOPS="$(kubectl -n "$DEMO_NS" get pod -l app=oops-critical -o jsonpath='{.items[0].metadata.name}')"
  ```
  ```bash
  echo "POD_OOPS=$POD_OOPS"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Diagnostica con `describe` y lee los Events (deberías ver taint no tolerado / FailedScheduling).

  ```bash
  kubectl -n "$DEMO_NS" describe pod "$POD_OOPS" | sed -n '1,280p' | tee outputs/oops_describe_before_fix.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa los eventos recientes del namespace para capturar la razón exacta.

  ```bash
  kubectl -n "$DEMO_NS" get events --sort-by=.lastTimestamp | tail -n 60 | tee outputs/task6_events_before_fix.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ejecuta el siguiente comando para que karpenter haga un nodo más grande (con más pod capacity).

  ```bash
  kubectl patch nodepool "$POOL_CRITICAL" --type='json' -p='[
    {"op":"add","path":"/spec/template/spec/requirements/-","value":{
      "key":"karpenter.k8s.aws/instance-size",
      "operator":"NotIn",
      "values":["nano","micro"]
    }}
  ]'
  ```

- {% include step_label.html %} Corrige el Deployment agregando la toleration faltante (patch JSON) y espera el rollout.

  ```bash
  kubectl -n "$DEMO_NS" patch deploy oops-critical --type='json' -p='[
    {"op":"add","path":"/spec/template/spec/tolerations","value":[
      {"key":"dedicated","operator":"Equal","value":"critical","effect":"NoSchedule"}
    ]}
  ]'
  ```
  ```bash
  kubectl -n "$DEMO_NS" rollout status deploy/oops-critical
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que `oops-critical` ya corre y observa en qué nodo quedó programado.

  ```bash
  kubectl -n "$DEMO_NS" get pods -l app=oops-critical -o wide | tee outputs/oops_after_fix.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura un snapshot de colocación de Pods/Nodos para todo el demo.

  ```bash
  kubectl -n "$DEMO_NS" get pods -o wide | tee outputs/task6_pods_wide_after_fix.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Afinidad avanzada: distribuir réplicas con podAntiAffinity (required)

Forzarás que las réplicas de `api-critical` se distribuyan en **nodos distintos** evitando que dos réplicas queden en el mismo nodo (topologyKey: `kubernetes.io/hostname`). Esto incrementa resiliencia ante falla de nodo.

> **NOTA (CKAD):** Anti‑affinity `required` es *duro* (puede dejar Pods Pending). Si no hay nodos suficientes, esto puede provocar **scale‑up**.
{: .lab-note .info .compact}

#### Tarea 7.1

- {% include step_label.html %} Aplica `podAntiAffinity` (required) al Deployment `api-critical` y espera el rollout.

  ```bash
  kubectl -n "$DEMO_NS" patch deploy api-critical --type='merge' -p "
  spec:
    template:
      spec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: dedicated
                  operator: In
                  values: [\"${POOL_CRITICAL}\"]
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: api-critical
              topologyKey: kubernetes.io/hostname
  "
  ```
  ```bash
  kubectl -n "$DEMO_NS" rollout status deploy/api-critical
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa Pods `api-critical`: si solo existía 1 nodo critical, una réplica puede quedar Pending y Karpenter deberá crear otro nodo.

  ```bash
  kubectl -n "$DEMO_NS" get pods -l app=api-critical -o wide -w
  ```
  {% include step_image.html %}

- {% include step_label.html %} **En otra terminal**, observa si aparecen nodos `dedicated=critical` adicionales (evidencia de scale‑up).

  ```bash
  kubectl get nodes -l dedicated="${POOL_CRITICAL}" -L dedicated -o wide -w
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura evidencia de distribución por hostname (cada réplica debe tener `NODE` distinto).

  ```bash
  kubectl -n "$DEMO_NS" get pod -l app=api-critical -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,READY:.status.containerStatuses[0].ready
  ```
  ```bash
  kubectl -n "$DEMO_NS" get pod -l app=api-critical -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}' | tee outputs/task7_api_placement.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda eventos recientes (si hubo Pending, aquí queda la causa y el momento del cambio).

  ```bash
  kubectl -n "$DEMO_NS" get events --sort-by=.lastTimestamp | tail -n 60 | tee outputs/task7_events.txt
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Evidencias finales

Generarás evidencias rápidas de las actividades de la practica

#### Tarea 8.1

- {% include step_label.html %} Guarda tabla de colocación Pods/Nodos (fuente de verdad del scheduling).

  ```bash
  kubectl -n "$DEMO_NS" get pods -o wide | tee outputs/final_pods_wide.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda lista de nodos con labels `dedicated` (para identificar qué pool creó qué nodos).

  ```bash
  kubectl get nodes -L dedicated -o wide | tee outputs/final_nodes_labeled.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura taints/labels de un nodo critical (confirma el “candado” NoSchedule).

  ```bash
  NODEC="$(kubectl get nodes -l dedicated="${POOL_CRITICAL}" -o jsonpath='{.items[0].metadata.name}')"
  ```
  ```bash
  echo "NODEC=$NODEC"
  ```
  ```bash
  kubectl describe node "$NODEC" | sed -n '1,260p' | egrep -n "Taints|Labels|dedicated" | tee outputs/final_node_taints_labels.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 9. Limpieza (evitar costos)

Eliminaras los objetos creados y el cluster de Amazon EKS.

> **Tip (costos):** Tras el cleanup, espera unos minutos para que Karpenter consolide y retire nodos vacíos (según `consolidateAfter`).
{: .lab-note .info .compact}

#### Tarea 9.1 

- {% include step_label.html %} (Opcional) Elimina NodePool/EC2NodeClass si NO los reutilizarás.

  ```bash
  kubectl delete -f manifests/ec2nodeclass-default.yaml --ignore-not-found
  kubectl delete -f manifests/nodepools-dedicated.yaml --ignore-not-found
  kubectl delete -f manifests/workloads-affinity.yaml --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el workload de prueba y el namespace.

  ```bash
  kubectl delete ns karpenter --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} Recolecta el rol del stack a eliminar.

   ```bash
  STACK="Karpenter-eks-nodepool-lab"

  ROLE_NAME="$(aws cloudformation list-stack-resources \
    --stack-name "$STACK" \
    --query "StackResourceSummaries[?LogicalResourceId=='KarpenterNodeRole'].PhysicalResourceId" \
    --output text)"

  echo "ROLE_NAME=$ROLE_NAME"
  ```

- {% include step_label.html %} Encuentra el Instance Profile que lo está usando.

   ```bash
  aws iam list-instance-profiles-for-role \
    --role-name "$ROLE_NAME" \
    --query "InstanceProfiles[].InstanceProfileName" \
    --output text
  ```

- {% include step_label.html %} Guarda el nombre (si sale más de uno, hazlo para cada uno).

   ```bash
  INSTANCE_PROFILE="$(aws iam list-instance-profiles-for-role \
    --role-name "$ROLE_NAME" \
    --query "InstanceProfiles[0].InstanceProfileName" \
    --output text)"

  echo "INSTANCE_PROFILE=$INSTANCE_PROFILE"
  ```

- {% include step_label.html %} Quita el role del instance profile.

   ```bash
  aws iam remove-role-from-instance-profile \
    --instance-profile-name "$INSTANCE_PROFILE" \
    --role-name "$ROLE_NAME"
  ```
  ```bash
  aws iam delete-instance-profile --instance-profile-name "$INSTANCE_PROFILE"
  ```

- {% include step_label.html %} Elimina el clúster creado por `eksctl`.

  > **NOTA:** El cluster tardara **9 minutos** aproximadamente en eliminarse.
  {: .lab-note .info .compact}

  ```bash
  eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Elimina el stack CloudFormation (IAM/SQS).

  ```bash
  aws cloudformation delete-stack --stack-name "Karpenter-${CLUSTER_NAME}"
  ```

- {% include step_label.html %} Verifica que ya no queden stacks o esten en **DELETE_IN_PROGRESS**

  ```bash
  aws cloudformation describe-stacks \
  --query "Stacks[].{StackName:StackName,Status:StackStatus,Created:CreationTime,Updated:LastUpdatedTime}" \
  --output table
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el cluster se haya eliminado correctamente.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" >/dev/null 2>&1 || echo "Cluster eliminado"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[8] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}