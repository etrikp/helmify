# Helmify
Helmify reads kubernetes resources from std.in and produces a [Helm](https://github.com/helm/helm) chart.

Main [use-case](#integrate-to-your-operator-sdk-project) is to create Helm charts for kubernetes operators build with
[Operator-SDK](https://github.com/operator-framework/operator-sdk).

## Run
Clone repo and execute command: 

```shell
cat test_data/kustomize.output | go run cmd/helmify/main.go mychart
```

Will generate `mychart` Helm chart form file `test_data/kustomize.output` representing typical 
[kustomize](https://github.com/kubernetes-sigs/kustomize) output.

## Integrate to your Operator-SDK project
Tested with operator-sdk version: "v1.8.0".
1. Open `Makefile` in your operator project generated by [Operator-SDK](https://github.com/operator-framework/operator-sdk).
2. Add these lines to `Makefile`:
```makefile
HELMIFY = $(shell pwd)/bin/helmify
helmify:
	$(call go-get-tool,$(HELMIFY),github.com/arttor/helmify/cmd/helmify@v0.1.0)

helm: manifests kustomize helmify
	$(KUSTOMIZE) build config/default | $(HELMIFY)
```
3. Run `make helm` in project root. It will generate helm chart with name 'chart' in 'chart' directory.

## Available options
Helmify takes a chart name for an argument.
Usage:

```helmify CHART_NAME [flags]```  -  `CHART_NAME` is optional. Default is 'chart'.

| flag | description | sample |
| --- | --- | --- |
| -h -help | Prints help | `helmify -h`|
| -v | Enable verbose output. | `helmify -v`|

## Status
Supported default operator resources:
- deployment
- service
- RBAC (serviceaccount, (cluster-)role, (cluster-)rolebinding)
- configmap



### Known issues
- Helmify will not overwrite `Chart.yaml` file if presented. Done on purpose.
- Helmify will not delete existing template file, only overwrite. So, if you delete CRD, re-run `kustomize | helmify` 
crd file will still be in templates directory. (todo: add option for this)
- Helmify overwrites templates and values files on every run. 
  This meas that all your manual changes in helm template files will be lost on the next run. 
  Use kustomize /config folder as a single source of true and make changes there.
