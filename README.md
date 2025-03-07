

# Cloud-Native Web Voting Application with Kubernetes

This cloud-native web application is built using a mix of technologies. It's designed to be accessible to users via the internet, allowing them to vote for their preferred programming language out of six choices: C#, Python, JavaScript, Go, Java, and NodeJS.

## Technical Stack

- **Frontend**: The frontend of this application is built using React and JavaScript. It provides a responsive and user-friendly interface for casting votes.

- **Backend and API**: The backend of this application is powered by Go (Golang). It serves as the API handling user voting requests. MongoDB is used as the database backend, configured with a replica set for data redundancy and high availability.

1. Create EKS Cluster and Node Group
```
eksctl create cluster --name EKS_CLUSTER_NAME --region us-west-2 --nodegroup-name workers --node-type t2.medium --nodes 2
```
2. Configure IAM Role for EC2 (Optional)
```
{
	"Version": "2012-10-17",
	"Statement": [{
		"Effect": "Allow",
		"Action": [
			"eks:DescribeCluster",
			"eks:ListClusters",
			"eks:DescribeNodegroup",
			"eks:ListNodegroups",
			"eks:ListUpdates",
			"eks:AccessKubernetesApi"
		],
		"Resource": "*"
	}]
}
```
3. Install AWS CLI & Kubectl

## AWS CLI Installation
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

## Kubectl Installation
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.11/2023-03-17/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/
4. Configure Kubeconfig
```
aws eks update-kubeconfig --name EKS_CLUSTER_NAME --region us-west-2
kubectl get nodes
```
## MongoDB Setup

5. Create Namespace
```
kubectl create ns cloudchamp
kubectl config set-context --current --namespace cloudchamp
```
6. Deploy MongoDB
```
kubectl apply -f mongo-statefulset.yaml
kubectl apply -f mongo-service.yaml
kubectl apply -f mongo-secret.yaml
```

7. Initialize MongoDB Replica Set
```
cat << EOF | kubectl exec -it mongo-0 -- mongo
rs.initiate();
rs.add("mongo-1.mongo:27017");
rs.add("mongo-2.mongo:27017");
cfg = rs.conf();
cfg.members[0].host = "mongo-0.mongo:27017";
rs.reconfig(cfg, {force: true});
EOF
```
8. Insert Sample Data
```
cat << EOF | kubectl exec -it mongo-0 -- mongo
use langdb;
db.languages.insert({"name" : "python", "codedetail" : {"usecase" : "system, web, server-side", "rank" : 3, "homepage" : "https://www.python.org/", "votes" : 0}});
db.languages.insert({"name" : "javascript", "codedetail" : {"usecase" : "web, client-side", "rank" : 7, "homepage" : "https://en.wikipedia.org/wiki/JavaScript", "votes" : 0}});
EOF
```
## API Deployment

9. Deploy API
```
kubectl apply -f api-deployment.yaml
```
10. Expose API as LoadBalancer
```
kubectl expose deploy api --name=api --type=LoadBalancer --port=80 --target-port=8080
```
11. Get API Endpoint
```
API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
curl -s $API_ELB_PUBLIC_FQDN/languages | jq .
```

## Frontend Deployment

12. Deploy Frontend
```
kubectl apply -f frontend-deployment.yaml
```
13. Expose Frontend as LoadBalancer
```
kubectl expose deploy frontend --name=frontend --type=LoadBalancer --port=80 
--target-port=8080
```
14. Get Frontend URL
```
FRONTEND_ELB_PUBLIC_FQDN=$(kubectl get svc frontend -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
echo "Frontend URL: http://$FRONTEND_ELB_PUBLIC_FQDN"
```
## Final Test

15. Check MongoDB Data
```
kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
```