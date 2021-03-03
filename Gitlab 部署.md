kubernetes | 1.16.6

安装和配置分为俩部分Gitlab，Drone

第一步Gitlab安装配置，服务需要配置redis,postgresql,gitlab

1. redis安装
## PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis
  labels:
    app: redis
spec:
  capacity:          
    storage: 5Gi
  accessModes:       
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
  - hard
  - nfsvers=4.1
  nfs:               
    server: x.x.x.x
    path: /nfs/gitlab/redis      
---
## PVC
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: redis
spec:
  resources:
    requests:
      storage: 5Gi      
  accessModes:
  - ReadWriteOnce
  selector:
    matchLabels:
      app: redis
---
## Service
kind: Service
apiVersion: v1
metadata:
  name: gitlab-redis
  labels:
    name: gitlab-redis
spec:
  type: ClusterIP
  ports:
    - name: redis
      protocol: TCP
      port: 6379
      targetPort: redis
  selector:
    name: gitlab-redis
---
## Deployment
kind: Deployment
apiVersion: apps/v1
metadata:
  name: gitlab-redis
  labels:
    name: gitlab-redis
spec:
  replicas: 1
  selector:
    matchLabels:
      name: gitlab-redis
  template:
    metadata:
      name: gitlab-redis
      labels:
        name: gitlab-redis
    spec:
      containers:
      - name: gitlab-redis
        image: 'sameersbn/redis:4.0.9-3'  #可以选择最新版本
        ports:
        - name: redis
          containerPort: 6379
          protocol: TCP
        resources:
          limits:
            cpu: 1000m
            memory: 2Gi
          requests:
            cpu: 1000m
            memory: 2Gi
        volumeMounts:
          - name: data
            mountPath: /var/lib/redis
        livenessProbe:
          exec:
            command:
              - redis-cli
              - ping
          initialDelaySeconds: 5
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          exec:
            command:
              - redis-cli
              - ping
          initialDelaySeconds: 5
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
      volumes:
      - name: data     
        persistentVolumeClaim:
          claimName: redis   #pvc 挂载

2. postgresql安装配置
## PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgresql
  labels:
    app: postgresql
spec:
  capacity:          
    storage: 20Gi
  accessModes:       
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
  - hard
  - nfsvers=4.1    
  nfs:
    server: x.x.x.x
    path: /nfs/gitlab/postgresql
---
## PVC
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgresql
spec:
  resources:
    requests:
      storage: 20Gi 
  accessModes:
  - ReadWriteOnce
  selector:
    matchLabels:
      app: postgresql
---
## Service
kind: Service
apiVersion: v1
metadata:
  name: gitlab-postgresql
  labels:
    name: gitlab-postgresql
spec:
  ports:
    - name: postgres
      protocol: TCP
      port: 5432
      targetPort: postgres
  selector:
    name: postgresql
  type: ClusterIP
---
## Deployment
kind: Deployment
apiVersion: apps/v1
metadata:
  name: postgresql
  labels:
    name: postgresql
spec:
  replicas: 1
  selector:
    matchLabels:
      name: postgresql
  template:
    metadata:
      name: postgresql
      labels:
        name: postgresql
    spec:
      containers:
      - name: postgresql
        image: sameersbn/postgresql:12-20200524  #可以选择最新版本
        ports:
        - name: postgres
          containerPort: 5432
        env:
        - name: DB_USER
          value: gitlab
        - name: DB_PASS
          value: admin@mydlq
        - name: DB_NAME
          value: gitlabhq_production
        - name: DB_EXTENSION
          value: 'pg_trgm,btree_gist'
        resources: 
          requests:
            cpu: 2
            memory: 2Gi
          limits:
            cpu: 2
            memory: 2Gi
        livenessProbe:
          exec:
            command: ["pg_isready","-h","localhost","-U","postgres"]
          initialDelaySeconds: 30
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          exec:
            command: ["pg_isready","-h","localhost","-U","postgres"]
          initialDelaySeconds: 5
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: postgresql  #pvc挂载
3. gitlab 安装配置
## PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gitlab
  labels:
    app: gitlab
