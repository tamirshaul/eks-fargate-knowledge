## ALB INGRESS CONTROLLER
---

1. Create an IAM policy called ALBIngressControllerIAMPolicy for your worker node instance profile that allows the ALB Ingress Controller to make calls to AWS APIs on your behalf. Use the following AWS CLI commands to create the IAM policy in your AWS account. You can view the policy document on GitHub.

Download the policy document from GitHub.
```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/iam-policy.json
```

Create the policy.
```bash
aws iam create-policy \
--policy-name ALBIngressControllerIAMPolicy \
--policy-document file://iam-policy.json
```

<b>Take note of the policy ARN that is returned.</b>

2. Get the IAM role name for your worker nodes. Use the following command to print the aws-auth configmap.
```bash
kubectl -n kube-system describe configmap aws-auth
```
Output:
```
Name:         aws-auth
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
mapRoles:
----
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::111122223333:role/eksctl-alb-nodegroup-ng-b1f603c5-NodeInstanceRole-GKNS581EASPU
  username: system:node:{{EC2PrivateDNSName}}

Events:  <none>
```
Record the role name for any ```rolearn``` values that have the ```system:nodes``` group assigned to them. In the above example output, the role name is ```eksctl-alb-nodegroup-ng-b1f603c5-NodeInstanceRole-GKNS581EASPU```. You should have one value for each node group in your cluster.

Also, record the AWS account number after ```iam::```. In the above example output, the number is ```111122223333```

3. Attach the new ALBIngressControllerIAMPolicy IAM policy to each of the worker node IAM roles you identified earlier with the following command, substituting the text in '<>' with your own AWS account number and worker node IAM role name.
```bash
aws iam attach-role-policy \
--policy-arn arn:aws:iam::<111122223333>:policy/ALBIngressControllerIAMPolicy \
--role-name <eksctl-alb-nodegroup-ng-b1f603c5-NodeInstanceRole-GKNS581EASPU>
```

4. Create a service account, cluster role, and cluster role binding for the ALB Ingress Controller to use with the following command.
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/rbac-role.yaml
```
5. Deploy the ALB Ingress Controller with the following command.
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/alb-ingress-controller.yaml
```

7. Open the ALB Ingress Controller deployment manifest for editing with the following command.
```bash
kubectl edit deployment.apps/alb-ingress-controller -n kube-system
```

Add the cluster name, VPC ID, and AWS Region name for your cluster after the --ingress-class=alb line and then save and close the file.
```bash
    spec:
      containers:
      - args:
        - --ingress-class=alb
        - --cluster-name=<my_cluster>
        - --aws-vpc-id=<vpc-03468a8157edca5bd>
        - --aws-region=<us-east-2>
```

8. set up the OIDC ID provider (IdP) in AWS:
```bash
eksctl utils associate-iam-oidc-provider --name <cluster-name> --approve
```

9. Attach the correct IAM Policy to the ```alb-ingress-controller``` ServiceAccount so it will be able to create LoadBalancers via AWS API.
Remembeer to replace the ```AWS Account number``` (```111122223333```) and the ```cluster-name``` with yours.
```bash
eksctl create iamserviceaccount --name alb-ingress-controller \
--namespace kube-system \
--cluster <cluster-name>  \
--attach-policy-arn arn:aws:iam::<111122223333>:policy/ALBIngressControllerIAMPolicy \
--approve \
--override-existing-serviceaccounts
```

10. Restart the ```alb-ingress-controller``` by deleting the pod:
```bash
kubectl scale -n kube-system deployment.apps/alb-ingress-controller --replicas=0
kubectl scale -n kube-system deployment.apps/alb-ingress-controller --replicas=1
```

#### Test your ingress
---

1. Deploy a sample nginx app:
```bash
kubectl create deployment nginx --image=nginx
kubectl create service clusterip nginx --tcp=80:80
```

2. Create an Ingress yaml file:
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: "nginx-ingress"
  namespace: "default"
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
  labels:
    app: nginx
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: "nginx"
              servicePort: 80
```

Apply the file above:
```bash
kubectl apply -f nginx-ingress.yaml
```
Check the ingress status:
```bash
kubectl get ingress nginx-ingress
```

After a few minutes try to reach the url written under ```ADDRESS``` via ```HTTP```. 
nginx welcome page should be displayed.

#### Debugging
---

If after a few minutes the nginx welcome page isn't showing try to look at the alb-ingress-controller logs:
```bash
kubectl logs -n kube-system   deployment.apps/alb-ingress-controller
```
