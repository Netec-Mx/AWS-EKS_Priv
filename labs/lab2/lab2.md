---
layout: lab
title: "Práctica 2: Implementación de ArgoCD en EKS con un despliegue GitOps"
permalink: /lab2/lab2/
images_base: /labs/lab2/img
duration: "75 minutos"
objective:
  - "Instalar ArgoCD en un clúster Amazon EKS y habilitar un flujo GitOps completo: Git como source of truth, una app Kubernetes definida con Kustomize (base + overlay dev) y una Application de Argo CD con auto-sync, prune y self-heal, validando despliegues por commit y corrección automática de drift."
prerequisites:
  - "Clúster Amazon EKS accesible con kubectl."
  - "Windows + Visual Studio Code + Terminal GitBash."
  - "Herramientas: AWS CLI v2, kubectl, eksctl, git y Docker Desktop."
  - "Permisos AWS para EKS (lectura/escritura básica) y ECR (crear repositorio, push/pull)."
  - "Repositorio Git remoto (recomendado GitHub). Para evitar fricción, usa un repositorio público en esta práctica."
introduction:
  - "ArgoCD implementa GitOps reconciliando el estado del clúster con lo declarado en Git. En esta práctica ejecutarás el pipeline Docker → ECR → Manifiestos en Git (Kustomize) → ArgoCD Application → Auto-sync (prune/self-heal) y comprobarás que Argo CD despliega por commit y corrige el drift si alguien cambia recursos a mano dentro del clúster."
slug: lab2
lab_number: 2
final_result: >
  Al finalizar, Argo CD quedará instalado en EKS y administrará una aplicación desplegada desde tu repositorio Git (Kustomize overlay dev) con sincronización automática, prune y self-heal, demostrando que el despliegue ocurre por commit y que cualquier cambio manual en el clúster se revierte para volver al estado definido en Git.
notes:
  - "En laboratorios, accede a Argo CD por port-forward para evitar costos de LoadBalancer/Ingress."
  - "Por defecto, Argo CD usa certificado self-signed; el navegador mostrará advertencia (normal en lab)."
  - "GitHub ya no permite contraseña por HTTPS para push: usa un Personal Access Token (PAT) o SSH."
  - "Si el repo fuera privado, Argo CD necesitará credenciales (Repository Secret/Config) para poder leerlo. En esta práctica usamos repo público para simplificar."
references:
  - text: "Argo CD - Getting Started (instalación, port-forward, password inicial, borrado del secret)"
    url: https://argo-cd.readthedocs.io/en/stable/getting_started/
  - text: "Argo CD - Declarative Setup (Application spec)"
    url: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/
  - text: "Argo CD - Automated Sync (prune/self-heal)"
    url: https://argo-cd.readthedocs.io/en/latest/user-guide/auto_sync/
  - text: "Kubernetes - Kustomize (overview)"
    url: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
  - text: "Amazon ECR Pricing"
    url: https://aws.amazon.com/ecr/pricing/
  - text: "Amazon EKS Pricing"
    url: https://aws.amazon.com/eks/pricing/
  - text: "Elastic Load Balancing Pricing"
    url: https://aws.amazon.com/elasticloadbalancing/pricing/
prev: /lab1/lab1/
next: /lab3/lab3/
---

---

### Tarea 1. Preparar la carpeta de la práctica

Confirmarás identidad/cuenta/región, asegurarás que `kubectl` apunta al clúster correcto, y dejarás listas variables de ECR y Git para el flujo GitOps.

> **IMPORTANTE:** GitOps implica que **Git manda**. Evita aplicar cambios “a mano” en producción. Aquí lo haremos a propósito solo para demostrar **drift + self-heal**.
{: .lab-note .important .compact}

> **NOTA (CKAD):** GitOps refuerza Deployments, Services, ConfigMaps, Namespaces, probes, rollouts, troubleshooting (desired vs actual) y Kustomize (base/overlays).
{: .lab-note .info .compact}

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo asignado al curso con el usuario que tenga permisos administrativos.

