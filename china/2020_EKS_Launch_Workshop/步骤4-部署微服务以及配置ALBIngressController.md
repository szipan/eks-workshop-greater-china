# 步骤4 部署微服务以及配置ALB Ingress Controller

4.1 参考 eksworkshop的样例，部署微服务 (可选)
* [eksworkshop的样例微服务] (https://eksworkshop.com/beginner/050_deploy/)

> 4.1.1 下载样例
```bash
## Ruby Frontend
git clone https://github.com/brentley/ecsdemo-frontend.git
## NodeJS Backend and crystal backend
git clone https://github.com/brentley/ecsdemo-nodejs.git
git clone https://github.com/brentley/ecsdemo-crystal.git
```

> 4.1.2 部署后台
```bash
 cd ecsdemo-nodejs 
 kubectl apply -f kubernetes/deployment.yaml
 kubectl apply -f kubernetes/service.yaml
 # 检查部署是否正确
 kubectl get deployment ecsdemo-nodejs
 #
 cd ../ecsdemo-crystal
 kubectl apply -f kubernetes/deployment.yaml
 kubectl apply -f kubernetes/service.yaml
 # 检查部署是否正确
 kubectl get deployment ecsdemo-crystal
```

> 4.1.3 部署前台
```bash
 # 检查ELB Service Role以及在您的账号下创建，如果没有创建，请参考AWS文档进行创建
 aws iam get-role --role-name "AWSServiceRoleForElasticLoadBalancing" --region ${AWS_REGION}
 # 部署
 cd ../ecsdemo-frontend
 kubectl apply -f kubernetes/deployment.yaml
 kubectl apply -f kubernetes/service.yaml
 kubectl get deployment ecsdemo-frontend
 # 检查状态
 kubectl get service ecsdemo-frontend -o wide
 # 访问前端服务
 ELB=$(kubectl get service ecsdemo-frontend -o json | jq -r '.status.loadBalancer.ingress[].hostname')
echo ${ELB}
 # 浏览器访问或者通过curl命令进行验证
 curl -m3 -v $ELB
```

> 微服务部署扩展
我们发现集群并不是跨多节点的高可用的架构，因此我们需要对部署进行扩展

```bash
 # 每一个微服务目前都只有一个部署单元
 kubectl get deployments
 # NAME               READY   UP-TO-DATE   AVAILABLE   AGE
 # ecsdemo-crystal    1/1     1            1           19m
 # ecsdemo-frontend   1/1     1            1           7m51s
 # ecsdemo-nodejs     1/1     1            1           24m

 # scale 到3个replicas
 kubectl scale deployment ecsdemo-nodejs --replicas=3
 kubectl scale deployment ecsdemo-crystal --replicas=3
 kubectl scale deployment ecsdemo-frontend --replicas=3

 kubectl get deployments
 # NAME               READY   UP-TO-DATE   AVAILABLE   AGE
 # ecsdemo-crystal    3/3     3            3           21m
 # ecsdemo-frontend   3/3     3            3           9m51s
 # ecsdemo-nodejs     3/3     3            3           26m
```

> 清除资源
```bash
 cd ../ecsdemo-frontend
 kubectl delete -f kubernetes/service.yaml
 kubectl delete -f kubernetes/deployment.yaml
 cd ../ecsdemo-crystal
 kubectl delete -f kubernetes/service.yaml
 kubectl delete -f kubernetes/deployment.yaml
 cd ../ecsdemo-nodejs
 kubectl delete -f kubernetes/service.yaml
 kubectl delete -f kubernetes/deployment.yaml
```

4.2 使用ALB Ingress Controller

参考文档 

https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html

https://aws.amazon.com/cn/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/

> 4.2.1 创建ALB Ingress Controller所需要的IAM policy , EKS OIDC provider, service account

> 4.2.1.1 创建EKS OIDC Provider (这个操作每个集群只需要做一次）

```bash
eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}
[ℹ]  eksctl version 0.15.0-rc.1
[ℹ]  using region cn-northwest-1
[ℹ]  will create IAM Open ID Connect provider for cluster "eksworkshop" in "cn-northwest-1"
[✔]  created IAM Open ID Connect provider for cluster "eksworkshop" in "cn-northwest-1"
```

> 4.2.1.2 创建所需要的IAM policy
[https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.5/docs/examples/iam-policy.json](https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.5/docs/examples/iam-policy.json)
 * 请注意官方的policy里面包含了WAF等服务，中国区没有所以需要手动删除,修改好的已经放在resource/alb-ingress-controller目录下

```bash
cd ~/eks-workshop-greater-china/china/2020_EKS_Launch_Workshop
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://./resource/alb-ingress-controller/iam_policy.json --region ${AWS_REGION}

# 记录返回的Plociy ARN
POLICY_NAME=$(aws iam list-policies --query 'Policies[?PolicyName==`AWSLoadBalancerControllerIAMPolicy`].Arn' --output text --region ${AWS_REGION})

```

>4.2.1.3 请使用上述返回的policy ARN创建service account

```bash
eksctl create iamserviceaccount \
       --cluster=${CLUSTER_NAME} \
       --namespace=kube-system \
       --name=aws-load-balancer-controller \
       --attach-policy-arn=${POLICY_NAME} \
       --override-existing-serviceaccounts \
       --approve

参考输出
[ℹ]  eksctl version 0.15.0-rc.2
[ℹ]  using region cn-northwest-1
[ℹ]  1 iamserviceaccount (kube-system/alb-ingress-controller) was included (based on the include/exclude rules)
[!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
[ℹ]  1 task: { 2 sequential sub-tasks: { create IAM role for serviceaccount "kube-system/alb-ingress-controller", create serviceaccount "kube-system/alb-ingress-controller" } }
[ℹ]  building iamserviceaccount stack "eksctl-eksworkshop-addon-iamserviceaccount-kube-system-alb-ingress-controller"
[ℹ]  deploying stack "eksctl-eksworkshop-addon-iamserviceaccount-kube-system-alb-ingress-controller"
[ℹ]  created serviceaccount "kube-system/alb-ingress-controller"
```



4.3 部署 Amazon Load Balancer Controller
首先参照步骤9，安装helm

 helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=${CLUSTER_NAME} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set enableShield=false \
  --set enableWaf=false \
  --set enableWafv2=false

验证Amazon Load Balancer Controller安装成功
kubectl get deployment -n kube-system aws-load-balancer-controller


 #参考输出
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           84s

  ```

4.4 使用ALB Ingress，部署2048 game (可选）

注意，默认已经使用2.4章节自动修改image mirror的webhook，否则请修改Image地址为国内可以访问的。

```bash
curl -s https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.3.1/docs/examples/2048/2048_full_latest.yaml \
    | sed 's=alb.ingress.kubernetes.io/target-type: ip=alb.ingress.kubernetes.io/target-type: instance=g' \
    | kubectl apply -f -

# 获取访问地址，在浏览器中访问2048游戏
kubectl get ingress/ingress-2048 -n game-2048

#删除游戏
curl -s https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.3.1/docs/examples/2048/2048_full_latest.yaml \
    | sed 's=alb.ingress.kubernetes.io/target-type: ip=alb.ingress.kubernetes.io/target-type: instance=g' \
    | kubectl delete -f -
```

