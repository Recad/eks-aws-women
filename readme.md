# EKS 
## _KUBERNETES + AWS_

## Desplegar nuestro primer Cluster de EKS.
Para el despliegue de nuestro cluster utilizaremos EKSCTL. 
Adicionalmente se requiere instalar la herramienta kubectl y helm.

***Crear nuestro cluster***
 ```sh
eksctl create cluster -f creation.yaml
```
nota: recuerda tener la ultima version de eksctl


***Guardar el nombre del Role que usara nuestro cluster***
 ```sh
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo "export ROLE_NAME=${ROLE_NAME}" | tee -a ~/.bash_profile
```

O pueden buscar en la consola de aws en el servicio de cloudformation 

***Ingresar a nuestro cluster***
 ```sh
 aws eks --region us-east-1 update-kubeconfig --name aws-demo 
```
***Añadir un proveedor de OIDC a nuestro cluster***
 ```sh
 eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=aws-demo --approve 
```

***Añadir el complemento de vpc cni***
 ```sh
 eksctl create addon  --name vpc-cni --version latest --cluster aws-demo 
```
 
***Ver nuestros nodos***
```sh
 kubectl get nodes -o wide
```
***Crear nuestro namespace de dev***
```sh
 kubectl create ns dev
```

***Desplegar nuestro primer microservicio***
```sh
  kubectl apply -f .\deployment_v1.yaml
```
nota: No te asustes si no despliega tu pod, esto lo cubriremos mas adelante.

***Revisar el estado de nuestros pods desplegados***
```sh
  kubectl get pod -n dev
```
veras los nodos en un estado de ErrImagePull esto se debe a la falta de permisos por parte de los nodos a nuestro repositorio de ECR. Debes añadir permisos a ese rol para poder hacer pull de las imágenes que necesitamos.

## INTEGRACIÓN INGRESS-ALB.

Instalación del complemento alb ingress controller.
Para esto debe tener instalado HELM.

***Instalar el complemento para aws-load-balancer***
```sh
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm repo add eks https://aws.github.io/eks-charts
helm install  eks/aws-load-balancer-controller --generate-name   --set autoDiscoverAwsRegion=true --set autoDiscoverAwsVpcID=true --set clusterName=<cluster name> --namespace <namespace>
```

***Desplegar el ingress***
```sh
kubectl apply -f ingress.yaml
```

## UN ROL PARA NUESTROS PODS

## SERVICEACCOUNT.
El service account permite autenticar un pod contra el plano de control y en el caso de eks se pueden usar para que los pods puedan "asumir" un rol de iam.

***Crear el sa***
```sh
kubectl create sa service-dynamo -n dev
```

***Ver el oidc issuer***
```sh
aws eks describe-cluster --name <mi_cluster_name> --query "cluster.identity.oidc.issuer" --output text
```

***Usar el oidc issuer para mi cluster***
```sh
eksctl utils associate-iam-oidc-provider --cluster <mi cluster> --approve
```

***crear un rol para que lo asuman los pods***
el archivo trust.json debe tener la siguiente estructura
```json
{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Federated": "arn:aws:iam::<aws_account_id>:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/<ID OIDC ISSUER>"
        },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
          "StringEquals": {
            "oidc.eks.us-east-1.amazonaws.com/id/<ID OIDC ISSUER>:sub": "system:serviceaccount:<NAMESPACE>:<SA NAME>"
          }
        }
      }
    ]
  }
```

```sh
aws iam create-role --role-name <IAM_ROLE_NAME> --assume-role-policy-document file://trust.json --description "<IAM_ROLE_DESCRIPTION>"
```

***añadir una politica al rol que asumiran los pods***
recuerda que la politica puede ser una creada por ti
```sh
aws iam attach-role-policy --role-name <IAM_ROLE_NAME> --policy-arn=arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess
```

```sh
aws iam attach-role-policy --role-name <IAM_ROLE_NAME> --policy-arn=arn:aws:iam::aws:policy/AmazonEC2FullAccess
```



***firmar serviceaccount para que use el rol***
```sh
kubectl annotate serviceaccount -n <namespace> <serviceaccountname> eks.amazonaws.com/role-arn=arn:aws:iam::692137641826:role/<role_name>
```

después de esto debes añadir a tus deployments el uso de un service account.

## AUTO ESCALADO DE PODS

para poder hacer uso del hpa debes tener instalado el metric-server en tu cluster de eks.

***Instalar metric-server con helm***
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install metric-server bitnami/metrics-server
helm upgrade --namespace default metric-server bitnami/metrics-server --set apiService.create=true
```

***Instalar metric-server con apply***
```sh
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.0/components.yaml

```

***Desplegar un hpa***
```sh
kubectl apply -f hpa.yaml
```

***Entrar a un pod***
```sh
kubectl exec -it <pod_name> bash -n <namespace>
```

***Generar aumento en consumo de cpu dentro del pod***
```sh
while true; do true; done
```
## AUTO ESCALADO DE CLUSTER

Se debe instalar el cluster autoscaler como un deployment. en este repositorio ya se encuentra el deployment listo


```sh
kubectl apply -f cluster_autoscaler.yaml
```

Ahora nuestros pods pueden escalar sin problema