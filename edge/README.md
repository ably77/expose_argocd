# install Gloo Edge Enterprise 1.9.1
```
glooctl upgrade --release=v1.9.1
helm repo add glooe https://storage.googleapis.com/gloo-ee-helm
helm upgrade --install gloo glooe/gloo-ee --namespace gloo-system --create-namespace --version 1.9.1 --set license_key=$LICENSE_KEY
```

# watch gloo-system
```
kubectl get pods -n gloo-system -w
```

# install argocd
```
kubectl create ns argocd
kubectl apply -f 1-argo-insecure.yaml -n argocd
```

# change password
```
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2a$10$79yaoOg9dL5MO8pn8hGqtO4xQDejSEVNWAGQR268JHLdrCw6UCYmy",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
```

# create secret in gloo-system
In my case i'm using a cloudflare cert for my domain *.kapoozi.com
```
kubectl create secret tls cf-upstream-tls --key kapoozi-tls.key \
   --cert kapoozi-tls.crt --namespace gloo-system
```

# configure DNS for your hosts
In my case I have configured these records in Cloudflare and my cert is a wildcard cert for `*.kapoozi.com`
```
- 'argocd.kapoozi.com'
- 'argogrpc.kapoozi.com'
```

# expose argocd ui endpoint on port 80/443
```
kubectl apply -f 2a-argo-ui-http.yaml
kubectl apply -f 2b-argo-ui-https.yaml
```

You should now be able to navigate to argocd ui http(s)://argocd.kapoozi.com/argo

# test login using argocd cli
```
argocd login argocd.kapoozi.com:443
```

This will fail, because the upstream is not using GRPC
```
% argocd login argocd.kapoozi.com:443
Username: admin
Password: 
FATA[0009] rpc error: code = PermissionDenied desc = Forbidden 
```

# create a static upstream for grpc
The key here is leveraging `useHttp2: true` in the static upstream to require GRPC connection
```
kubectl apply -f 3-grpc-static-upstream.yaml
```

# expose argocd cli grpc endpoint with a virtualservice
```
kubectl apply -f 4-argo-grpc-vs.yaml
```

# login to argocd
```
argocd login argogrpc.kapoozi.com:443
```

# output
```
% argocd login argogrpc.kapoozi.com:443
WARN[0000] Failed to invoke grpc call. Use flag --grpc-web in grpc calls. To avoid this warning message, use flag --grpc-web. 
Username: admin
Password: 
'admin:login' logged in successfully
Context 'argogrpc.kapoozi.com:443' updated
```

# logout
``` 
argocd logout argogrpc.kapoozi.com:443
```