- {% include step_label.html %} Abre el **`Visual Studio Code`**. Lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- {% include step_label.html %} Una vez abierto **VSCode**, da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  {% include step_image.html %}

- {% include step_label.html %} Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de estar dentro de la carpeta del curso llamada **labs-eks-ckad** en la terminal de **VSCode**.

  > **Nota.** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para retornar a la raíz del directorio los laboratorios.
  {: .lab-note .info .compact}

- {% include step_label.html %} Ahora crea el directorio para trabajar en la Práctica 2.

  ```bash
  mkdir lab02 && cd lab02
  ```

- {% include step_label.html %} Valida en el **Explorador** de archivos dentro de **VSCode** que se haya creado el directorio.

  {% include step_image.html %}

- {% include step_label.html %} Ejecuta el siguiente comando para crear el resto de los archivos y directorios de la practica

  ```bash
  mkdir -p app gitops argocd scripts outputs
  mkdir -p gitops/base gitops/overlay
  mkdir -p gitops/overlay/dev
  ```

- {% include step_label.html %} Confirma la estructura de los directorios creados, ejecuta el siguiente comando

  > **Nota.** Organizar cada práctica en carpetas separadas facilita la gestión de ejemplos y evita confusiones.
  {: .lab-note .info .compact}

  ```bash
  find . -maxdepth 3 -type d | sort
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear un clúster EKS para la práctica

Crearás un clúster EKS con Managed Node Group usando `eksctl`. Listo para instalar Argo CD.

> **IMPORTANTE:** Si ya tienes un clúster EKS puedes avanzar al **Paso 17** solo ajusta las variables necesarias para conectarte al cluster.
{: .lab-note .important .compact}

#### Tara 2.1

- {% include step_label.html %} (Recomendado) Revisa versiones soportadas en la región antes de elegir `K8S_VERSION`.

  ```bash
  export AWS_REGION="us-west-2"
  ```
  ```bash
  eksctl utils describe-cluster-versions strings --region "$AWS_REGION" \
    | jq -r '.clusterVersions[].ClusterVersion'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de que la versión elegida aparece en la lista, usaremos la version **`1.33`**.

- {% include step_label.html %} Define las variables del clúster (región/nombres/versión) y tamaño mínimo de nodos.

  ```bash
  export AWS_REGION="us-west-2"
  export CLUSTER_NAME="eks-gitops-lab"
  export NODEGROUP_NAME="mng-1"
  export K8S_VERSION="1.33"
  ```

- {% include step_label.html %} Ejecuta el siguiente comando para validar el guardado correcto de las variables.

  ```bash
  echo "AWS_REGION=$AWS_REGION"
  echo "CLUSTER_NAME=$CLUSTER_NAME"
  echo "NODEGROUP_NAME=$NODEGROUP_NAME"
  echo "K8S_VERSION=$K8S_VERSION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica la identidad de AWS (reduce riesgo de crear recursos en la cuenta equivocada).

  ```bash
  aws sts get-caller-identity
  aws configure get region || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que el `Account` y `Arn` sean los que se te asignaron al curso.

  > **IMPORTANTE:** Recuerda que el usuario y la cuenta pueden ser diferentes.
  {: .lab-note .important .compact}

- {% include step_label.html %} Crea el clúster EKS con Managed Node Group (2 nodos) usando `eksctl`.

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

