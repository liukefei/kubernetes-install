##安装部署
在master节点的/home/k8-install/k8s-v1.10.1-manual/k8s-v1.10.1/kubernetes/addons/下面建立文件kube-dns.yml

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-dns
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
  namespace: kube-system

apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.1.100.100
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  strategy:
    rollingUpdate:
      maxSurge: 10%
      maxUnavailable: 0
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      dnsPolicy: Default
      serviceAccountName: kube-dns
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      volumes:
      - name: kube-dns-config
        configMap:
          name: kube-dns
          optional: true
      containers:
      - name: kubedns
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-kube-dns-amd64:1.14.7
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        livenessProbe:
          httpGet:
            path: /healthcheck/kubedns
            port: 10054
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /readiness
            port: 8081
            scheme: HTTP
          initialDelaySeconds: 3
          timeoutSeconds: 5
        args:
        - "--domain=cluster.local"
        - --dns-port=10053
        - --v=2
        env:
        - name: PROMETHEUS_PORT
          value: "10055"
        ports:
        - containerPort: 10053
          name: dns-local
          protocol: UDP
        - containerPort: 10053
          name: dns-tcp-local
          protocol: TCP
        - containerPort: 10055
          name: metrics
          protocol: TCP
        volumeMounts:
        - name: kube-dns-config
          mountPath: /kube-dns-config
      - name: dnsmasq
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.7
        livenessProbe:
          httpGet:
            path: /healthcheck/dnsmasq
            port: 10054
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        args:
        - "-v=2"
        - "-logtostderr"
        - "-configDir=/etc/k8s/dns/dnsmasq-nanny"
        - "-restartDnsmasq=true"
        - "--"
        - "-k"
        - "--cache-size=1000"
        - "--log-facility=-"
        - "--server=/cluster.local/127.0.0.1#10053"
        - "--server=/in-addr.arpa/127.0.0.1#10053"
        - "--server=/ip6.arpa/127.0.0.1#10053"
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        resources:
          requests:
            cpu: 150m
            memory: 20Mi
        volumeMounts:
        - name: kube-dns-config
          mountPath: /etc/k8s/dns/dnsmasq-nanny
      - name: sidecar
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-sidecar-amd64:1.14.7
        livenessProbe:
          httpGet:
            path: /metrics
            port: 10054
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        args:
        - "--v=2"
        - "--logtostderr"
        - "--probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.cluster.local,5,A"
        - "--probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.cluster.local,5,A"
        ports:
        - containerPort: 10054
          name: metrics
          protocol: TCP
        resources:
          requests:
            memory: 20Mi
            cpu: 10m

```
注：clusterIP: 10.1.100.100 根据构建的环境调整地址

##检查dns服务：
```
kubectl apply -f kube-dns.yml

kubectl -n kube-system get po -l k8s-app=kube-dns
NAME                        READY     STATUS    RESTARTS   AGE
kube-dns-7774c69f5b-xcgsm   3/3       Running   0          55m
```
##验证dns服务：
```
kubectl create -f https://k8s.io/examples/admin/dns/busybox.yaml
pod/busybox created

kubectl get pods busybox
NAME      READY     STATUS    RESTARTS   AGE
busybox   1/1       Running   0          <some-time>
  
kubectl exec -ti busybox -- nslookup baidu.com
Server:    10.1.100.100
Address 1: 10.1.100.100 kube-dns.kube-system.svc.cluster.local

Name:      baidu.com
Address 1: 123.125.115.110
Address 2: 220.181.57.216
```

