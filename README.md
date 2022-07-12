# k8s-ntfy-deployment-service


This is an automatic k8s ntfy deployment from "https://ntfy.sh/docs/install/"  which also creates the admin user account and password.<br>
After installing just create an traefik or nginx ingress pointing your domain "https://ntfy.example.com" to the k8s service "ntfy"<br>
This Setup works greate with kuma-uptime Monitor from https://github.com/louislam/uptime-kuma :-)<br>

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: ntfy-server-config
  labels:
    app.kubernetes.io/name: ntfy
data:
  NTFY_AUTH_FILE: /var/lib/ntfy/user.db
  NTFY_AUTH_DEFAULT_ACCESS: deny-all
  NTFY_BASE_URL: "https://ntfy.example.com"
  NTFY_UPSTREAM_BASE_URL: "https://ntfy.sh"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ntfy-user-config
  labels:
    app.kubernetes.io/name: ntfy
data:
  NTFY_USERNAME: admin
  NTFY_PASSWORD: admin
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ntfy
  labels:
    app: ntfy
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ntfy
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ntfy
    spec:
      containers:
      - name: ntfy
        image: binwiederhier/ntfy:latest
        imagePullPolicy: Always
        command: ["ntfy", "serve"]
        envFrom:
        - configMapRef:
            name: ntfy-server-config  
        env:      
        ports:
        - containerPort: 80
          protocol: TCP
        volumeMounts:
        - name: user-db
          mountPath: "/var/lib/ntfy/"
      initContainers:
      - name: init-user-db
        image: binwiederhier/ntfy:latest
        imagePullPolicy: Always
        command: ["/bin/sh", "-c"]
        args:
        - |
          ntfy serve &
           if [ $? -eq 0 ]; then
             while true; do sleep 3 && test -f $NTFY_AUTH_FILE && break; done
             ntfy user add --role=admin $NTFY_USERNAME
             pkill -TERM ntfy
           else
             exit $?
           fi
        envFrom:
        - configMapRef:
            name: ntfy-server-config
        - configMapRef:
            name: ntfy-user-config
        volumeMounts:
        - name: user-db
          mountPath: "/var/lib/ntfy/"
      volumes:
      - name: user-db
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: ntfy
  name: ntfy
  namespace: default
spec:
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: 80-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app.kubernetes.io/name: ntfy
  sessionAffinity: None
  type: ClusterIP
  
  
