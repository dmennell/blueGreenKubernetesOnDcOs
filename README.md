# blueGreenKubernetesOnDcOs

Blue Green Deployment Steps


# Cluster URL:
standingby.ishmaelsolutions.com
honeybadger/dontcare

# Prep Cluster

## Install Enterprise CLI
`dcos package install dcos-enterprise-cli`
## Create Attributes

## Create PROMETHEUS Service Account
```
dcos security org service-accounts keypair private-key.pem public-key.pem
dcos security org service-accounts create -p public-key.pem -d 'prometheus service account' prometheus
dcos security secrets create-sa-secret private-key.pem prometheus prometheus/sa
** dcos security org groups add_user superusers prometheus
```

# Deploy MarathonLB
`dcos package install marathon-lb`

# Deploy Prometheus
```
dcos security org service-accounts keypair private-key.pem public-key.pem
dcos security org service-accounts create -p public-key.pem -d 'prometheus service account' prometheus
dcos security secrets create-sa-secret private-key.pem prometheus prometheus/sa
** dcos security org groups add_user superusers prometheus
```
# Deploy Jenkins
## Create JENKINS Service Account
```
dcos security org service-accounts keypair private-key.pem public-key.pem
dcos security org service-accounts create -p public-key.pem -d 'jenkins service account' jenkins
dcos security secrets create-sa-secret private-key.pem jenkins jenkins/sa
** dcos security org groups add_user superusers jenkins
```
# Deploy Gitlab
## Create GITLAB Service Account
```
dcos security org service-accounts keypair private-key.pem public-key.pem
dcos security org service-accounts create -p public-key.pem -d 'gitlab service account' gitlab
dcos security secrets create-sa-secret private-key.pem gitlab gitlab/sa
** dcos security org groups add_user superusers gitlab
```
# Deploy Docker Registry

# Deploy "kubernetes-blue"
## Create "kubernetes-blue Service Account & Permissions
```
dcos security org service-accounts keypair private-key.pem public-key.pem
dcos security org service-accounts create -p public-key.pem -d 'kubernetes-blue service account' kubernetes-blue
dcos security secrets create-sa-secret private-key.pem kubernetes-blue kubernetes-blue/sa

dcos:mesos:master:framework:role:kubernetes-blue-role create
dcos:mesos:master:task:user:root create
dcos:mesos:agent:task:user:root create
dcos:mesos:master:reservation:role:kubernetes-blue-role create
dcos:mesos:master:reservation:principal:kubernetes-blue delete
dcos:mesos:master:volume:role:kubernetes-blue-role create
dcos:mesos:master:volume:principal:kubernetes-blue delete

dcos:service:marathon:marathon:services:/ create
dcos:service:marathon:marathon:services:/ delete

dcos:secrets:default:/kubernetes-blue/* full
dcos:secrets:list:default:/kubernetes-blue read
dcos:adminrouter:ops:ca:rw full
dcos:adminrouter:ops:ca:ro full

dcos:mesos:master:framework:role:slave_public/kubernetes-blue-role create
dcos:mesos:master:framework:role:slave_public/kubernetes-blue-role read
dcos:mesos:master:reservation:role:slave_public/kubernetes-blue-role create
dcos:mesos:master:volume:role:slave_public/kubernetes-blue-role create
dcos:mesos:master:framework:role:slave_public read
dcos:mesos:agent:framework:role:slave_public read
```

## Build "kubectl-blue-proxy" Proxy File
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
## Deploy the Service
`dcos edgelb create kubectl-green-proxy.json`


# Deploy "kubernetes-green" 
## Create kubernetes-green Service Account & Permissions
```
dcos security org service-accounts keypair private-key.pem public-key.pem
dcos security org service-accounts create -p public-key.pem -d 'kubernetes-green service account' kubernetes-green
dcos security secrets create-sa-secret private-key.pem kubernetes-green kubernetes-green/sa

dcos:mesos:master:framework:role:kubernetes-green-role create
dcos:mesos:master:task:user:root create
dcos:mesos:agent:task:user:root create
dcos:mesos:master:reservation:role:kubernetes-green-role create
dcos:mesos:master:reservation:principal:kubernetes-green delete
dcos:mesos:master:volume:role:kubernetes-green-role create
dcos:mesos:master:volume:principal:kubernetes-green delete

dcos:service:marathon:marathon:services:/ create
dcos:service:marathon:marathon:services:/ delete

dcos:secrets:default:/kubernetes-green/* full
dcos:secrets:list:default:/kubernetes-green read
dcos:adminrouter:ops:ca:rw full
dcos:adminrouter:ops:ca:ro full

dcos:mesos:master:framework:role:slave_public/kubernetes-green-role create
dcos:mesos:master:framework:role:slave_public/kubernetes-green-role read
dcos:mesos:master:reservation:role:slave_public/kubernetes-green-role create
dcos:mesos:master:volume:role:slave_public/kubernetes-green-role create
dcos:mesos:master:framework:role:slave_public read
dcos:mesos:agent:framework:role:slave_public read

```
## Build "kubectl-green-proxy" Proxy File
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
## Deploy the Service
`dcos edgelb create kubectl-green-proxy.json`

# Create Kubernetes Contexts
## Create kubernetes-blue context
dcos kubernetes-blue kubeconfig \
    --apiserver-url https://Path:4443 \
    --no-activate-context /
    --insecure-skip-tls-verify
## Create kubernetes-green context
dcos kubernetes-blue kubeconfig \
    --apiserver-url https://Path:5443 \
    --no-activate-context \
    --insecure-skip-tls-verify
## Switch Contexts
kubectl config use-context kubernetes-blue
kubectl config use-context kubernetes-green

