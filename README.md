# Helm repo add
```
helm repo add registry-mirror https://shanaider.github.io/docker-registry-mirror-chart/
```
# Helm install
```
helm upgrade --install my-docker-registry-mirror registry-mirror/docker-registry-mirror 
```
# Helm values
1. inside the container need to connect the https://registry-1.docker.io if your environment can't directly to internet please consider add your env of proxy server
```
extraVars:
- name: HTTPS_PROXY
  value: http://{proxy_ip}:{proxy_port}
- name: https_proxy
  value: http://{proxy_ip}:{proxy_port}
```
2. Enable ingress put your ingress url (this is for sample with ingress-nginx and don't use tls)
```
ingress:
  enabled: true
  className: nginx
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: docker-registry-mirror.example.com
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local
```
3. Enable PVC (this is for sample with longhorn storage class)
```
persistence:
  accessMode: 'ReadWriteOnce'
  enabled: true
  size: 1Gi
  storageClass: longhorn
```
4. Keep up to date the image of registry (please see in https://hub.docker.com/_/registry)
```
image:
  repository: registry
  tag: 2.8.3
  pullPolicy: IfNotPresent
  # imagePullSecrets:
    # - name: docker
```
5. Configure the cache

To configure a Registry to run as a pull through cache, the addition of a proxy section is required to the config file.
Ref. https://docs.docker.com/docker-hub/mirror/#run-a-registry-as-a-pull-through-cache
```
configData:
  version: 0.1
  log:
    fields:
      service: registry
  storage:
    cache:
      blobdescriptor: inmemory
    filesystem:
      rootdirectory: /var/lib/registry
  http:
    addr: :5000
    headers:
      X-Content-Type-Options: [nosniff]
  health:
    storagedriver:
      enabled: true
      interval: 10s
      threshold: 3

proxy:
  remoteurl: https://registry-1.docker.io
  username:
  password:
```

# Config the container service on all nodes
**This is a simple way. Do not in the Production for skip user/pw and skip certificate.
## for containerd

edit /etc/containerd/config.toml
```
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["https://docker-registry-mirror.example.com"]
[plugins."io.containerd.grpc.v1.cri".registry.configs."docker-registry-mirror.example.com".tls]
  insecure_skip_verify = true
```

```
sudo systemctl restart containerd
```

## for crio
edit /etc/containers/registries.conf

```
unqualified-search-registries = ["docker.io"]
​
[[registry]]
prefix = "docker.io"
insecure = false
blocked = false
location = "docker.io"
​
[[registry.mirror]]
location = "docker-registry-mirror.example.com"
insecure = true
```

```
sudo systemctl restart crio
```
## for kaniko builder
```
REGISTRY_MIRROR=docker-registry-mirror.example.com
executor \ 
  --registry-mirror $REGISTRY_MIRROR \
  --insecure-pull \
...
```
# Test deployment some application

```kubectl create deployment http-echo --image hashicorp/http-echo
# Watch the log on docker-registry-mirror
k logs docker-registry-mirror-fd5796786-wxlzh -f

time="2023-12-20T06:34:54.82182464Z" level=info msg="response completed" go.version=go1.20.8 http.request.host=docker-registry-mirror.mora-mora.tech http.request.id=65ab5506-0227-4b81-8b74-7f0bbfc6ed16 http.request.method=HEAD http.request.remoteaddr=159.138.229.95 http.request.uri="/v2/hashicorp/http-echo/manifests/latest" http.request.useragent="containerd/v1.6.24" http.response.contenttype="application/vnd.docker.distribution.manifest.list.v2+json" http.response.duration=2.429836681s http.response.status=200 http.response.written=743
```
# Test pull that image directly to docker register mirror
```
crictl pull docker-registry-mirror.example.com/hashicorp/http-echo
Image is up to date for sha256:04fa556e62bddac2a783622a3cccc35d6d5c599199eecf75b6b4f2bc76c8c825
```
# Inside the container look like this
```
$ k exec -ti docker-registry-mirror-fd5796786-wxlzh -- sh
$ cd /var/lib/registry/docker/registry/v2/repositories
$ /var/lib/registry/docker/registry/v2/repositories $ tree
.
├── hashicorp
│   └── http-echo
│       └── _manifests
│           ├── revisions
│           │   └── sha256
│           │       └── fcb75f691c8b0414d670ae570240cbf95502cc18a9ba57e982ecac589760a186
│           │           └── link
│           └── tags
│               └── latest
│                   ├── current
│                   │   └── link
│                   └── index
│                       └── sha256
│                           └── fcb75f691c8b0414d670ae570240cbf95502cc18a9ba57e982ecac589760a186
│                               └── link
```

ref.
https://github.com/t83714/docker-registry-mirror