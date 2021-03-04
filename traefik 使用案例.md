kubernetes | 1.16.6
traefik 2.2

证书创建：kubectl create secret generic secret-tls --from-file=tls.crt --from-file=tls.key -n kube-system

1.traefik-ui
---
http
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
        - name: redirect-https   #http -> https 需要提前创建
          namespace: kube-system   
---
https
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-ui.htexam.com-tls
  namespace: kube-system 
spec:
  entryPoints:
    - websecure
  tls:
    secretName: htexam-tls 
    certResolver: default
  routes:
    - match: Host(`traefik-ui.htexam.com`)
      kind: Rule
      services:
        - name: traefik
          port: 8080
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
          
3.kinaba-ui
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: kibana-ui
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`kibana.xxx.com`)
      kind: Rule
      services:
        - name: kibana-logging  
          port: 5601
      middlewares:
        - name: kibana-replacepathregex #组件提前生成
          namespace: kube-system
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: kibana-replacepathregex   #跳转组件
spec:
  replacePathRegex:
    regex: ^/api/v1/namespaces/kube-system/services/kibana-logging/proxy/(.*)
    replacement: /$1
