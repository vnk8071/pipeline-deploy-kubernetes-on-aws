# Pipeline deployment K8s cluster on AWS

## Prerequisites


## Installation
### Install AWS CLI
```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```
Reference: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

### AWS Configure
```bash
aws configure
export AWS_CONFIG_FILE=~/.aws/config
export AWS_SHARED_CREDENTIALS_FILE=~/.aws/credentials
```

### Install Eksctl
```bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

### Install Kubectl
```bash
brew install kubectl
```

### Monitoring Kubernetes Cluster
```bash
brew install k9s
```

### Install packages
```bash
pip install -r requirements.txt
```

## Build Application
### Run locally
```bash
python main.py
```

Request method `POST`:
```bash
export TOKEN=`curl --data '{"email":"khoivn@email.com","password":"mypwd"}' --header "Content-Type: application/json" -X POST localhost:8080/auth  | jq -r '.token'`
```

Response: `echo $TOKEN`
```bash
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2OTc5NjUyNjcsIm5iZiI6MTY5Njc1NTY2NywiZW1haWwiOiJraG9pdm5AZW1haWwuY29tIn0.hZr2RQqnroNdly_j9yRKpO7Z6yEwx7pNleFfB25IjHs
```

Request method `GET`:
```bash
curl --request GET 'http://localhost:8080/contents' -H "Authorization: Bearer ${TOKEN}" | jq .
```

Response:
```bash
{
  "email": "khoivn@email.com",
  "exp": 1697965267,
  "nbf": 1696755667
}
```

### Run Pytest
```bash
pytest test_main.py --disable-pytest-warnings
```

Result:
```python
=================================================================================== test session starts ====================================================================================
platform darwin -- Python 3.8.18, pytest-6.2.2, py-1.11.0, pluggy-0.13.1
rootdir: /Users/macos/projects/Kelvin/pipeline-deploy-kubernetes-on-aws
collected 2 items                                                                                                                                                                          

test_main.py ..                                                                                                                                                                      [100%]

============================================================================== 2 passed, 5 warnings in 0.22s ===============================================================================
```

### Build Docker Image
```bash
docker build -t token-flask-app .
```

### Run Docker Container
```bash
docker run --detach --publish 80:8080 --env-file=.env_file token-flask-app
```

### Test Docker Container
```bash
export TOKEN=`curl --data '{"email":"khoivn@email.com","password":"mypwd"}' --header "Content-Type: application/json" -X POST localhost:80/auth  | jq -r '.token'`
```

Result:
```bash
curl --request GET 'http://localhost:80/contents' -H "Authorization: Bearer ${TOKEN}" | jq .
```

Result"
```bash
{
  "email": "khoivn@email.com",
  "exp": 1697977237,
  "nbf": 1696767637
}
```

### Push Docker Image to Docker Hub
```bash
docker tag token-flask-app:latest vnk8071/token-flask-app:latest
docker push vnk8071/token-flask-app:latest
```

## Deploy Application to Kubernetes Cluster
### Create EKS Cluster
```bash
eksctl create cluster --name eksctl-demo-flask-token --nodes=2 --instance-types=t2.medium --region=us-east-1
```

![eksctl_create](images/eksctl_create.png)

Get cluster information:
```bash
kubectl get nodes
```
![get_nodes](images/get_nodes.png)

![k9s_deployment](images/k9s_deployment.png)

### Get current AWS account
```bash
aws sts get-caller-identity --query Account --output text 
```
Result: `993324276116`

### Create IAM Role and attach policy
```bash
aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document file://trust.json --output text --query 'Role.Arn'
```

Result: `arn:aws:iam::993324276116:role/UdacityFlaskDeployCBKubectlRole`

```bash
aws iam put-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file://iam-role-policy.json
```

### Create Kubernetes ConfigMap
```bash
kubectl get -n kube-system configmap/aws-auth -o yaml > /tmp/aws-auth-patch.yml
kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
```

Result: `configmap/aws-auth patched`

### Get GitHub Access Token
In GitHub, go to Settings > Developer settings > Personal access tokens > Generate new token

### Create CloudFormation Stack
1. Modify `ci-cd-codepipeline.cfn.yaml` file
2. Review the resources
3. Create stack