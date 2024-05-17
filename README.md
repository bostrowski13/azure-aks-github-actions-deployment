- [azure-aks-github-actions-deployment](#azure-aks-github-actions-deployment)
  - [Scope Of Repository](#scope-of-repository)
  - [Pipeline Information](#pipeline-information)
  - [Prerequisites](#prerequisites)
  - [Features](#features)
    - [Common Steps](#common-steps)
    - [Service Specific Steps](#service-specific-steps)
  - [Pipeline Steps](#pipeline-steps)
    - [Retrieving Secrets](#retrieving-secrets)
      - [Writing Secrets to Files](#writing-secrets-to-files)
      - [Writing Secrets to GitHub Action environment variables](#writing-secrets-to-github-action-environment-variables)
    - [Application Deployment Repeatable Step](#application-deployment-repeatable-step)
    - [Config Lookup](#config-lookup)
    - [Linting](#linting)
    - [Bake and Deploy](#bake-and-deploy)
  - [How To Setup the Pipeline](#how-to-setup-the-pipeline)
  - [Setting the Kubernetes Service Type](#setting-the-kubernetes-service-type)
    - [Service.yaml](#serviceyaml)
      - [Example](#example)
    - [Values.yaml](#valuesyaml)
      - [Example Values.yaml Snippet](#example-valuesyaml-snippet)
    - [Annotations for IP and LB Association](#annotations-for-ip-and-lb-association)
      - [Internal LB behind firewall](#internal-lb-behind-firewall)
        - [Example](#example-1)
      - [Public LB without a firewall](#public-lb-without-a-firewall)
        - [Example](#example-2)
  - [Reference Documentation](#reference-documentation)

# azure-aks-github-actions-deployment

## Scope Of Repository

This repository holds GitHub Workflows and Actions to deploy Applications using Helm to an AKS Cluster in Azure cloud.

## Pipeline Information

This `help-deploy` workflow is an extensible pipeline mean to simplify the common tasks for a deployment, and make it repeatable for specific one to many application  deployments.  The common steps for the workflow are located in the `helm-deploy.yml` file, and the repeatable, extensible steps for LInting, Fetching Secrets, Baking, and Deploying the application are hosted in the `.github/actions/helm-deploy/actions.yml` file.

## Prerequisites

* The infrastructure pipeline for the target environment must have successfully deployed first before this pipeline can be run to deploy any applications.
* Github Secrets for the Service Pricipal (Azure CLient ID, Tenent ID, and Subscription ID) for the target subscription have been added to the Github environment(s).
* Secrets loaded into Key Vault in the target environment for each application that are **not** created/loaded in the Infrastructure pipeline.

## Features

### Common Steps

This pipeline provides the same level of interface as the Infrastructure deployment pipeline (e.g. Branch, Environment, and Profile, etc) options.  In addition, it includes these common steps.

* Helm Install
* Azure Login using the Service Principal
* ACR Login
* Setup Kubectl
* Dynamically Fetching the Resource Group, AKS Cluster Name, and Key Vault for the environment selected
* Setting the AKS Context
* Logging out of AWS

### Service Specific Steps

In addition, for each application a yml block can be declared that contains the specific configuration parameters for the application to deploy to the AKS cluster.  These include:

* Linting/Validating Manifest Files
* Reading Base, Profile, and Environment Specific Override configs
* Fetching application specific secrets from Key Vault and writing them to the secrets tpl file
* Baking the manifest files
* Deploying the application

## Pipeline Steps

### Retrieving Secrets

You can reuse the following code snippet to pull secrets from Key Vault.  Note that this step must be run after retrieving the Resource Group and Key Vault name in the `Get AKS Information for Deploy and Set Environment Vars` step since it relies on information gathered from that step.

#### Writing Secrets to Files

```yaml
- name: Fetch Secrets from Key Vault and Assign to TPL file
  uses: azure/CLI@2
  with:
    inlineScript: |
      value=$(az keyvault secret show --vault-name ${{ env.AZ_KEYVAULT_NAME }} --name __THE_SECRET_NAME_HERE__ --query value -o tsv)

      # Write the output to a file
      echo "$value" > "../../helm/${{ inputs.application-key }}.secrets.tpl"

```

#### Writing Secrets to GitHub Action environment variables

```yaml
- name: Fetch Secrets from Key Vault and Assign to TPL file
  uses: azure/CLI@2
  with:
    inlineScript: |
      value=$(az group list --query "[?ends_with(name, '${{ inputs.environment }}-rg-aks')].name" -o tsv)

      # Write the output to a github environment variable
      echo "AZ_RESOURCE_GROUP=$(value)" >> $GITHUB_ENV

```

### Application Deployment Repeatable Step

The following is the code block that will need to be repeated with every application that is to be deployed to AKS.  It uses commonly retrieved values from the `heml-deploy.yaml` workflow file, as well as takes in application specific parameters to perform lookup and configuration that are specific to the service being deploy.  This uses the templated `action.yaml` in `actions/helm-deploy` to easily repeat the common build and deployment steps for each service.

```yaml
- name: Deploying __YOUR_APP_NAME_HERE__ Application via Helm
  uses: ./.github/actions/helm-deploy
  with:
    helm-namespace: ${{ inputs.helm-namespace }}       # retrieved from inputs, only change if different than default
    application-key: "__YOUR_APP_NAME_HERE__"          # e.g. myapp
    stamp: ${{ inputs.stamp }}                         # retrieved from inputs, no need to change
    environment: ${{ inputs.environment }}             # retrieved from inputs, no need to change
    cert-file-path: ${{ inputs.cert-file-path }}       # retrieved from inputs, only change if different than default
    ca-cert-file-path: ${{ inputs.ca-cert-file-path }} # retrieved from inputs, only change if different than default
```

### Config Lookup

The hierarchy used for configuration settings and overrides used here are stored in the `env` folder and mirrors the same override pattern (base > profile > environment) so that the last configuration passed to the deployment is used (most specific).

The following steps retrieve the configurations from this hierarchy and store them as yaml to be passed in to the `overrides` parameter, in the the order noted above, in the `Deploy Helm Chart for the Application` step.

```yaml
- name: Read Base Config File
  uses: pietrobolcato/action-read-yaml@1.1.0
  id: read_base_config_file
  with:
    config: ${{ github.workspace }}/env/base-config.yaml

- name: Read Profile Config File
  uses: pietrobolcato/action-read-yaml@1.1.0
  id: read_profile_config_file
  with:
    config: ${{ github.workspace }}/env/${{ inputs.stamp }}/profile-config.yaml

- name: Read Overrides Config File
  uses: pietrobolcato/action-read-yaml@1.1.0
  id: read_overrides_config_file
  with:
    config: ${{ github.workspace }}/env/${{ inputs.stamp }}/${{ inputs.environment }}/overrides-config.yaml
```

### Linting

This pipeline also uses a linter to validate the Helm Chart files before a deployment is attempted.  The previous pipeline used this and it makes sense from a time-saving standpoint to use it again here.

```yaml
- name: Lint manifest files
  uses: azure/k8s-lint@v1
  with:
    lintType: dryrun
    manifests: |
      "../../helm/${{ inputs.application-key }}"
```

### Bake and Deploy

The following 2 steps are used to bake and deploy the Helm Chart to the AKS Cluster specified in the `Set AKS Context` step.  The deployment step uses the produced output manifest files for the deployment step by referencing them via the `steps.bake.outputs.manifestsBundle` variable.  The `arguments` block can be used to pass any additional configuration options that aren't supported with native blocks of the GitHub Actions tep.

```yaml
- name: Bake manifest files to be used for deployments
  uses: azure/k8s-bake@v3
  with:
    renderEngine: 'helm'
    helmChart: "../../helm/${{ inputs.application-key }}"
    arguments: |
      --cert-file
      ${{ inputs.cert-file-path }}
      --ca-file
      ${{ inputs.ca-cert-file-path }}
    overrides: |
      ${{ steps.read_base_config_file.outputs['${{ inputs.application-key }}.chart_values'] }}
      ${{ steps.read_profile_config_file.outputs['${{ inputs.application-key }}.chart_values'] }}
      ${{ steps.read_overrides_config_file.outputs['${{ inputs.application-key }}.chart_values'] }}
    releaseName: ${{ steps.read_base_config_file.outputs['${{ inputs.application-key }}.chart_version'] }}
    silent: 'false'
  id: bake

- name: Deploy Helm Chart for the Application
  uses: Azure/k8s-deploy@v5
  with:
    namespace: ${{ inputs.helm-namespace }}
    manifests: ${{ steps.bake.outputs.manifestsBundle }}
    action: deploy
    force: ${{ inputs.force_deploy }}
    strategy: basic
    annotate-resources: ${{ inputs.annotate-namespace }}
    annotate-namespace: ${{ inputs.annotate-namespace-extra }}
```

## How To Setup the Pipeline

1. Copy the `.github/workflows/helm-deploy.yml` and `.github/actions/helm-deploy/action.yml` files to the target repository in the same location.
2. in `helm-deploy.yml` modify the `HELM_REGISTRY` to the correct registry, if needed.
3. Update the `environment`, `stamp`, and `helm-namespace` parameters as needed.
4. In the `helm-deploy.yml` file, update the `Installing Application via Helm` task to the correct name and update its parameters to your service specific ones.
5. If necessary, copy the above block and update it for each application you want to deploy, updating to use a unique name and configuration parameters as needed.

**NOTE:**  For additional applications, you will only need to repeat
step 4 above for each application.

## Setting the Kubernetes Service Type

New services should use the `LoadBalancer` Service type.  The following provides instructions on the updates needed to support this.

### Service.yaml

* Change the `spec.type` value to `LoadBalancer` and add the `spec.ports.targetPort` which the container will run on internally in the cluster (example 8080), and add `spec.ports.port` for the target port of the load balancer to expose the Service for the network traffic outside of the cluster (example 80).

* Add `spec.selector.app` and its underlying properties to map the service to the corresponding pods with matching labels. (Note this doesn't require any updates to Deployment.yaml as you already have this defined there).  See the example below.
**NOTE:** The Values.yaml and environment specific Value yml files will used for specific configuration.  Updates to Values.yaml are detailed below if you are using templating.

#### Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.app.name }}
  labels:
    app.kubernetes.io/name: {{ .Values.app.name }}
    chart: {{ .Chart.Name }}
    release: {{ .Release.Name }}
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: MyApp
      release: 1.2.3
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
```

### Values.yaml

* Change `service.type` to have a value of `LoadBalancer`
* Change `service.dataport` to `service.port` and set its value to `80`
* Add `service.targetPort` and set its value to `8080`

#### Example Values.yaml Snippet

```yaml
service:
  type: LoadBalancer
  port: 80
  targetPort: 8080
```

### Annotations for IP and LB Association

In addition to changing to the LoadBalancer Service type, Annotations will be used to associate the Service with the IP Address, pre-provisioned or as part of the Infra deployment, which is how the IPs will be associated with a particular service.

#### Internal LB behind firewall

The following will need to be included annotations for deployment where the public IPs reside on the Firewall (e.g. Production), and the  Load Balancer is internal.  This creates the Service load balancer in the node resource group connected to the same virtual network as your AKS cluster.

```yaml
service.beta.kubernetes.io/azure-load-balancer-internal: "true"
service.beta.kubernetes.io/azure-load-balancer-ipv4: <pre-determined IP address from internal subnet>
```
##### Example

```yaml
metadata:
  name: my-app
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

#### Public LB without a firewall

The following will need to be included annotations for deployment where the Public IPs are attached directly to the Load Balancer:

```yaml
service.beta.kubernetes.io/azure-pip-prefix-id: <yourprefixid>
** OR **
service.beta.kubernetes.io/azure-pip-name: <your pip name>
```

##### Example

```yaml
metadata:
  name: my-app
  annotations:
    service.beta.kubernetes.io/azure-pip-name: "pip-ip"
```

## Reference Documentation

* https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types
* https://cloud-provider-azure.sigs.k8s.io/topics/loadbalancer/#loadbalancer-annotations
* https://learn.microsoft.com/en-us/azure/aks/internal-lb?tabs=set-service-annotations
* https://kubernetes.io/docs/concepts/services-networking/service/
* https://learn.microsoft.com/en-us/azure/aks/static-ip