- {% include step_label.html %} Configura kubeconfig y el contexto para entrar a kubernetes.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  kubectl config current-context
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida la conectividad a Kubernetes.

  ```bash
  kubectl cluster-info
  kubectl get nodes -o wide
  kubectl get ns
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el cluster y el node group este correctamente creado, con los siguientes comandos.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.status" --output text
  ```
  ```bash
  aws eks describe-nodegroup --cluster-name "$CLUSTER_NAME" --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION" --query "nodegroup.status" --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Recomendado) Guarda evidencia inicial en el directorio de `outputs/`, solo como referencia.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.version" --output text | tee outputs/cluster_version.txt
  ```
  ```bash
  kubectl get nodes -o wide | tee outputs/nodes.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Preparación del repositorio y variables del entorno

Confirmarás identidad/cuenta/región, asegurarás que `kubectl` apunta al clúster correcto, y dejarás listas variables de ECR y Git para el flujo GitOps.

#### Tarea 3.1

- {% include step_label.html %} Confirma la identidad de AWS y define la región (si ya definiste en la tarea anterior, solo valida).

  ```bash
  aws sts get-caller-identity
  export AWS_REGION="${AWS_REGION:-us-west-2}"
  echo $AWS_REGION
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asegura que tu kubeconfig apunta al clúster correcto.

  ```bash
  export CLUSTER_NAME="${CLUSTER_NAME:-TU-CLUSTER}"
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida correctamente el acceso al motor de kuberentes.

  ```bash
  kubectl get nodes -o wide
  ```

- {% include step_label.html %} Valida que las herramientas esten instaladas localmente.

  ```bash
  kubectl version --client
  ```
  ```bash
  eksctl version
  ```
  ```bash
  git --version
  ```
  ```bash
  docker --version
  ```
  ```bash
  aws --version
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables base para Amazon ECR y valida formato del Account ID.

  ```bash
  export ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  export ECR_REPO="gitops-web"
  export ECR_URI="$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO"
  echo "ECR_URI=$ECR_URI"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Accede a tu cuenta de **GitHub**, sino tienes cuenta crea una nueva y crea un repositorio **publico**.

  - Da clic [AQUÍ](https://github.com/login) para abrir GitHub.
  - La sugerencia del nombre del repositorio puede ser: `repogitops-##xx`
  - Sustituye los **#** por numeros y las **x** por letras aleatorias.
  - Continua al siguiente paso una vez creado tu repositorio.

  {% include step_image.html %}

