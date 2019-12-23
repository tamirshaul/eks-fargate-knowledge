## Installation
---

This command will create a serverless EKS cluster and initialize fargate so the ```coredns``` pods will be deployed on fargate as well.

This will also configure that every pod that scheduled to the ```default``` and ```kube-system namespaces``` will be on fargate too.
```bash
eksctl create cluster --name my-cluster --version <1.14-or_above> --fargate --vpc-cidr <10.0.0.0/16>
```

To add more namespaces or schedule pods by labels on Fargate you can add ```fargate profiles``` at:

```Services -> EKS -> <your-cluster> -> Add Fargate profile```


