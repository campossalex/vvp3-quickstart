## Infrastructure
Spin up a m5.4xlarge ec2 instance. This instance has 16 cores and 64 Gb memory. If you need more resources to run Flink jobs, use a m5.8xlarge (32 cores / 128 Gb memory).  
Any Amazong Linux 2023 AMI should be fine to run this quickstart.  

## Minikube, Minio, Mysql and VVP 3.1.0 installation process  

Connect to the instance's command line (ssh) to run the following commands as `sudo su`:  

```
# Additional packages
sudo yum install -y jq
sudo yum install -y nginx

# Install kubctl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /bin/kubectl

# Install minikube
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /bin/minikube

# Install helm chart
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
sudo HELM_INSTALL_DIR=/bin ./get_helm.sh

# Install Docker
sudo yum update -y
sudo yum install -y docker
sudo service docker start

# start minikube
sudo minikube start --memory=40G --cpus=12 --force

# start nginx
systemctl start nginx

# Create ns:
kubectl create ns vvp-system
kubectl create ns vvp-deploy

# Set registry secrets:
kubectl -n vvp-system create secret docker-registry ververica-registry --docker-username=<username> --docker-password=<password> --docker-server=registry.ververica.cloud
kubectl -n vvp-deploy create secret docker-registry ververica-registry --docker-username=<username> --docker-password=<password> --docker-server=registry.ververica.cloud

# Mysql password secrete
kubectl -n vvp-system create secret generic mysql-secret   --from-literal=mysql-root-password='admin123'

# Deploy mysql
kubectl apply -f values-mysql.yaml

# Deploy minio (s3)
helm --namespace "vvp-system" upgrade --install "minio" "minio" --repo https://charts.helm.sh/stable --values values-minio.yaml

# Deploy the VVP 3 helm
helm upgrade --install ververica-platform oci://registry.ververica.cloud/platform-charts/ververica-platform --version 3.1.0 --namespace vvp-system --values values-vvp.yaml

# Wait for the pods to start to get token
kubectl logs vvp-appmanager-0 -n vvp-system

# Ask for a license to Ververica sendinf the TOKEN obtained before.  
One the license is received, create a license-vvp.yaml with the license content and use the REPLACE placeholders with the license values.

One you get the license file, add the value to the values-vvp-with-license.yaml. Ensure the indentation is correct and at the right level (whitspaces).  

# Upgrade the VVP 3 helm with the values-vvp-with-license.yaml file
helm upgrade --install ververica-platform oci://registry.ververica.cloud/platform-charts/ververica-platform --version 3.1.0 --namespace vvp-system --values values-vvp-with-license.yaml -f license-vvp.yaml
```

## Expose VVP 3 Web Console   

### Enable minikube ingress
```
minikube addons enable ingress
```

### Edit ingress.yaml
Replace "EC2_INSTANCE_DNS" for the actual ec2 instance dns (you can get this in the AWS Console, in the EC2 section, selecting the virtual machine you are using to deploy VVP 3.  

### Apply ingress  
```
kubectl apply -f ingress.yaml
```

### Configure nginx

```
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

PUBLIC_DNS=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-hostname)

rm -f /etc/nginx/conf.d/default.conf
ls /etc/nginx/sites-enabled/ 2>/dev/null && rm -f /etc/nginx/sites-enabled/default

cat > /etc/nginx/conf.d/minikube-proxy.conf << EOF
server {
    listen 80;
    server_name ${PUBLIC_DNS};
    location / {
        proxy_pass http://192.168.49.2:80;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        # Strip Secure flag so cookies work over HTTP
        proxy_cookie_flags ~ nosecure;
    }
}
EOF

sed -i '/server_names_hash_bucket_size/d' /etc/nginx/nginx.conf
sed -i 's/http {/http {\n    server_names_hash_bucket_size 128;/' /etc/nginx/nginx.conf

nginx -t && systemctl reload nginx

```

### Start a port-forward  
```
kubectl port-forward svc/api-gateway 8080:8080 -n vvp-system --address 0.0.0.0
```


