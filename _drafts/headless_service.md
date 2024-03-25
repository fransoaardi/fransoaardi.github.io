리서치중:
- k8s svc 동작관련..
    nodeport
    clusterip
    loadbalancer
    + headless service: 

nslookup 을 통해서 확인해보니 일단 DNS 는 kube-system NS 의 kube-dns svc 에 요청하고있는게 확인된다.
```
$ k get svc -A
kube-system      kube-dns                                    ClusterIP      172.20.0.10
```

참고: nslookup 은 debian 에서 `apt install dnsutils` 로 설치했다.

- headless service 를 nslookup 했을때는 각 pod 의 ip(cluster-ip) 가 각각의 A 레코드로 등록됨
```
$ nslookup testsvc.stage-platform.svc.cluster.local

Server:		172.20.0.10
Address:	172.20.0.10#53

Name:	testsvc.stage-platform.svc.cluster.local
Address: 10.2.3.232
Name:	testsvc.stage-platform.svc.cluster.local
Address: 10.2.1.11
```

- headless 아닌 svc 를 호출했을때는 k8s 의 svc 의 clusterIP 가 등장한다. 심지어 
```
$ nslookup gcp-apis.stage-platform.svc.cluster.local

Server:		172.20.0.10
Address:	172.20.0.10#53

Name:	gcp-apis.stage-platform.svc.cluster.local
Address: 172.20.126.150
```

```
apiVersion: v1
kind: Service
metadata:
  name: testsvc
  namespace: stage-platform
spec:
  selector:
    app: gcp-apis
  clusterIP: None
  ports:
  - port: 8080
    targetPort: 8080
    name: http
```