- {% include step_label.html %} Prepara variables para el repo Git remoto (ajusta a tu usuario/repositorio real).

  > **IMPORTANTE:** Sustituye **TU_USUARIO/TU_REPO.git** por el que creaste en el paso anterior.
  {: .lab-note .important .compact}

  > **IMPORTANTE:** GitHub no acepta contraseña por HTTPS. Usa PAT o SSH.
  {: .lab-note .important .compact}

  ```bash
  export GIT_REMOTE_URL="https://github.com/TU_USUARIO/TU_REPO.git"
  echo "GIT_REMOTE_URL=$GIT_REMOTE_URL"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida que el clúster tiene subnets con IPs disponibles.

  ```bash
  SUBNETS="$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query 'cluster.resourcesVpcConfig.subnetIds' --output text)"
  ```
  ```bash
  aws ec2 describe-subnets --subnet-ids $SUBNETS --region "$AWS_REGION"     --query 'Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone,Available:AvailableIpAddressCount}' --output table
  ```
  {% include step_image.html %}

{% capture r1 %}
**Resultado esperado:** Acceso a EKS confirmado (kubectl apunta al clúster correcto), herramientas validadas, y variables base de AWS/ECR/Git listas para ejecutar el pipeline GitOps.
{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Compilación de la app, push a ECR y manifiestos GitOps con Kustomize

Construirás una app mínima (contenedor), la publicarás en ECR y crearás una estructura GitOps con Kustomize: `base` (común) + `overlays/dev` (ambiente dev). El overlay controlará **tag de imagen** y **réplicas**.

#### Tarea 4.1

- {% include step_label.html %} Crea una app mínima para la prueba, primero crea el **Dockerfile**.

  > **NOTA:** El comando se ejecuta desde la raíz del direcotorio **lab02**
  {: .lab-note .info .compact}

  - `app/Dockerfile`:

  ```bash
  cat > app/Dockerfile <<'EOF'
  FROM nginx:alpine
  RUN apk add --no-cache gettext
  COPY index.html.template /usr/share/nginx/html/index.html.template
  COPY entrypoint.sh /entrypoint.sh
  RUN chmod +x /entrypoint.sh
  ENTRYPOINT ["/entrypoint.sh"]
  CMD ["nginx", "-g", "daemon off;"]
  EOF
  ```

- {% include step_label.html %} Ahora crea el archivo **entrypoint.sh**.

  > **NOTA:** El comando se ejecuta desde la raíz del direcotorio **lab02**
  {: .lab-note .info .compact}

  - `app/entrypoint.sh`:

  ```bash
  cat > app/entrypoint.sh <<'EOF'
  #!/bin/sh
  set -eu
  envsubst < /usr/share/nginx/html/index.html.template > /usr/share/nginx/html/index.html
  exec "$@"
  EOF
  ```

- {% include step_label.html %} Continua con la creación del archivo **index.html.template**. 

  > **NOTA:** El comando se ejecuta desde la raíz del direcotorio **lab02**
  {: .lab-note .info .compact}

  - `app/index.html.template`:

  ```bash
  cat > app/index.html.template <<'EOF'
  <h1>GitOps on EKS - Argo CD</h1>
  <p>Version: v1</p>
  <p>MESSAGE: ${MESSAGE}</p>
  EOF
  ```

- {% include step_label.html %} Verifica que los archivos esten creados correctamente.

  > **NOTA:** El comando se ejecuta desde la raíz del direcotorio **lab02**
  {: .lab-note .info .compact}

  ```bash
  ls -la app
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el repositorio no este creado.

  ```bash
  aws ecr describe-repositories --repository-names "$ECR_REPO" --region "$AWS_REGION" >/dev/null 2>&1
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el repo en ECR.

  ```bash
  aws ecr create-repository --repository-name "$ECR_REPO" --region "$AWS_REGION" --output table
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora autenticate al repositorio.
  
  ```bash
  aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define la version de la imagen.

  ```bash
  export IMAGE_TAG="v1"
  export IMAGE="$ECR_URI:$IMAGE_TAG"
  
  echo $IMAGE
  ```
  {% include step_image.html %}

- {% include step_label.html %} Compila el archivo Dockerfile y sube la imagen al repositorio.

  ```bash
  docker build -t "$IMAGE" app
  ```
  ```bash
  docker push "$IMAGE"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida la existencia de la imagen etiquetada en el servicio ECR.

  > **NOTA:** El resultado debe de ser el valor del **digest** de la imagen cargada.
  {: .lab-note .info .compact}

  ```bash
  aws ecr describe-images \
    --repository-name "$ECR_REPO" \
    --region "$AWS_REGION" \
    --query "imageDetails[?imageTags!=null && contains(imageTags, 'v1')].imageDigest" \
    --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora crea los manifiestos de GitOps en `gitops/base`.

  > **NOTA:** El comando se ejecuta desde la raíz del direcotorio **lab02**
  {: .lab-note .info .compact}

  ```bash
  cat > gitops/base/kustomization.yaml <<'EOF'
  resources:
    - namespace.yaml
    - configmap.yaml
    - deployment.yaml
    - service.yaml
  EOF

  cat > gitops/base/namespace.yaml <<'EOF'
  apiVersion: v1
  kind: Namespace
  metadata:
    name: gitops-demo
  EOF

  cat > gitops/base/configmap.yaml <<'EOF'
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: web-config
    namespace: gitops-demo
  data:
    MESSAGE: "Hola desde GitOps (BASE)"
  EOF

  cat > gitops/base/deployment.yaml <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: web
    namespace: gitops-demo
    labels:
      app: web
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: web
    template:
      metadata:
        labels:
          app: web
      spec:
        containers:
        - name: web
          image: REPLACE_IMAGE:latest
          ports:
          - containerPort: 80
          env:
          - name: MESSAGE
            valueFrom:
              configMapKeyRef:
                name: web-config
                key: MESSAGE
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
  EOF

  cat > gitops/base/service.yaml <<'EOF'
  apiVersion: v1
  kind: Service
  metadata:
    name: web
    namespace: gitops-demo
  spec:
    selector:
      app: web
    ports:
    - port: 80
      targetPort: 80
  EOF
  ```

- {% include step_label.html %} Valida con **--dry-run** los archivos YAML.

  ```bash
  kubectl apply --dry-run=client -f gitops/base/namespace.yaml
  ```
  ```bash
  kubectl apply --dry-run=client -f gitops/base/deployment.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el overlay `dev` para controlar imagen (ECR + tag) y réplicas (2 en dev).

  > **NOTA:** El comando se ejecuta desde la raíz del direcotorio **lab02**
  {: .lab-note .info .compact}

  ```bash
  cat > gitops/overlay/dev/kustomization.yaml <<'EOF'
  resources:
    - ../../base

  images:
    - name: REPLACE_IMAGE
      newName: REPLACE_ECR_URI
      newTag: v1

  patches:
    - target:
        kind: Deployment
        name: web
        namespace: gitops-demo
      patch: |-
        - op: replace
          path: /spec/replicas
          value: 2
  EOF
  ```
  ```bash
  sed -i "s#REPLACE_ECR_URI#$ECR_URI#g" gitops/overlay/dev/kustomization.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida con **--dry-run** los archivos YAML y que kustomize este correcto.

  ```bash
  kubectl kustomize gitops/overlay/dev | head -n 140
  ```
  ```bash
  kubectl apply --dry-run=client -k gitops/overlay/dev
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Instalación de Argo CD en EKS

