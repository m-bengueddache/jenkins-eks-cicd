# CI/CD Pipeline — Jenkins, EKS & ECR

> **FR** — Pipeline CI/CD complet avec Jenkins déployant une application Java Spring Boot sur Amazon EKS. Le projet couvre trois phases : accès kubectl depuis Jenkins, déploiement via Dockerhub, puis migration vers AWS ECR. Un bonus couvre l'équivalent sur Linode LKE.
>
> **EN** — Full CI/CD pipeline with Jenkins deploying a Java Spring Boot app to Amazon EKS. The project covers three phases: kubectl access from Jenkins, deployment via Dockerhub, then migration to AWS ECR. A bonus section covers the equivalent setup on Linode LKE.

---

## Stack

![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-red?logo=jenkins)
![AWS](https://img.shields.io/badge/AWS-EKS%20%7C%20ECR-orange?logo=amazonaws)
![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS%20%7C%20LKE-326CE5?logo=kubernetes)
![Docker](https://img.shields.io/badge/Docker-Hub%20%7C%20ECR-2496ED?logo=docker)
![Java](https://img.shields.io/badge/Java-17-blue?logo=openjdk)
![Maven](https://img.shields.io/badge/Maven-build-red?logo=apachemaven)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3-green?logo=springboot)

---

## FR — Description

### Part 1 — Accès kubectl depuis Jenkins

Configuration de l'environnement Jenkins pour communiquer avec un cluster EKS :

1. **Installation de kubectl** — installé manuellement dans le conteneur Jenkins (absent par défaut)
2. **AWS IAM Authenticator** — EKS délègue l'authentification à AWS IAM ; ce binaire génère un token court signé avec les credentials AWS à chaque appel `kubectl`
3. **kubeconfig** — fichier créé manuellement et copié dans le conteneur ; nommé `config-aws` pour ne pas écraser un contexte local existant
4. **Credentials Jenkins** — `AWS_ACCESS_KEY_ID` et `AWS_SECRET_ACCESS_KEY` injectés via l'environnement du stage `deploy` pour que l'authenticator puisse signer les requêtes

### Part 2 — Pipeline CI/CD complet avec EKS + Dockerhub

Extension du pipeline existant (incrémentation de version, build Maven, push image) avec un stage `deploy` vers EKS :

1. **Manifests Kubernetes** — `deployment.yaml` et `service.yaml` avec des placeholders `$APP_NAME` et `$IMAGE_NAME` substitués dynamiquement à chaque build
2. **envsubst** — outil du paquet `gettext-base`, injecte les variables d'environnement shell dans les YAML avant de les passer à `kubectl apply`
3. **Secret Dockerhub** — `imagePullSecret` créé une seule fois localement pour autoriser Kubernetes à puller l'image privée ; sans lui, le pull est anonyme et échoue
4. **kubectl apply** — remplace `kubectl create` pour des déploiements idempotents ; l'image est mise à jour à chaque run via le tag dynamique `<version>-<build_number>`

### Part 3 — Migration vers AWS ECR

Remplacement de Dockerhub par AWS ECR comme registre de conteneurs :

1. **Dépôt ECR** — un dépôt par application (contrairement à Dockerhub où un dépôt peut héberger plusieurs images)
2. **Credentials** — username `AWS`, token récupéré via `aws ecr get-login-password` ; créé comme credential global Jenkins
3. **Secret Kubernetes** — même principe que Dockerhub, `aws-registry-key` référencé dans `imagePullSecrets`
4. **Variables d'environnement** — `DOCKER_REPO_SERVER` et `DOCKER_REPO` définis au niveau global du pipeline pour éviter le hardcoding et centraliser les changements

### Bonus — Linode LKE

LKE utilise un kubeconfig standard avec token, sans couche IAM. Le plugin Jenkins **Kubernetes CLI** fournit le step `withKubeConfig` qui configure `kubectl` à la volée depuis un credential Jenkins de type *Secret file* — plus besoin de copier manuellement des fichiers dans le conteneur.

**EKS vs LKE**

| Aspect | EKS (AWS) | LKE (Linode) |
|---|---|---|
| Authentification | IAM via `aws-iam-authenticator` | Token kubeconfig standard |
| Credential Jenkins | 2 × Secret text (key ID + secret) | 1 × Secret file (kubeconfig) |
| Plugin Jenkins | Aucun (setup manuel) | Kubernetes CLI |
| `KUBECONFIG` | Variable d'env requise | Géré par `withKubeConfig` |

---

## EN — Description

### Part 1 — kubectl access from Jenkins

Setting up the Jenkins environment to communicate with an EKS cluster:

1. **kubectl installation** — installed manually inside the Jenkins container (not available by default)
2. **AWS IAM Authenticator** — EKS delegates authentication to AWS IAM; this binary generates a short-lived token signed with AWS credentials on every `kubectl` call
3. **kubeconfig** — created manually and copied into the container; named `config-aws` to avoid overwriting an existing local context
4. **Jenkins credentials** — `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` injected via the `deploy` stage environment so the authenticator can sign requests

### Part 2 — Full CI/CD pipeline with EKS + Dockerhub

Extending the existing pipeline (version increment, Maven build, image push) with a `deploy` stage targeting EKS:

1. **Kubernetes manifests** — `deployment.yaml` and `service.yaml` with `$APP_NAME` and `$IMAGE_NAME` placeholders substituted dynamically at each build
2. **envsubst** — part of the `gettext-base` package; injects shell environment variables into YAML files before piping to `kubectl apply`
3. **Dockerhub secret** — `imagePullSecret` created once locally to allow Kubernetes to pull the private image; without it, the pull is anonymous and fails
4. **kubectl apply** — replaces `kubectl create` for idempotent deployments; the image is updated on every run via the dynamic `<version>-<build_number>` tag

### Part 3 — Migrate to AWS ECR

Replacing Dockerhub with AWS ECR as the container registry:

1. **ECR repository** — one repository per application (unlike Dockerhub where one repository can host multiple images)
2. **Credentials** — username `AWS`, token retrieved via `aws ecr get-login-password`; created as a global Jenkins credential
3. **Kubernetes secret** — same pattern as Dockerhub, `aws-registry-key` referenced in `imagePullSecrets`
4. **Environment variables** — `DOCKER_REPO_SERVER` and `DOCKER_REPO` defined at pipeline level to avoid hardcoding and centralise changes

### Bonus — Linode LKE

LKE uses a standard kubeconfig token without an IAM layer. The Jenkins **Kubernetes CLI** plugin provides the `withKubeConfig` step, which configures `kubectl` on the fly from a Jenkins *Secret file* credential — no manual file copying into the container needed.

**EKS vs LKE**

| Aspect | EKS (AWS) | LKE (Linode) |
|---|---|---|
| Authentication | IAM via `aws-iam-authenticator` | Standard kubeconfig token |
| Jenkins credential | 2 × Secret text (key ID + secret) | 1 × Secret file (kubeconfig) |
| Jenkins plugin | None (manual setup) | Kubernetes CLI |
| `KUBECONFIG` | Required env var | Handled by `withKubeConfig` |

---

## Architecture

```
Git Push
   │
   ▼
Jenkins Pipeline
   ├── 1. Increment version ──── pom.xml patch++
   ├── 2. Build app ──────────── mvn clean package → app.jar
   ├── 3. Build & push image ─── docker build + push → ECR / Dockerhub
   ├── 4. Deploy to EKS ──────── envsubst → kubectl apply
   │                                  └── Deployment (2 replicas) + Service
   └── 5. Commit version bump ── git push → main
```

---

## Jenkins Prerequisites

| Credential ID | Type | Usage |
|---|---|---|
| `ecr-credentials` | Username/Password | Docker login to AWS ECR (`AWS` / token) |
| `jenkins_aws_access_key_id` | Secret text | AWS IAM auth for EKS |
| `jenkins_aws_secret_access_key` | Secret text | AWS IAM auth for EKS |
| `git-credentials` | Username/Password | Push version bump commit to GitHub |

---

## Project Structure

```
.
├── Jenkinsfile                 # Full CI/CD pipeline (ECR version)
├── Dockerfile                  # Java app image (Amazon Corretto 17)
├── pom.xml                     # Maven project (Spring Boot 3)
├── kubernetes/
│   ├── deployment.yaml         # Deployment with envsubst placeholders
│   └── service.yaml            # Service (port 80 → 8080)
└── src/                        # Java Spring Boot application source
```
