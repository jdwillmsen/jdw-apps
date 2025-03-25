# JDW Apps Helm Repository

## Overview

JDW Apps is a Helm repository used for deploying applications via Argo CD. It provides a centralized way to manage
applications in Kubernetes clusters with environment-specific configurations. The setup ensures that deployments are
automated and self-healing through Argo CD’s `ApplicationSet`.

## Repository Structure

```text
.
├── envs                  # Environment-specific configurations
│   ├── dev.yaml          # Development environment values
│   ├── prd.yaml          # Production environment values
│   └── uat.yaml          # UAT environment values
├── excluded              # Excluded Helm charts and configurations
│   ├── *                 # Other excluded manifests
│   └── secrets-tester    # Example/test Helm chart for secrets management
├── charts                # Helm charts for various applications
│   ├── authui            # Authentication UI Helm chart
│   ├── container         # Containerized application Helm chart
│   ├── secret-store      # Secret storage management Helm chart
│   ├── servicediscovery  # Service discovery Helm chart
│   ├── usersrole         # User role management Helm chart
│   └── usersui           # User UI Helm chart
├── jdw-apps.yaml         # Main application configuration
├── LICENSE               # Repository license
└── README.md             # This documentation
```

## Automated Deployment with Argo CD

This repository is designed for seamless deployments using Argo CD. It leverages `ApplicationSet` to dynamically
configure applications based on environment definitions in `envs/`.

### Example `dev.yaml`

Each environment file defines the applications to be deployed, their Helm chart locations, and corresponding values
files:

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
  - appName: usersrole-dev
    helmPath: charts/usersrole
    namespace: dev
    values:
      - values-dev.yaml
  - appName: authui-dev
    helmPath: charts/authui
    namespace: dev
    values:
      - values-dev.yaml
  - appName: servicediscovery-dev
    helmPath: charts/servicediscovery
    namespace: dev
    values:
      - values-dev.yaml
  - appName: usersui-dev
    helmPath: charts/usersui
    namespace: dev
    values:
      - values-dev.yaml
```

### Bootstrap Configuration

A single `ApplicationSet` resource ensures all applications are deployed and managed automatically:

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

This configuration ensures:

- Automatic detection of applications based on `envs/` files.
- Dynamic Helm chart deployment.
- Self-healing and pruning of outdated resources.

## Adding a New Environment

This section should explain how to introduce a new environment, including:

* Creating a new environment YAML file under envs/
* Defining applications in the new environment file
* Committing and pushing changes to trigger Argo CD updates

Example content:
```yaml
# Example: envs/staging.yaml
apps:
  - appName: secret-store-staging
    helmPath: charts/secret-store
    namespace: staging
    values:
      - values-staging.yaml
  - appName: usersui-staging
    helmPath: charts/usersui
    namespace: staging
    values:
      - values-staging.yaml
```

## Adding a New Helm Chart

This section should cover:

* Creating a new Helm chart inside the helm/ directory
* Defining the necessary Chart.yaml, values.yaml, and templates/
* Defining any overrides per environment(s) values-<env>.yaml
* Adding the chart to the appropriate environment file (envs/*.yaml)
* Testing the chart with Helm before committing

Example:
```sh
helm create charts/<app-name>
```

## Helm Chart Usage

Each Helm chart can be deployed individually using the following commands:

```sh
charts repo add jdw-apps https://github.com/jdwillmsen/jdw-apps
charts install my-app jdw-apps/<chart-name> -f charts/<chart-name>/values-dev.yaml
```

## License

This repository is licensed under the MIT License. See the `LICENSE` file for more details.