Instalarás Argo CD en el namespace `argocd` usando manifiestos oficiales “stable” y validarás que sus componentes queden listos.

#### Tarea 5.1

- {% include step_label.html %} Crea el namespace `argocd`.

  ```bash
  kubectl create namespace argocd
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el name space se haya creado correctamente.

  ```bash
  kubectl get ns argocd
  ```
  {% include step_image.html %}

- {% include step_label.html %} Instala Argo CD desde el manifiesto oficial stable.

  > **NOTA:** La instalación no debe generar errores.
  {: .lab-note .info .compact}

  ```bash
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  ```

- {% include step_label.html %} Espera a que los componentes principales estén Ready.

  ```bash
  kubectl -n argocd get pods
  ```
  ```bash
  kubectl -n argocd rollout status deploy/argocd-server
  kubectl -n argocd rollout status deploy/argocd-repo-server
  kubectl -n argocd rollout status sts/argocd-application-controller
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa los servicios y eventos.

  ```bash
  kubectl -n argocd get svc
  ```
  ```bash
  kubectl -n argocd get deploy
  ```
  ```bash
  kubectl -n argocd get events --sort-by=.lastTimestamp | tail -n 15
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Acceso a Argo CD y creación declarativa de la Application

Accederás a Argo CD por port-forward, obtendrás el password inicial, subirás tu repo Git y crearás una `Application` declarativa que apunte al overlay dev.

#### Tarea 6.1

- {% include step_label.html %} Accede a la UI/API de ArgoCD por port-forward. **Abre una nueva Terminal en GitBash y ejecuta el siguiente comando**

  ```bash
  kubectl -n argocd port-forward svc/argocd-server 8080:443
  ```
  {% include step_image.html %}

- {% include step_label.html %} Abre el navegador **(Chrome/Edge)** en el equipo de trabajo y escribe la siguiente URL

  > **IMPORTANTE:** Acepta la advertencia de que la conexión no es privada.
  {: .lab-note .important .compact}

  ```bash
  https://localhost:8080
  ```
  {% include step_image.html %}

- {% include step_label.html %} **(Opcional)** Tambien puedes validar por medio de la terminal, pero es necesario que abras otra **tercera nueva terminal en el Bash**.

  ```bash
  curl -k -sS https://localhost:8080/ | head
  ```
  {% include step_image.html %}

- {% include step_label.html %} **En una nueva terminal** obtén el password inicial del usuario `admin`.

  > **NOTA:** Puede ser en la **Tercera terminal** que abriste en el paso anterior. Sino abre una nueva.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda el **password** en un bloc de notas. Usalo para entrar a la aplicación de ArgoCD con el usuario **`admin`**

   {% include step_image.html %}

- {% include step_label.html %} Prepara tu repo Git remoto (GitHub) y configura autenticación (PAT o SSH).

  > **IMPORTANTE:** **Regresa a la primera terminal de tu area de trabajo**. Si usas HTTPS, Git te pedirá usuario + PAT (no contraseña).
  {: .lab-note .important .compact}

- {% include step_label.html %} Inicializa Git local y sube `app/` y `gitops/` al repo remoto.

  > **IMPORTANTE:** Antes de ejecutar los comandos ajusta tu **tu_nombre** y **tu_correo**
  {: .lab-note .important .compact}

  ```bash
  git config --global user.name "tu_nombre"
  git config --global user.email "tu_correo"
  git config --global core.autocrlf input
  ```
  ```bash
  git init
  ```
  ```bash
  git add app gitops
  ```
  ```bash
  git commit -m "gitops: base + overlay dev (v1)"
  ```
  ```bash
  git branch -M main
  ```
  ```bash
  git remote remove origin 2>/dev/null || true
  git remote add origin https://TU_REPOSITORIO_AQUÍ.git
  ```
  ```bash
  git pull origin main --allow-unrelated-histories --no-edit
  ```
  - Si aparece el editor Vim presiona la tecla **Esc** y luego las teclas **:wq** y **Enter**
  ```bash
  git push -u origin main
  ```

  > **IMPORTANTE:** Si al finalizar el Push te pide autenticación sigue los pasos para autenticar el repositorio.
  {: .lab-note .important .compact}

  {% include step_image.html %}
  {% include step_image.html %}

- {% include step_label.html %} Crea la `Application` declarativa apuntando al overlay `gitops/overlays/dev`.

  ```bash
  cat > argocd/application-gitops-web.yaml <<'EOF'
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: gitops-web-dev
    namespace: argocd
    finalizers:
      - argocd.argoproj.io/resources-finalizer
  spec:
    project: default
    source:
      repoURL: REPLACE_REPO_URL
      targetRevision: HEAD
      path: gitops/overlay/dev
    destination:
      server: https://kubernetes.default.svc
      namespace: gitops-demo
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
      syncOptions:
        - CreateNamespace=true
  EOF
  ```
  ```bash
  sed -i "s#REPLACE_REPO_URL#$GIT_REMOTE_URL#g" argocd/application-gitops-web.yaml
  kubectl apply -f argocd/application-gitops-web.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora valida la aplicación con los siguientes comandos.

  ```bash
  kubectl -n argocd get applications
  ```
  ```bash
  kubectl -n argocd describe application gitops-web-dev | sed -n '1,260p'
  ```
  ```bash
  kubectl get ns gitops-demo
  ```
  {% include step_image.html %}
  {% include step_image.html %}
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Auto-sync, verificación del despliegue y prueba de self-heal

