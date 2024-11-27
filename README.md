# Helm Chart for Hugging Face Generative AI Services (HUGS)

<img src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hugs/hugs-purple-no-bg.png" width="200" alt="HUGS Logo">

![Version: 0.0.4](https://img.shields.io/badge/Version-0.0.4-informational?style=flat-square)
![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square)
![AppVersion: 1.16.0](https://img.shields.io/badge/AppVersion-1.16.0-informational?style=flat-square)

> [!WARNING]
> HUGS Helm Chart is subject to changes before the 0.1.0 release!

## Installing the Chart

To add the chart from the current repository you need to run:

```console
$ helm repo add hugs https://raw.githubusercontent.com/huggingface/hugs-helm-chart/main/charts/hugs
$ helm repo update hugs
```

Then to install the chart on a Kubernetes cluster you can run the following:

```console
$ helm install hugs-demo hugs/hugs \
    --set image.repository="hfhugs" \
    --set image.name="nvidia-meta-llama-meta-llama-3.1-8b-instruct" \
	--set resources.requests.nvidia\\.com/gpu=1 \
	--set resources.limits.nvidia\\.com/gpu=1
```
