# Deploy nearly identical Blue/Green Kubernetes Environments on Mesosphere Enterprise DC/OS

## Deploy "kubernetes-blue" Services
### Create "kubernetes-blue Service Account & Permissions
```
dcos security org service-accounts keypair private-key.pem public-key.pem
dcos security org service-accounts create -p public-key.pem -d 'kubernetes-blue service account' kubernetes-blue
dcos security secrets create-sa-secret private-key.pem kubernetes-blue kubernetes-blue/sa
sleep 5
dcos:mesos:master:framework:role:kubernetes-blue-role create
dcos:mesos:master:task:user:root create
dcos:mesos:agent:task:user:root create
dcos:mesos:master:reservation:role:kubernetes-blue-role create
dcos:mesos:master:reservation:principal:kubernetes-blue delete
dcos:mesos:master:volume:role:kubernetes-blue-role create
dcos:mesos:master:volume:principal:kubernetes-blue delete
sleep 5
dcos:service:marathon:marathon:services:/ create
dcos:service:marathon:marathon:services:/ delete
sleep 5
dcos:secrets:default:/kubernetes-blue/* full
dcos:secrets:list:default:/kubernetes-blue read
dcos:adminrouter:ops:ca:rw full
dcos:adminrouter:ops:ca:ro full
sleep 5
dcos:mesos:master:framework:role:slave_public/kubernetes-blue-role create
dcos:mesos:master:framework:role:slave_public/kubernetes-blue-role read
dcos:mesos:master:reservation:role:slave_public/kubernetes-blue-role create
dcos:mesos:master:volume:role:slave_public/kubernetes-blue-role create
dcos:mesos:master:framework:role:slave_public read
dcos:mesos:agent:framework:role:slave_public read
```
### Build "kubernetes-blue.json" Deployment File
```
cat > kubernete-blue.json << 'EOF'
{
  "service": {
    "name": "kubernetes-blue",
    "sleep": 1000,
    "service_account": "kubernetes-blue",
    "service_account_secret": "kubernetes-blue/sa",
    "log_level": "INFO"
  },
  "kubernetes": {
    "authorization_mode": "RBAC",
    "high_availability": true,
    "enable_insecure_port": true,
    "control_plane_placement": "[[\"rack\", \"like\", \"one\"]]",
    "service_cidr": "10.100.0.0/16",
    "network_provider": "dcos",
    "cloud_provider": "(none)",
    "node_count": 3,
    "node_placement": "[[\"rack\", \"like\", \"one\"]]",
    "public_node_count": 1,
    "public_node_placement": "[[\"rack\", \"like\", \"one\"]]",
    "reserved_resources": {
      "kube_cpus": 4,
      "kube_mem": 8192,
      "kube_disk": 10240,
      "system_cpus": 1,
      "system_mem": 1024
    },
    "public_reserved_resources": {
      "kube_cpus": 2,
      "kube_mem": 4096,
      "kube_disk": 10240,
      "system_cpus": 1,
      "system_mem": 1024
    }
  },
  "etcd": {
    "cpus": 0.5,
    "mem": 1024,
    "data_disk": 3072,
    "wal_disk": 512,
    "disk_type": "ROOT"
  },
  "scheduler": {
    "cpus": 0.5,
    "mem": 512
  },
  "controller_manager": {
    "cpus": 0.5,
    "mem": 512
  },
  "apiserver": {
    "cpus": 0.5,
    "mem": 1024
  },
  "kube_proxy": {
    "cpus": 0.1,
    "mem": 512
  }
}
EOF
```
### Deploy the "kubernetes-blue.json" Deployment File
`dcos package install kubernetes --options=kubernetes-blue.json`
### Build "kubectl-blue-proxy.json" Deployment File
This exposes the kubernetes-blue API service to the outside world for use by kubectl or CI/CD tooling
```
cat > kubectl-blue-proxy.json << 'EOF'
{
  "id": "/kubectl-proxy-blue",
  "instances": 1,
  "cpus": 0.001,
  "mem": 16,
  "cmd": "tail -F /dev/null",
  "container": {
    "type": "MESOS"
  },
  "portDefinitions": [
    {
      "protocol": "tcp",
      "port": 0
    }
  ],
  "labels": {
    "HAPROXY_GROUP": "external",
    "HAPROXY_0_MODE": "http",
    "HAPROXY_0_PORT": "4443",
    "HAPROXY_0_SSL_CERT": "/etc/ssl/cert.pem",
    "HAPROXY_0_BACKEND_SERVER_OPTIONS": "  timeout connect 10s\n  timeout client 86400s\n  timeout server 86400s\n  timeout tunnel 86400s\n  server kube-apiserver apiserver.kubernetes-blue.l4lb.thisdcos.directory:6443 ssl verify none\n"
  }
}
EOF
```
### Deploy the "kubectl-blue-proxy.json" Deployment File
`dcos marathon app add kubectl-blue-proxy.json`
### Create "kubernetes-blue" Kubectl Context
```
dcos kubernetes-blue kubeconfig \
    --apiserver-url https://<Path-To-Marathon-LB-Proxy-Server>:4443 \
    --no-activate-context \
    --insecure-skip-tls-verify
```
### Switch to "kubernetes-blue" Context
`kubectl config use-context kubernetes-green`
### Verify "kubernetes-blue" Kubectl Context Works
`kubectl get nodes`