Confirmarás el despliegue desde Git, simularás drift (cambio manual) y observarás la autocorrección. Luego publicarás v2 (ECR + commit) y validarás el rollout.

#### Tarea 7.1

- {% include step_label.html %} Verifica los recursos desplegados en `gitops-demo`.

  ```bash
  kubectl -n gitops-demo get deploy,svc,cm
  ```
  ```bash
  kubectl -n gitops-demo rollout status deploy/web
  ```
  {% include step_image.html %}

- {% include step_label.html %} En otra terminal prueba la app por port-forward. Necesitas abrir otra terminal para realizar la prueba.

  > **NOTA:** Puede que ya la tengas abierta sino abre una nueva terminal.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n gitops-demo port-forward svc/web 8081:80
  ```
  {% include step_image.html %}

- {% include step_label.html %} Necesitas abrir otra terminal para realizar la prueba.

  > **NOTA:** Puedes reutilizar alguna terminal que ya no estes ocupando.
  {: .lab-note .info .compact}

  ```bash
  curl -sS http://localhost:8081 | head -n 20
  ```
  {% include step_image.html %}

- {% include step_label.html %} Simula el drift y observa que Argo CD lo revierte.

  ```bash
  kubectl -n gitops-demo scale deploy/web --replicas=1
  ```
  ```bash
  kubectl -n gitops-demo get deploy/web
  ```
  {% include step_image.html %}

- {% include step_label.html %} Debes validar que vuelve a 2.

  ```bash
  kubectl -n gitops-demo get deploy/web -w
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si notas que tarda, fuerza la actualización con el siguiente comando:

  ```bash
  kubectl -n argocd annotate application gitops-web-dev argocd.argoproj.io/refresh=hard --overwrite
  ```
  {% include step_image.html %}

