# eks-alb-ingress	

This document walks you through Amazon EKS with [CoreOS ALB ingress conbtroller](https://github.com/coreos/alb-ingress-controller).



### Getting Started

Attach extra IAM policy allowing all **elasticloadbalancing:*** method to the EC2 node instance role. The ingress controller will need 

```
aws iam put-role-policy --role-name <EC2_NODE_INSTANCE_ROLE> --policy-name alb-allow --policy-document file://alb-inline-iam-policy.json
```







Install the default-http-backend

```bash
$ kubectl apply -f https://raw.githubusercontent.com/coreos/alb-ingress-controller/master/examples/default-backend.yaml
```

modify the `alb-ingress-controller.yaml` file:

- `AWS_REGION`: Region of your Amazon EKS cluster.

  ```yaml
  - name: AWS_REGION
    value: us-west-2
  ```

  ​

- `CLUSTER_NAME`: name of the cluster

  ```yaml
  - name: CLUSTER_NAME
    value: mycluster
  ```



Create the `ClusterRole`, `ClusterRoleBinding` and `ServiceAccount`

```bash
$ kubectl apply -f albrbac.yaml
```



Deploy the ingress-controller

```bash
$ kubectl apply -f alb-ingress-controller.yaml
```

Verify the deployment was successful and the controller started.

```bash
$ kubectl logs -n kube-system \
    $(kubectl get po -n kube-system | \
    egrep -o alb-ingress[a-zA-Z0-9-]+) | \
    egrep -o '\[ALB-INGRESS.*$'
```

Create the sample application

```bash
$ kubectl apply -f app.yaml
```

Update the ingress resource

```bash
$ vim ingress-resource.yaml
```



```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: "webapp-alb-ingress"
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS": 443}]'
    alb.ingress.kubernetes.io/subnets: 'subnet-xxxxxxxx,subnet-xxxxxxxx'
    alb.ingress.kubernetes.io/security-groups: 'sg-xxxxxxxx,sg-xxxxxxxx'
    alb.ingress.kubernetes.io/certificate-arn: <ACM_CERT_ARN>
  labels:
    app: webapp-service
spec:
  rules:
  - http:
      paths:
      - path: /greeting
        backend:
          serviceName: "webapp-service"
          servicePort: 80
      - path: /
        backend:
          serviceName: "caddy-service"
          servicePort: 80
```

1. make sure to modify the `subnet` `security-groups` and `certificate-arn` if required.
2. make sure public internet can access ALB TCP 80 and 443 and ALB can access any TCP port on node group - double check your security groups setting. You would usually need two security groups - one for TCP 80 and 443 public from all and the other for the NodeSecurityGroup.



find your EKS NodeSecurityGroup

```bash
$ aws ec2 describe-security-groups --query "SecurityGroups[?VpcId=='vpc-e692c79f']|[?contains(GroupName, 'NodeSecurityGroup')].GroupId"

[
    "sg-49c86737"
]
```



find your EKS subnets with `aws cli` :

```bash
$ aws ec2 describe-subnets --query "join(',', Subnets[?VpcId=='vpc-e692c79f'].SubnetId)" --output text
subnet-eb16cba0,subnet-7ef24007
```



Deploy the ingress resource

```Bash
$ kubectl apply -f ingress-resource.yaml
ingress "webapp-alb-ingress" created
```

Describe your ingress resource

```bash
$ kubectl describe ing/webapp-alb-ingress
Name:             webapp-alb-ingress
Namespace:        default
Address:          mycluster-default-webappal-9895-232573660.us-west-2.elb.amazonaws.com
Default backend:  default-http-backend:80 (192.168.109.73:8080)
Rules:
  Host  Path  Backends
  ----  ----  --------
  *
        /greeting   webapp-service:80 (<none>)
        /           caddy-service:80 (<none>)
Annotations:
Events:
  Type    Reason  Age              From                Message
  ----    ------  ----             ----                -------
  Normal  CREATE  2m               ingress-controller  Ingress default/webapp-alb-ingress
  Normal  CREATE  2m               ingress-controller  mycluster-default-webappal-9895 created
  Normal  CREATE  2m               ingress-controller  mycluster-32235-HTTP-7c52fdc target group created
  Normal  CREATE  2m               ingress-controller  mycluster-30214-HTTP-7c52fdc target group created
  Normal  CREATE  2m               ingress-controller  80 listener created
  Normal  CREATE  2m (x2 over 2m)  ingress-controller  1 rule created
  Normal  CREATE  2m (x2 over 2m)  ingress-controller  2 rule created
  Normal  CREATE  2m               ingress-controller  443 listener created
  Normal  UPDATE  1m               ingress-controller  Ingress default/webapp-alb-ingress
```



After a few minutes of DNS propagation of your ALB, you should be able to test it like this:



```bash
$ curl "http://<YOUR_ALB_DNS_NAME>/greeting?name=pahud"
Hello pahud
```



and the root path will go to [Caddy](https://caddyserver.com/) web server document root:



```bash
$ curl http://<YOUR_ALB_DNS_NAME>
```