## Deploy "kubernetes-green" Services
### Create "kubernetes-green" Service Account & Permissions
```
dcos security org service-accounts keypair private-key.pem public-key.pem
dcos security org service-accounts create -p public-key.pem -d 'kubernetes-green service account' kubernetes-green
dcos security secrets create-sa-secret private-key.pem kubernetes-green kubernetes-green/sa
sleep 5
dcos:mesos:master:framework:role:kubernetes-green-role create
dcos:mesos:master:task:user:root create
dcos:mesos:agent:task:user:root create
dcos:mesos:master:reservation:role:kubernetes-green-role create
dcos:mesos:master:reservation:principal:kubernetes-green delete
dcos:mesos:master:volume:role:kubernetes-green-role create
dcos:mesos:master:volume:principal:kubernetes-green delete
sleep 5
dcos:service:marathon:marathon:services:/ create
dcos:service:marathon:marathon:services:/ delete
sleep 5
dcos:secrets:default:/kubernetes-green/* full
dcos:secrets:list:default:/kubernetes-green read
dcos:adminrouter:ops:ca:rw full
dcos:adminrouter:ops:ca:ro full
sleep 5
dcos:mesos:master:framework:role:slave_public/kubernetes-green-role create
dcos:mesos:master:framework:role:slave_public/kubernetes-green-role read
dcos:mesos:master:reservation:role:slave_public/kubernetes-green-role create
dcos:mesos:master:volume:role:slave_public/kubernetes-green-role create
dcos:mesos:master:framework:role:slave_public read
dcos:mesos:agent:framework:role:slave_public read
```
### Create "kubernetes-green.json" Deployment File
```
cat > kubernetes-green.json << 'EOF'
{
  "service": {
    "name": "kubernetes-green",
    "sleep": 1000,
    "service_account": "kubernetes-green",
    "service_account_secret": "kubernetes-green/sa",
    "log_level": "INFO"
  },
  "kubernetes": {
    "authorization_mode": "RBAC",
    "high_availability": true,
    "enable_insecure_port": true,
    "control_plane_placement": "[[\"rack\", \"like\", \"two\"]]",
    "service_cidr": "10.101.0.0/16",
    "network_provider": "dcos",
    "cloud_provider": "(none)",
    "node_count": 3,
    "node_placement": "[[\"rack\", \"like\", \"two\"]]",
    "public_node_count": 1,
    "public_node_placement": "[[\"rack\", \"like\", \"two\"]]",
    "reserved_resources": {
      "kube_cpus": 4,
      "kube_mem": 8192,
      "kube_disk": 10240,
      "system_cpus": 1,
      "system_mem": 1024
    },
    "public_reserved_resources": {
      "kube_cpus": 2,
      "kube_mem": 4096,
      "kube_disk": 10240,
      "system_cpus": 1,
      "system_mem": 1024
    }
  },
  "etcd": {
    "cpus": 0.5,
    "mem": 1024,
    "data_disk": 3072,
    "wal_disk": 512,
    "disk_type": "ROOT"
  },
  "scheduler": {
    "cpus": 0.5,
    "mem": 512
  },
  "controller_manager": {
    "cpus": 0.5,
    "mem": 512
  },
  "apiserver": {
    "cpus": 0.5,
    "mem": 1024
  },
  "kube_proxy": {
    "cpus": 0.1,
    "mem": 512
  }
}
EOF
```
### "Deploy kubernetes-green.json Deployment File"
`dcos package install kubernetes --options=kubernetes-green.json`
### Build "kubectl-green-proxy.json" Deployment File
this exposes the kubernetes-green API service to the outside world for use by kubectl or CI/CD tooling
```
cat > kubectl-green-proxy.json << 'EOF'
{
  "id": "/kubectl-proxy-green",
  "instances": 1,
  "cpus": 0.001,
  "mem": 16,
  "cmd": "tail -F /dev/null",
  "container": {
    "type": "MESOS"
  },
  "portDefinitions": [
    {
      "protocol": "tcp",
      "port": 0
    }
  ],
  "labels": {
    "HAPROXY_GROUP": "external",
    "HAPROXY_0_MODE": "http",
    "HAPROXY_0_PORT": "5443",
    "HAPROXY_0_SSL_CERT": "/etc/ssl/cert.pem",
    "HAPROXY_0_BACKEND_SERVER_OPTIONS": "  timeout connect 10s\n  timeout client 86400s\n  timeout server 86400s\n  timeout tunnel 86400s\n  server kube-apiserver apiserver.kubernetes-green.l4lb.thisdcos.directory:6443 ssl verify none\n"
  }
}
EOF
```
### Deploy the "kubectl-green-proxy.json" Deployment File
`dcos edgelb create kubectl-green-proxy.json`
### Create "kubernetes-green" kubectl context
```
dcos kubernetes-green kubeconfig \
    --apiserver-url https://<Path-To-Marathon-LB-Proxy-Server>:5443 \
    --no-activate-context \
    --insecure-skip-tls-verify
```
### Switch Contexts to "kubernetes-green"
`kubectl config use-context kubernetes-green`
### Verify kubernetes-green Kubectl Context Works
`kubectl get nodes`