- {% include step_label.html %} Compila y publica la v2. **Desde la primera terminal donde asegures tener las variables declaradas**

  > **IMPORTANTE:** El comando se ejecuta desde el directorio **lab02/app**
  {: .lab-note .important .compact}

  ```bash
  sed -i 's/Version: v1/Version: v2/' app/index.html.template
  ```
  ```bash
  export IMAGE_TAG="v2"
  export IMAGE="$ECR_URI:$IMAGE_TAG"
  docker build -t "$IMAGE" app
  docker push "$IMAGE"
  ```
  ```bash
  sed -i 's/newTag: v1/newTag: v2/' gitops/overlay/dev/kustomization.yaml
  ```
  {% include step_image.html %}
  {% include step_image.html %}

- {% include step_label.html %} Ahora promueve el tag por Git y sube el cambio.

  ```bash
  git add app/index.html.template gitops/overlay/dev/kustomization.yaml
  git commit -m "gitops: bump image to v2"
  git push
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida el rollout y la respuesta de la v2.

  > **IMPORTANTE:** El cambio puede tardar en reflejarse, puedes desactivar el portforward y volverlo a activar, si es necesario.
  {: .lab-note .important .compact}

  ```bash
  kubectl -n gitops-demo rollout status deploy/web
  ```
  ```bash
  curl -sS http://localhost:8081 | head -n 20
  ```
  {% include step_image.html %}
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Endurecimiento mínimo

Cerrarás el laboratorio con seguridad mínima (password inicial).

#### Tarea 8.1

- {% include step_label.html %} (Recomendado) Borra el secret del password inicial una vez cambies la contraseña.

  ```bash
  kubectl -n argocd delete secret argocd-initial-admin-secret
  ```
  {% include step_image.html %}

- {% include step_label.html %} Limpia la aplicación GitOps.
  ```bash
  kubectl delete ns gitops-demo
  ```
  ```bash
  kubectl -n argocd patch application gitops-web-dev --type=merge -p '{"metadata":{"finalizers":[]}}'
  kubectl -n argocd delete application gitops-web-dev --grace-period=0 --force
  ```

- {% include step_label.html %} Desinstala Argo CD.

  ```bash
  kubectl delete ns argocd
  ```

- {% include step_label.html %} Verifica que el ambiente este limpio.

  ```bash
  kubectl get ns | egrep "argocd|gitops-demo" || echo "Limpio (namespaces removidos)"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 9. Limpieza (opcional)

Removeras toda la configuración realizada por este laboratorio.

#### Tarea 9.1

- {% include step_label.html %} Elimina el repositorio ECR.

  ```bash
  aws ecr delete-repository --repository-name "$ECR_REPO" --region "$AWS_REGION" --force
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el repositorio se haya eliminado correctamente.

  ```bash
  aws ecr describe-repositories --repository-names "$ECR_REPO" --region "$AWS_REGION" >/dev/null 2>&1 || echo "ECR repo eliminado"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el clúster creado por `eksctl`.

  > **NOTA:** El cluster tardara **7 minutos** aproximadamente en eliminarse.
  {: .lab-note .info .compact}

  ```bash
  eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el cluster se haya eliminado correctamente.

  > **NOTA:** Costos principales:
  - EKS cobra por cluster/hora (standard vs extended support).
  - Los nodos se cobran como EC2/EBS/red.
  - ECR por almacenamiento/transferencia.
  {: .lab-note .info .compact}

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" >/dev/null 2>&1 || echo "Cluster eliminado"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[8] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}