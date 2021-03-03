kubernetes | 1.16.6
traefik 2.2

1.traefik-ui
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik.xxx.com 
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`traefik.xxx.com`)
      kind: Rule
      services:
        - name: traefik
          port: 8080
      middlewares:
        - name: redirect-https
          namespace: kube-system

2.dashboard-ui
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: dashboard-ui
  namespace: kubernetes-dashboard
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`dashboard.xxxx.com`) 
      kind: Rule
      services:
        - name: kubernetes-dashboard
          port: 443
