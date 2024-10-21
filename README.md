# Secure Pipelines Demo

Sample spring application with Jenkins pipeline script to demonstrate secure pipelines


![DevSecOps Pipeline](https://wac-cdn.atlassian.com/dam/jcr:5f26d67b-bed6-4be1-912b-4032de4d06b0/devsecops-diagram.png?cdnVersion=2318)


## Pre Requesites

1. K8s cluster 
2. eksctl, kubectl, helm cli

## Setup Setps

eksctl cli installation for Ubuntu
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

Create EKS cluster

```
eksctl create cluster -f setup/eks-cluster-setup.yaml
```

Let the EKS cluster creation in progress

Talisman Installation
====================
https://thoughtworks.github.io/talisman/docs

# Download the talisman binary
curl https://thoughtworks.github.io/talisman/install.sh > ~/install-talisman.sh
chmod +x ~/install-talisman.sh

# Install to a single project (as pre-push hook)
cd my-git-project
~/install-talisman.sh //pre-push hook
~/install-talisman.sh pre-commit //pre-commit hook

mkdir sec_files && cd sec_files
echo "username:sidd-harth" > file1
echo "secure-password1234" > password.txt
echo "apiKey=Ajbehjruhr82832jcshh3435" > file2
echo "base64encodedsecret=cDSgwjekrnekrjitut" > file3

git commit 
git push 

If you want to ignore any file, add it under .talismanrc
fileignoreconfig:
- filename:
  checksum:


OIDC is the replacement for using IAM roles or keys to create/modify/update/delete AWS service
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


Create CI/CD namespace
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

$ helm repo update
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
$ kubectl exec --namespace ci -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
r0QrC73ql4BqIdMQqUdxQM
```

Jenkins configuration:
=====================

- Add additonal plugins to Jeninks server (Manage Jenkins -> Manage plugins)

  - BlueOcean
  - Configuration as Code
  - OWASP Dependency-Track

### New Jenkins Pipeline

Create a new Jenkins pipeline with this repo and trigger build

- Login to Jenkins -> New Item -> Enter name and choose Pipeline -> Choose GitHub project and set project URL
- Under pipeline section, Choose Pipeline script from SCM
- Choose git as SCM and provide repo details
- Save


Allow all in SG eksctl-dev-secops-cluster-nodegroup-ng-2-SG-6nzKi3VwyZve

Access Jenkins 

Modify admin password - It will prompt when restart jenkins after installing plugins


kubectl create secret -n ci docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=chandikas --docker-password=xxxxxx --docker-email=erchandika@gmail.com


Configure github repo with jenkins by creating a new pipeline item

Run the build

Configure the pipeline to poll every minute

Add Kaniko stage in Jenkinsfile

```
        stage('OCI image build') {
          steps {
            container('kaniko') {
              sh '/kaniko/executor -f "$(pwd)/Dockerfile" -c "$(pwd)" --insecure --skip-tls-verify --cache=true --destination=docker.io/chandikas/dso-demo --verbosity=debug' 
              }
            }
          }
```

Cleanup
=======
eksctl delete cluster --name=dev-secops-cluster --region=us-west-2



# Pipeline

Refer the below screenshot for the stages in the pipeline

##### Pipeline View

![Pipeline View](imgs/Secure_Pipeline_1.png)

##### Stage View

![Stage View](imgs/Secure_Pipeline_2.png)

##### Dependency Track

![Dependency Track View](imgs/Dependency_Track.png)

## Tools

| Stage               | Tool                                                                      |
| ------------------- | ------------------------------------------------------------------------- |
| Secrets Scanner     | [truffleHog](https://github.com/dxa4481/truffleHog)                       |
| Dependency Checker  | [OWASP Dependency checker](https://jeremylong.github.io/DependencyCheck/) |
| SAST                | [OWASP Find Security Bugs](https://find-sec-bugs.github.io/)              |
| OSS License Checker | [LicenseFinder](https://github.com/pivotal/LicenseFinder)                 |
| SCA                 | [Dependency Track](https://dependencytrack.org/)                          |
| Image Scanner       | [Trivy](https://github.com/aquasecurity/trivy)                            |
| Image Hardening     | [Dockle](https://github.com/goodwithtech/dockle)                          |
| K8s Hardening       | [KubeSec](https://kubesec.io/)                                            |
| IaC Hardening       | [checkov](https://www.checkov.io/)                                        |
| DAST                | [OWASP Baseline Scan](https://www.zaproxy.org/docs/docker/baseline-scan/) |

---

### TODO

Image Malware scanning - [ClamAV](https://github.com/openbridge/clamav)

