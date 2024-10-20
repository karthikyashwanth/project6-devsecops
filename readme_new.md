eksctl installation
```
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo mv /tmp/eksctl /usr/local/bin

eksctl version
```

```
eksctl create cluster -f dso-cluster-creation.yaml
```

```
$ eksctl utils associate-iam-oidc-provider --cluster dev-secops-cluster --approve --region=us-west-2
2024-10-20 17:49:50 [ℹ]  will create IAM Open ID Connect provider for cluster "dev-secops-cluster" in "us-west-2"
2024-10-20 17:49:52 [✔]  created IAM Open ID Connect provider for cluster "dev-secops-cluster" in "us-west-2"

$ export cluster_name="dev-secops-cluster" 

$ aws eks describe-cluster --name dev-secops-cluster --query "cluster.identity.oidc.issuer" --output text -region us-west-2

cat <<EOF > trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::909293070315:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/726902B7EB55F793FE333D6D0D3A5C37"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-west-2.amazonaws.com/id/726902B7EB55F793FE333D6D0D3A5C37:aud": "sts.amazonaws.com",
          "oidc.eks.us-west-2.amazonaws.com/id/726902B7EB55F793FE333D6D0D3A5C37:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }
  ]
}
EOF

$ aws iam create-role  --role-name AmazonEKS_EBS_CSI_DriverRole  --assume-role-policy-document file://"trust-policy.json"

$ aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy --role-name AmazonEKS_EBS_CSI_DriverRole
```
RefL https://repost.aws/knowledge-center/eks-persistent-storage

```
k create namespace ci
namespace/ci created
```

Helm installation
```
$ curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

Downloading https://get.helm.sh/helm-v3.16.2-linux-amd64.tar.gz
Verifying checksum... Done.
Preparing to install helm into /usr/local/bin
helm installed into /usr/local/bin/helm

$ helm version
version.BuildInfo{Version:"v3.16.2", GitCommit:"13654a52f7c70a143b1dd51416d633e1071faffb", GitTreeState:"clean", GoVersion:"go1.22.7"}

$ helm repo add jenkins https://charts.jenkins.io
lm repo update
"jenkins" has been added to your repositories
chandika@QB-BLR-1596:~/Desktop/cdit/devsecops-ci$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jenkins" chart repository
Update Complete. ⎈Happy Helming!⎈

```

Jenkins Installation
```
$ helm install --namespace ci --values jenkins.values.yaml jenkins jenkins/jenkins
NAME: jenkins
LAST DEPLOYED: Sun Oct 20 19:15:27 2024
NAMESPACE: ci
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
  kubectl exec --namespace ci -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  export NODE_PORT=$(kubectl get --namespace ci -o jsonpath="{.spec.ports[0].nodePort}" services jenkins)
  export NODE_IP=$(kubectl get nodes --namespace ci -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT

3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http://$NODE_IP:$NODE_PORT/configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine

For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/


NOTE: Consider using a custom image with pre-installed plugins
```

```
k get pvc -n ci
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
jenkins   Bound    pvc-0a52149f-411e-4f9a-a1f8-ad2d0070fbbc   8Gi        RWO            gp2            11s

```
$ kubectl exec --namespace ci -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
r0QrC73ql4BqIdMQqUdxQM
```
Allow all in SG eksctl-dev-secops-cluster-nodegroup-ng-2-SG-6nzKi3VwyZve


Install plugins: 
  Blue Ocean 
  Configuration as Code Plugin - Groovy Scripting Extension

Modify admin password - It will prompt when restart jenkins after installing plugins




aws eks update-kubeconfig --name $cluster_name --region us-west-2

kubectl create secret -n ci docker-registry regcred --docker-server=https://hub.docker.com --docker-username=ranjiniganeshan@gmail.com --docker-password=Diehard12

stage('Docker BnP') {
 steps {
 container('kaniko') {
 sh '/kaniko/executor -f `pwd`/Dockerfile -c `pwd` --insecure --
skip-tls-verify --cache=true --destination=docker.io/ranjini/dso-demo'
 }
 }
 }