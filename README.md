# k8s-metadata-injector

## Introduction

Labels and annotations are important to classify Kubernetes resources. Further, there are some annotations that will tag the corresponding AWS resources created by the Kubernetes itself.

The `k8s-metadata-injector` has two goals:

* Inject additional labels and annotations to `pods`, `services` and `persistentvolumeclaims` based on predefined config per namespace.
* Add tags to created AWS EBS volumes created by `persistentvolumeclaims`

You can add tags to EBS volumes by setting the following annotations in `persistentVolumeClaim` or `volumeClaimTemplate` as follow (example):

```yaml
ebs-tagger.kubernetes.io/ebs-additional-resource-tags: "Team=devops,Env=prod,Project=k8s"
```

Further you can automatically inject this annotation into any `persistentVolumeClaim` or `volumeClaimTemplate`
by adding the annotation in `persistentVolumeClaim` of any namespace config for `k8s-metadata-injector` as shown in `deployment/cm.yaml`

The reason to supports these kind of resources is their importance to cost and usage estimations such that:
* pod will correspond to cpu/mem usages on AWS EC2s
* services will be correspond to AWS Load balancers
* persistentvolumeclaims will correspond to EBS volumes

The ability to automatically inject labels and annotations to the previous resources based on namespaces will simplify the classification and grouping of cluster resources that corresponds to AWS resources. It could be used with other tools for cost estimations like Cloudhealth.

To exclude resources from metadata injection you can use the following annotation:

```yaml
k8s-metadata-injector.kubernetes.io/skip": "true"
```

**Note:** the `kube-system` and `kube-public` namespaces are excluded from injection.

## Installation

To install `k8s-metadata-injector`:
* Ensure that MutatingAdmissionWebhook admission controllers are enabled.
* Ensure that the admissionregistration.k8s.io/v1beta1 API is enabled.

**Warning:** The `webhook-create-signed-cert.sh` and `webhook-patch-ca-bundle.sh` scripts will use `kubectl` default context. So if there are multiple clusters configured in kubeconfig, then you have to change the default context to the cluster where `k8s-metadata-injector` will be deployed.

First generate and deploy the required certificate for `k8s-metadata-injector` as follow:

```bash
./deployment/webhook-create-signed-cert.sh \
            --service k8s-metadata-injector \
            --namespace kube-system \
            --secret k8s-metadata-injector
```

Then modify the config in `cm.yaml` as desired to inject the annotations and labels to all defined namespaces, and deploy:

```bash
kubectl apply -f deployment/cm.yaml
```

Finally replacing `${CA_BUNDLE}` with cluster CA certificate and install `k8s-metadata-injector` as follows:

```bash
cat deployment/install.yaml | \
    ./deployment/webhook-patch-ca-bundle.sh | \
    kubectl apply -f -
```

### Required IAM policy:
To tag EBS volumes based on `ebs-tagger.kubernetes.io/ebs-additional-resource-tags` annotation, the following policy is needed:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "ec2:CreateTags"
            ],
            "Resource": "arn:aws:ec2:*:*:volume/*",
            "Effect": "Allow"
        }
    ]
}
```

In `deployment/install.yaml` kube2iam annotation `iam.amazonaws.com/role: KubernetesEBSCreateTagsAccess` is used as an example to allow `ebs-tagger` (small controller in k8s-metadata-injector) to tag EBS volumes.

## Build

To build `k8s-metadata-injector` as a docker container:

```bash
CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o build/k8s-metadata-injector .
docker build -t abdullahalmariah/k8s-metadata-injector:latest build/
docker push abdullahalmariah/k8s-metadata-injector:latest
```
