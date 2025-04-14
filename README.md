# JDW Apps Helm Repository

[![Deploy Pages](https://github.com/jdwillmsen/jdw-apps/actions/workflows/release.yaml/badge.svg)](https://github.com/jdwillmsen/jdw-apps/actions/workflows/release.yaml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Docs](https://img.shields.io/badge/docs-github--pages-blue)](https://jdwillmsen.github.io/jdw-apps/)

<details open>
<summary>📦 Helm Chart Versions</summary>

[![authui Chart Version](https://img.shields.io/badge/dynamic/yaml?url=https%3A%2F%2Fraw.githubusercontent.com%2Fjdwillmsen%2Fjdw-apps%2Frefs%2Fheads%2Fmain%2Fcharts%2Fauthui%2FChart.yaml&query=%24.appVersion&prefix=v&label=authui)](https://github.com/jdwillmsen/jdw-apps/blob/main/charts/authui/Chart.yaml)
[![container Chart Version](https://img.shields.io/badge/dynamic/yaml?url=https%3A%2F%2Fraw.githubusercontent.com%2Fjdwillmsen%2Fjdw-apps%2Frefs%2Fheads%2Fmain%2Fcharts%2Fcontainer%2FChart.yaml&query=%24.appVersion&prefix=v&label=container)](https://github.com/jdwillmsen/jdw-apps/blob/main/charts/container/Chart.yaml)
[![rolesui Chart Version](https://img.shields.io/badge/dynamic/yaml?url=https%3A%2F%2Fraw.githubusercontent.com%2Fjdwillmsen%2Fjdw-apps%2Frefs%2Fheads%2Fmain%2Fcharts%2Frolesui%2FChart.yaml&query=%24.appVersion&prefix=v&label=rolesui)](https://github.com/jdwillmsen/jdw-apps/blob/main/charts/rolesui/Chart.yaml)
[![usersui Chart Version](https://img.shields.io/badge/dynamic/yaml?url=https%3A%2F%2Fraw.githubusercontent.com%2Fjdwillmsen%2Fjdw-apps%2Frefs%2Fheads%2Fmain%2Fcharts%2Fusersui%2FChart.yaml&query=%24.appVersion&prefix=v&label=usersui)](https://github.com/jdwillmsen/jdw-apps/blob/main/charts/usersui/Chart.yaml)
[![servicediscovery Chart Version](https://img.shields.io/badge/dynamic/yaml?url=https%3A%2F%2Fraw.githubusercontent.com%2Fjdwillmsen%2Fjdw-apps%2Frefs%2Fheads%2Fmain%2Fcharts%2Fservicediscovery%2FChart.yaml&query=%24.appVersion&prefix=v&label=servicediscovery)](https://github.com/jdwillmsen/jdw-apps/blob/main/charts/servicediscovery/Chart.yaml)
[![usersrole Chart Version](https://img.shields.io/badge/dynamic/yaml?url=https%3A%2F%2Fraw.githubusercontent.com%2Fjdwillmsen%2Fjdw-apps%2Frefs%2Fheads%2Fmain%2Fcharts%2Fusersrole%2FChart.yaml&query=%24.appVersion&prefix=v&label=usersrole)](https://github.com/jdwillmsen/jdw-apps/blob/main/charts/usersrole/Chart.yaml)
[![secret-store Chart Version](https://img.shields.io/badge/dynamic/yaml?url=https%3A%2F%2Fraw.githubusercontent.com%2Fjdwillmsen%2Fjdw-apps%2Frefs%2Fheads%2Fmain%2Fcharts%2Fsecret-store%2FChart.yaml&query=%24.appVersion&prefix=v&label=secret-store)](https://github.com/jdwillmsen/jdw-apps/blob/main/charts/secret-store/Chart.yaml)

</details>

---

## 🧭 Overview

JDW Apps is a Helm repository used for deploying applications via Argo CD. It provides a centralized way to manage
Kubernetes applications with environment-specific configurations and GitOps automation.

---

## 📁 Repository Structure

```text
.
├── envs/                 📦 Environment-specific configurations (dev/prd/uat)
├── excluded/             🗃️ Excluded/test charts and manifests
├── charts/               🛠️ Helm charts for JDW apps
├── jdw-apps.yaml         ⚙️ Main app configuration
├── LICENSE               📄 License info
└── README.md             📝 This file
```

---

## 🚀 Automated Deployment via Argo CD

This repo is wired to Argo CD using `ApplicationSet`, which dynamically configures and deploys Helm apps defined in
`envs/`.

### 🧾 Example `dev.yaml`

```yaml
apps:
  - appName: secret-store-dev
    helmPath: charts/secret-store
    namespace: dev
    values:
      - values-dev.yaml
  - appName: container-dev
    helmPath: charts/container
    namespace: dev
    values:
      - values-dev.yaml
  # ...and so on for other apps
```

### 🔧 ApplicationSet Bootstrap Configuration

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: jdw-apps-config
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: [ "missingkey=error" ]
  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/jdwillmsen/jdw-apps.git
              revision: main
              files:
                - path: envs/*
          - list:
              elementsYaml: "{{ .apps | toJson }}"
  template:
    metadata:
      name: "{{ .appName }}"
      annotations:
        argocd.argoproj.io/sync-wave: "1"
    spec:
      project: default
      source:
        repoURL: https://github.com/jdwillmsen/jdw-apps.git
        targetRevision: main
        path: "{{ .helmPath }}"
      destination:
        namespace: "{{ .namespace }}"
        server: https://kubernetes.default.svc
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - PruneLast=true
  templatePatch: |
    spec:
      source:
        helm:
          valueFiles:
            {{- toYaml .values | nindent 12 }}
```

---

## 🏗️ Adding a New Environment

1. Create a new YAML file under `envs/`, e.g. `envs/staging.yaml`
2. Define applications as shown:
   ```yaml
   apps:
     - appName: usersui-staging
       helmPath: charts/usersui
       namespace: staging
       values:
         - values-staging.yaml
   ```
3. Commit and push — Argo CD will detect and deploy automatically

---

## 📦 Adding a New Helm Chart

1. Create your chart:

   ```bash
   helm create charts/<app-name>
   ```

2. Customize `Chart.yaml`, `values.yaml`, and templates.
3. Add the chart to the appropriate environment file (e.g., `envs/dev.yaml`).
4. (Optional) Add `values-dev.yaml`, `values-prd.yaml`, etc. for overrides.
5. Test your chart locally:

   ```bash
   helm install --dry-run --debug ./charts/<app-name>
   ```

---

## 🧱 JDW Platform Repos

### ☸️ JDW Kube

The [**JDW Kube**](https://github.com/jdwillmsen/jdw-kube) repo manages the Kubernetes infrastructure for the JDW
platform.

- 🔒 Cluster Configuration & Secrets Management
- 🔁 Automated Deployments via Argo CD
- ⚙️ Base Ingress, Monitoring, and Platform Services

### 🧩 JDW Project

The [**JDW Monorepo**](https://github.com/jdwillmsen/jdw) powers all front-end and back-end services.

- 🧱 Apps (Angular, Go, Spring Boot)
- 🧬 Libs (UI, feature, util, data-access)
- ⚙️ Tools (CI/CD, Docker, versioning, linting)

---

## 🌍 Documentation

All Helm charts and deployment details are available via GitHub Pages:

🔗 [**JDW Apps GitHub Pages**](https://jdwillmsen.github.io/jdw-apps/)

This site is auto-updated by 🚀 [GitHub Actions](https://github.com/jdwillmsen/jdw-apps/actions/workflows/release.yaml).

---

## 📄 License

This repository is licensed under the [MIT License](https://opensource.org/licenses/MIT). See the `LICENSE` file for
full details.