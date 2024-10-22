# HUGS on AWS

The Hugging Face Generative AI Services, also known as HUGS, can be deployed in Amazon Web Services (AWS) via the AWS Marketplace offering.

This collaboration brings Hugging Face's extensive library of pre-trained models and their Text Generation Inference (TGI) solution to AWS customers, enabling seamless integration of state-of-the-art Large Language Models (LLMs) within the AWS infrastructure.

HUGS provides access to a hand-picked and manually benchmarked collection of the most performant and latest open LLMs hosted in the Hugging Face Hub to TGI-optimized container applications, allowing users to deploy third-party Kubernetes applications on AWS or on-premises environments.

With HUGS, developers can easily find, subscribe to, and deploy Hugging Face models using AWS infrastructure, leveraging the power of NVIDIA GPUs on optimized, zero-configuration TGI containers.

## Subscribe to HUGS on AWS Marketplace

1. Go to [HUGS AWS Marketplace listing](https://aws.amazon.com/marketplace/pp/prodview-bqy5zfvz3wox6)

   ![HUGS on AWS Marketplace](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hugs/aws/hugs-marketplace-listing.png)

2. Subscribe to the product in AWS Marketplace by following the instructions on the page. At the time of writing (September 2024), the steps are to:

   1. Click `Continue to Subscribe`, then go to the next page.
   2. Click `Continue to Configuration`, then go to the next page.
   3. Select the fulfillment option & software version (HUGS Version and model you want to use) from the list.

   ![HUGS Configuration on AWS Marketplace](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hugs/aws/hugs-configuration.png)

3. Then click `Continue to Launch`. You successfully subscribe to HUGS. You can now follow the steps below to deploy your preferred HUGS container and model using AWS EKS.

> [!NOTE]
> To know whether you are subscribed or not, you can either see if a blue modal appears on top of the product page with a text saying "You have access to this product", meaning that either you or someone else from your organization has already requested access for your account; otherwise, you can go to the AWS Marketplace service in the AWS Console and search for "HUGS (HUgging Face Generative AI Services)" is listed among your subscribed products.

## Deploy HUGS on AWS EKS

This example showcases how to create a Kubernetes Cluster on AWS EKS, how to create a node group with the necessary permissions and compute requirements, and how to deploy HUGS on AWS EKS using a Helm template.

> [!NOTE]
> This example assumes that you have an AWS Account, that you have [installed and setup the AWS CLI](https://docs.aws.amazon.com/eks/latest/userguide/install-awscli.html), and that you are logged in into your account with the necessary permissions to subscribe to offerings in the AWS Marketplace, and create and manage IAM permissions and resources such as Elastic Kubernetes Service (EKS), Elastic Container Service (ECS), and EC2.

### Requirements

Before proceeding, you need to have installed both `kubectl`, `eksctl`, and `helm`, to interact with the Kubernetes Cluster, to create, configure and delete resources on AWS EKS, and to interact with the Helm templates, respectively.

To install both `kubectl` and `eksctl` you can follow the instructions at [AWS EKS Documentation - Set up kubectl and eksctl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html). Whilst for `helm` you can follow the instructions at [Helm Documentation - Installing Helm](https://helm.sh/docs/intro/install/).

Finally, for convenience the following environment variables will be set:

```bash
export REGION="us-east-1"
export NAMESPACE="default"
export CLUSTER_NAME="hugs-cluster"
export NODE_GROUP_NAME="hugs-node-group"
export SERVICE_ACCOUNT_NAME="hugs-service-account"
export DEPLOYMENT_NAME="hugs"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

> [!NOTE]
> If your AWS Account has multiple profiles remember to set the `AWS_PROFILE` environment variable to your profile so that the EKS commands use the correct profile.

### Setup AWS EKS

To create the EKS Cluster, Node Group, and add the necessary IAM permissions, you should run `eksctl` with the provided configuration file `eks-cluster.yaml` that looks like:

```yaml
# Specifies the API version and kind of the configuration
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

# Defines the basic cluster metadata
metadata:
  name: $CLUSTER_NAME # Cluster name, used in various AWS resource names
  region: $REGION # AWS region where the cluster will be created
  version: "1.30" # Kubernetes version to use for the cluster

# IAM configuration for the cluster
iam:
  withOIDC: true # Enables IAM roles for service accounts (IRSA) using OIDC
  serviceAccounts:
    # Configures a service account for marketplace metering
    - metadata:
        name: $SERVICE_ACCOUNT_NAME
        namespace: $NAMESPACE
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AWSMarketplaceMeteringRegisterUsage
    # Configures a service account for the AWS Load Balancer Controller (just required
    # if ingress is enabled within the HUGS Helm Template)
    - metadata:
        name: aws-load-balancer-controller
        namespace: $NAMESPACE
      attachPolicyARNs:
        - arn:aws:iam::$AWS_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy
      roleName: AmazonEKSLoadBalancerControllerRole

# Defines the managed node group for the cluster
managedNodeGroups:
  - name: $NODE_GROUP_NAME
    instanceType: g5.xlarge # GPU-enabled instance type, required by HUGS
    minSize: 1
    maxSize: 2 # Set to a greater number if you want to enable auto-scaling
    desiredCapacity: 1 # Fixed size node group, can be adjusted for scaling

# Specifies the EKS add-ons to be installed (default ones)
# All the addons below will be installed within the `kube-system` namespace
addons:
  - name: vpc-cni # Amazon VPC CNI plugin for Kubernetes
  - name: coredns # CoreDNS for Kubernetes DNS services
  - name: kube-proxy # kube-proxy for Kubernetes network proxy

# Configures CloudWatch logging for the cluster
cloudWatch:
  clusterLogging:
    enableTypes: ["*"] # Enables all types of control plane logging
```

Optionally, before creating the cluster you may need to create the IAM Policy required to enable the AWS LoadBalancer Controller i.e. `AWSLoadBalancerControllerIAMPolicy`, so as to deploy the ingress which requires the AWS LoadBalancer Controller to be enabled, as per [Install AWS Load Balancer Controller with Helm - Step 1: Create IAM Role using eksctl](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html#lbc-helm-iam).

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

> [!NOTE]
> Note that the policy just needs to be created once per account, meaning that if it's already created then you can reuse the existing one. Otherwise, you can alternatively create the policy under a different name, but along this example the policy is assumed to be named `AWSLoadBalancerControllerIAMPolicy`.

Alternatively, if you decided to skip the AWS LoadBalancer creation above, then you should remove the `iam.serviceAccounts` for the `aws-load-balancer-controller` before moving on to the next step.

Then you need to run the following command, that will replace the values set in the environment variables above within the `eks-cluster.yaml` file. To replace the environment variable values within the `eks-cluster.yaml` file above, [`envsubst`](https://linux.die.net/man/1/envsubst) will be used, which may not be available for Windows users.

```bash
envsubst < eks-cluster.yaml > eks-cluster.yaml
```

Once the `eks-cluster.yaml` file has been updated, then you can just run the following command:

```bash
eksctl create cluster --config-file eks-cluster.yaml
```

If you want to enable or use the AWS LoadBalancer, you will need to deploy that separately before deploying HUGS, in order to enable the ingress.

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    --namespace $NAMESPACE \
    --set clusterName=$CLUSTER_NAME \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller
```

> [!NOTE]
> If you decide to deploy the AWS LoadBalancer Controller and enable the ingress within the HUGS deployment, then you will need to wait until the ALB controller is running before deploying HUGS, to do so you can use the following `kubectl` command:
>
> ```bash
> kubectl wait --namespace $NAMESPACE \
>     --for=condition=ready pod \
>     --selector=app.kubernetes.io/name=aws-load-balancer-controller \
>     --timeout=90s
> ```

### Deploy HUGS with Helm on AWS EKS

Finally, you can install the Helm template to deploy the HUGS container with the selected model, e.g. `meta-llama/Llama-3.1-8B-Instruct`; and you need to set the container URI provided by the AWS Marketplace for your account, either via the `--set` option when running `helm install` (as shown below), or modifying its value within the `eks-values.yaml` file provided.

```bash
helm repo add hugs https://raw.githubusercontent.com/huggingface/hugs-helm-chart/main/charts/hugs
helm repo update hugs
helm install $DEPLOYMENT_NAME hugs/hugs \
    -f eks-values.yaml \
    --set image.registry="XXXXXXXXXXXX.dkr.ecr.us-east-1.amazonaws.com" \
    --set image.repository="hugging-face" \
    --set image.name="nvidia-meta-llama-meta-llama-3.1-8b-instruct" \
    --set image.tag="0.1.0" \
    --set serviceAccountName=$SERVICE_ACCOUNT_NAME \
    --set nodeSelector."eks\.amazonaws\.com/nodegroup"=$NODE_GROUP_NAME
```

> [!NOTE]
> The command above assumes that you followed the steps within this example, if you changed the node group name, the service account name, the container, or the number of accelerators; you should manually modify or create a new `values.yaml` file out of [`eks-values.yaml`](eks-values.yaml) with your custom settings.

### Inference on HUGS

To run the inference over the deployed HUGS service, you can either:

- Forward the port via port-forwarding to a local port as e.g. 8080 (so that you can send requests to the service via `localhost`) with the command:

  ```bash
  kubectl port-forward service/$DEPLOYMENT_NAME 8080:80
  ```

- Use the external IP or hostname of the ingress, if `ingress.enabled: true`, that can be retrieved with the following command:

  ```bash
  kubectl get ingress $DEPLOYMENT_NAME -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
  ```

Then you can send requests to the Messages API via either `localhost`, the ingress IP or the ingress hostname, from outside the running pod.

The Messages API is an OpenAI-compatible endpoint under `/v1/chat/completions` following the [OpenAI OpenAPI Specification](https://github.com/openai/openai-openapi). Being OpenAI-compatible implies that the inference can be run not only with `cURL` but also with both the `huggingface_hub.InferenceClient` and the `openai.OpenAI` SDK in Python, as well as any other OpenAI-compatible SDK in any programming language.

> [!NOTE]
> In the inference examples show below, the host is assumed to be `localhost` which is the case when deploying HUGS via Kubernetes with port-forwarding. If you have deployed HUGS on Kubernetes using an ingress under a specific IP, host, and/or with SSL (HTTPS), note that you should update the `localhost` references below with your host or IP.

#### cURL

Using `cURL` is pretty straight forward to [install](https://curl.se/docs/install.html) and use.

```bash
curl http://localhost:8080/v1/chat/completions \
    -X POST \
    -d '{"messages":[{"role":"user","content":"What is Deep Learning?"}],"temperature":0.7,"top_p":0.95,"max_tokens":128}}' \
    -H 'Content-Type: application/json'
```

#### Python

As already mentioned, you can either use the `huggingface_hub.InferenceClient` from the `huggingface_hub` Python SDK (recommended), the `openai` Python SDK, or any SDK with an OpenAI-compatible interface that can consume the Messages API.

##### `huggingface_hub`

You can install it via pip as `pip install --upgrade --quiet huggingface_hub`, and then run the following snippet to mimic the `cURL` commands above i.e. sending requests to the Messages API:

```python
from huggingface_hub import InferenceClient

client = InferenceClient(base_url="http://localhost:8080", api_key="-")

chat_completion = client.chat.completions.create(
    messages=[
        {"role":"user","content":"What is Deep Learning?"},
    ],
    temperature=0.7,
    top_p=0.95,
    max_tokens=128,
)
```

Read more about the [`huggingface_hub.InferenceClient.chat_completion` method](https://huggingface.co/docs/huggingface_hub/en/package_reference/inference_client#huggingface_hub.AsyncInferenceClient.chat_completion).

##### `openai`

Alternatively, you can also use the Messages API via `openai`; you can install it via `pip as pip install --upgrade openai`, and then run:

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8080/v1/", api_key="-")

chat_completion = client.chat.completions.create(
    model="tgi",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is Deep Learning?"},
    ],
    temperature=0.7,
    top_p=0.95,
    max_tokens=128,
)
```
