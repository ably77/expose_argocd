# install istioctl
```
export ISTIO_VERSION=1.11.4
curl -L https://istio.io/downloadIstio | sh -
```

# install istio
cd istio-1.11.4
kubectl create ns istio-system
./bin/istioctl install --set profile=default

# check
```
% k get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-7cdf96b4d9-kq8qt   1/1     Running   0          6m26s
istiod-64c58fdf-4jjgg                   1/1     Running   0          6m37s
```

```
% k get svc -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.116.38.253   34.136.241.164   15021:32350/TCP,80:31930/TCP,443:32590/TCP   44s
istiod                 ClusterIP      10.116.47.58    <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP        56s
```

# create argocd namespace and label for istio injection
```
kubectl create ns argocd
kubectl label namespace argocd istio-injection=enabled
```

# deploy argocd
```
kubectl apply -f 1-argo-insecure.yaml -n argocd
```

# watch argocd namespace
```
kubectl get pods -n argocd -w
```

Should see 2/2
```
% kubectl get pods -n argocd
NAME                                  READY   STATUS    RESTARTS   AGE
argocd-application-controller-0       2/2     Running   0          41s
argocd-dex-server-7946bfbf79-xxjjn    2/2     Running   1          43s
argocd-redis-7547547c4f-sxjxz         2/2     Running   0          43s
argocd-repo-server-6b5cf77fbc-spqfq   2/2     Running   0          42s
argocd-server-56ccf4965-m2tfp         2/2     Running   0          42s
```

# change password
```
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2a$10$79yaoOg9dL5MO8pn8hGqtO4xQDejSEVNWAGQR268JHLdrCw6UCYmy",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
```

# create secret in istio-system
In my case i'm using a cloudflare cert for my domain *.kapoozi.com
```
kubectl create secret tls cf-upstream-tls --key kapoozi-tls.key \
   --cert kapoozi-tls.crt --namespace istio-system
```

# configure DNS for your hosts
In my case I have configured these records in Cloudflare pointing to my `istio-ingressgateway` LoadBalancer IP, and my cert is a wildcard cert for `*.kapoozi.com`
```
- 'argocd.kapoozi.com'
- 'argogrpc.kapoozi.com'
```

# deploy argocd gateway
```
kubectl apply -f 2-argo-gw.yaml
```

# access argocd ui
You should now be able to access argocd UI at https://argocd.kapoozi.com/argo

# login to argocd
```
argocd login argocd.kapoozi.com:443
```

# output
```
% argocd login argocd.kapoozi.com:443
WARN[0010] Failed to invoke grpc call. Use flag --grpc-web in grpc calls. To avoid this warning message, use flag --grpc-web. 
Username: admin
Password: 
'admin:login' logged in successfully
Context 'argocd.kapoozi.com:443' updated
```

# logout
``` 
argocd logout argogrpc.kapoozi.com:443
```