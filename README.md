# Helmify
[![CI](https://github.com/arttor/helmify/actions/workflows/ci.yml/badge.svg)](https://github.com/arttor/helmify/actions/workflows/ci.yml)
![GitHub go.mod Go version](https://img.shields.io/github/go-mod/go-version/arttor/helmify)
![GitHub](https://img.shields.io/github/license/arttor/helmify)
![GitHub release (latest by date)](https://img.shields.io/github/v/release/arttor/helmify)
[![Go Report Card](https://goreportcard.com/badge/github.com/arttor/helmify)](https://goreportcard.com/report/github.com/arttor/helmify)
[![GoDoc](https://godoc.org/github.com/arttor/helmify?status.svg)](https://pkg.go.dev/github.com/arttor/helmify?tab=doc)
![GitHub total downloads](https://img.shields.io/github/downloads/arttor/helmify/total)

CLI that creates [Helm](https://github.com/helm/helm) charts from kubernetes yamls.

Helmify reads a list of [supported k8s objects](#status) from stdin and converts it to a helm chart. 
Designed to generate charts for [k8s operators](#integrate-to-your-operator-sdkkubebuilder-project) but not limited to.
See [examples](https://github.com/arttor/helmify/tree/main/examples) of charts generated by helmify.

Supports `Helm >=v3.6.0`

Submit issue if some features missing for your use-case.

## Usage

1) As pipe:

    ```shell
    cat my-app.yaml | helmify mychart
    ```
   Will create 'mychart' directory with Helm chart from yaml file with k8s objects.

    ```shell
    awk 'FNR==1 && NR!=1  {print "---"}{print}' /<my_directory>/*.yaml | helmify mychart
    ```
   Will create 'mychart' directory with Helm chart from all yaml files in `<my_directory> `directory.

2) From filesystem:
    ```shell
    helmify -f /my_directory/my-app.yaml mychart
    ```
    Will create 'mychart' directory with Helm chart from `my_directory/my-app.yaml`.
    ```shell
    helmify -f /my_directory mychart
    ```
    Will create 'mychart' directory with Helm chart from all yaml files in `<my_directory> `directory.
    ```shell
    helmify -f /my_directory -r mychart
    ```
    Will create 'mychart' directory with Helm chart from all yaml files in `<my_directory> `directory recursively.
    ```shell
    helmify -f ./first_dir -f ./second_dir/my_deployment.yaml -f ./third_dir  mychart
    ```
    Will create 'mychart' directory with Helm chart from multiple directories and files.


3) From [kustomize](https://kustomize.io/) output:
    ```shell
    kustomize build <kustomize_dir> | helmify mychart
    ```
    Will create 'mychart' directory with Helm chart from kustomize output.

### Integrate to your Operator-SDK/Kubebuilder project

1. Open `Makefile` in your operator project generated by 
   [Operator-SDK](https://github.com/operator-framework/operator-sdk) or [Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder).
2. Add these lines to `Makefile`:
- With operator-sdk version < v1.23.0 
    ```makefile
    HELMIFY = $(shell pwd)/bin/helmify
    helmify:
    	$(call go-get-tool,$(HELMIFY),github.com/arttor/helmify/cmd/helmify@v0.3.7)
    
    helm: manifests kustomize helmify
    	$(KUSTOMIZE) build config/default | $(HELMIFY)
    ```
- With operator-sdk version >= v1.23.0
    ```makefile
    HELMIFY ?= $(LOCALBIN)/helmify
    
    .PHONY: helmify
    helmify: $(HELMIFY) ## Download helmify locally if necessary.
    $(HELMIFY): $(LOCALBIN)
    	test -s $(LOCALBIN)/helmify || GOBIN=$(LOCALBIN) go install github.com/arttor/helmify/cmd/helmify@latest
        
    helm: manifests kustomize helmify
    	$(KUSTOMIZE) build config/default | $(HELMIFY)
    ```
3. Run `make helm` in project root. It will generate helm chart with name 'chart' in 'chart' directory.

## Install

With [Homebrew](https://brew.sh/) (for MacOS or Linux): `brew install arttor/tap/helmify`

Or download suitable for your system binary from [the Releases page](https://github.com/arttor/helmify/releases/latest).
Unpack the helmify binary and add it to your PATH and you are good to go!

## Available options
Helmify takes a chart name for an argument.
Usage:

```helmify [flags] CHART_NAME```  -  `CHART_NAME` is optional. Default is 'chart'. Can be a directory, e.g. 'deploy/charts/mychart'.

| flag                      | description                                                                                                                                                                                                 | sample                              |
|---------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------|
| -h -help                  | Prints help                                                                                                                                                                                                 | `helmify -h`                        |
| -f                        | File source for k8s manifests (directory or file), multiple sources supported                                                                                                                               | `helmify -f ./test_data`            |
| -r                        | Scan file directory recursively. Used only if -f provided                                                                                                                                                   | `helmify -f ./test_data -r`         |
| -v                        | Enable verbose output. Prints WARN and INFO.                                                                                                                                                                | `helmify -v`                        |
| -vv                       | Enable very verbose output. Also prints DEBUG.                                                                                                                                                              | `helmify -vv`                       |
| -version                  | Print helmify version.                                                                                                                                                                                      | `helmify -version`                  |
| -crd-dir                  | Place crds in their own folder per Helm 3 [docs](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/#method-1-let-helm-do-it-for-you). Caveat: CRDs templating is not supported by Helm. | `helmify -crd-dir`                  |
| -image-pull-secrets       | Allows the user to use existing secrets as imagePullSecrets                                                                                                                                                 | `helmify -image-pull-secrets`       |
| -cert-manager-as-subchart | Allows the user to install cert-manager as a subchart                                                                                                                                                       | `helmify -cert-manager-as-subchart` |
| -cert-manager-version | Allows the user to specify cert-manager subchart version. Only useful with cert-manager-as-subchart. (default "v1.12.2")                                                                                                                                                       | `helmify -cert-manager-as-subchart` |
## Status
Supported k8s resources:
- Deployment, DaemonSet, StatefulSet
- Job, CronJob
- Service, Ingress
- PersistentVolumeClaim
- RBAC (ServiceAccount, (cluster-)role, (cluster-)roleBinding)
- configs (ConfigMap, Secret)
- webhooks (cert, issuer, ValidatingWebhookConfiguration)
- custom resource definitions (CRD)

### Known issues
- Helmify will not overwrite `Chart.yaml` file if presented. Done on purpose.
- Helmify will not delete existing template files, only overwrite.
- Helmify overwrites templates and values files on every run. 
  This means that all your manual changes in helm template files will be lost on the next run.
- if switching between the using the `-crd-dir` flag it is better to delete and regenerate the from scratch to ensure crds are not accidentally spliced/formatted into the same chart. Bear in mind you will want to update your `Chart.yaml` thereafter.
  
## Develop
To support a new type of k8s object template:
1. Implement `helmify.Processor` interface. Place implementation in `pkg/processor`. The package contains 
examples for most k8s objects.
2. Register your processor in the `pkg/app/app.go`
3. Add relevant input sample to `test_data/kustomize.output`.


### Run
Clone repo and execute command:

```shell
cat test_data/k8s-operator-kustomize.output | go run ./cmd/helmify mychart
```

Will generate `mychart` Helm chart form file `test_data/k8s-operator-kustomize.output` representing typical operator
[kustomize](https://github.com/kubernetes-sigs/kustomize) output.

### Test
For manual testing, run program with debug output:
```shell
cat test_data/k8s-operator-kustomize.output | go run ./cmd/helmify -vv mychart
```
Then inspect logs and generated chart in `./mychart` directory.

To execute tests, run:
```shell
go test ./...
```
Beside unit-tests, project contains e2e test `pkg/app/app_e2e_test.go`.
It's a go test, which uses `test_data/*` to generate a chart in temporary directory. 
Then runs `helm lint --strict` to check if generated chart is valid.