spec:
  capacity:          
    storage: 100Gi
  accessModes:       
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
  - hard
  - nfsvers=4.1    
  nfs:
    server: x.x.x.x
    path: /nfs/gitlab/git
---
## PVC
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: gitlab
spec:
  resources:
    requests:
      storage: 100Gi
  accessModes:
  - ReadWriteOnce
  selector:
    matchLabels:
      app: gitlab
---
## Deployment
kind: Deployment
apiVersion: apps/v1
metadata:
  name: gitlab
  labels:
    name: gitlab
spec:
  replicas: 1
  selector:
    matchLabels:
      name: gitlab
  template:
    metadata:
      name: gitlab
      labels:
        name: gitlab
    spec:
      containers:
      - name: gitlab
        image: 'sameersbn/gitlab:13.8.4'
        ports:
        - name: ssh
          containerPort: 22
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        env:
        - name: TZ
          value: Asia/Shanghai
        - name: GITLAB_TIMEZONE
          value: Beijing
        - name: GITLAB_SECRETS_DB_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_SECRETS_SECRET_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_SECRETS_OTP_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_ROOT_PASSWORD
          value: admin@mysql@1       #登录密码
        - name: GITLAB_ROOT_EMAIL 
          value: 553126784@.com     
        - name: GITLAB_HOST           
          value: 'gitlab.xxxx.com'
        - name: GITLAB_PORT        
          value: '80'                   
        - name: GITLAB_SSH_PORT    #此处是service映射端口，如果映射端口是
          value: '31001'
        - name: GITLAB_NOTIFY_ON_BROKEN_BUILDS
          value: 'true'
        - name: GITLAB_NOTIFY_PUSHER
          value: 'false'
        - name: DB_TYPE             
          value: postgres
        - name: DB_HOST         
          value: gitlab-postgresql           
        - name: DB_PORT          
          value: '5432'
        - name: DB_USER        
          value: gitlab
        - name: DB_PASS         
          value: admin@mydlq
        - name: DB_NAME          
          value: gitlabhq_production
        - name: REDIS_HOST
          value: gitlab-redis              
        - name: REDIS_PORT      
          value: '6379'
      
        resources: 
          requests:
            cpu: 4
            memory: 8Gi
          limits:
            cpu: 4
            memory: 8Gi
        livenessProbe:
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 300
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 5
          timeoutSeconds: 30
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        volumeMounts:
        - name: data
          mountPath: /home/git/data
        - name: localtime
          mountPath: /etc/localtime
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: gitlab
      - name: localtime
        hostPath:
          path: /etc/localtime
---
#关于gitlab Service生成，这个自己看个人了喜好了。
#第一种方式
kind: Service
apiVersion: v1
metadata:
  name: gitlab
  labels:
    name: gitlab
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: http
    - name: https
      protocol: TCP
      port: 443 
      targetPort: https
    - name: ssh
      protocol: TCP
      port: 22
      targetPort: ssh
      nodePort: 31001
  type: NodePort
  selector:
    name: gitlab
---
第二种方式是拆分上面的。
kind: Service
apiVersion: v1
metadata:
  name: gitlab
  labels:
    name: gitlab
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: http
    - name: https
      protocol: TCP
      port: 443 
      targetPort: https
  selector:
    name: gitlab
---
kind: Service
apiVersion: v1
metadata:
  name: gitlab-ssh
  labels:
    name: gitlab-ssh
spec:
  ports:
    - name: ssh
      protocol: TCP
      port: 22
      targetPort: ssh
      nodePort: 31001
  type: NodePort
  selector:
    name: gitlab
---
#gitlab服务对外访问ingress
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: gitlab-ui.htexam.com
  namespace: devops
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`gitlab.xxxx.com`)
      kind: Rule
      services:
        - name: gitlab
          port: 80
访问：http://gitlab.xxxx.com 账号：root 密码：admin@mysql@1
