# Helm Chart for Hugging Face Generative AI Services (HUGS)

<img src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hugs/hugs-purple-no-bg.png" width="200" alt="HUGS Logo">

![Version: 0.0.1](https://img.shields.io/badge/Version-0.0.1-informational?style=flat-square)
![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square)
![AppVersion: 1.16.0](https://img.shields.io/badge/AppVersion-1.16.0-informational?style=flat-square)

## Installing the Chart

To add the chart from the current repository you need to run:

```console
$ helm repo add hugs https://raw.githubusercontent.com/huggingface/hugs-helm-chart/main/charts/hugs
$ helm repo update hugs
```

## Installing the Chart

Then to install the chart on e.g. AWS EKS you can run the following:

```console
$ helm install hugs-demo hugs/hugs -f aws/eks-values.yaml --set image.registry="XXXXXXXXXXXX.dkr.ecr.us-east-1.amazonaws.com"
```

> [!NOTE]
> Find more information about HUGS on AWS [here](./aws/).
