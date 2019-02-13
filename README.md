# Demo of Kubernetes load balancing on NodePort

We are using kubespray v2.4.0 to build Kubernetes clusters and they use iptables method
for [kube-proxy](https://github.com/kubernetes-sigs/kubespray/blob/v2.4.0/roles/kubernetes/node/defaults/main.yml#L17).
Apparently, this does not support true round robin load balancing.

But to prove or disprove this, we have a demo.

Here's a service that uses port 8888 on a pod and exposes that service to port 30090
on a NodePort:

```
$ cat demo1-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: demo1
  labels:
    demo: demo1
spec:
  type: NodePort
  ports:
    - name: http
      port: 8888
      targetPort: 8888
      protocol: TCP
      nodePort: 30090
  selector:
    app: dp-server
```

Here we have a deployment using the busybox image:

```
$ cat dp-deployment.yaml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: dp-server
spec:
  replicas: 2
  strategy: {}
  template:
    metadata:
      name: dp-server
      labels:
        app: dp-server
    spec:
      hostname: dp-server
      containers:
      - image: busybox
        imagePullPolicy: Always
        name: dp-server
        command: [ "sleep", "999999" ]
```

Note there are 2 replicas.  We goto a k8s cluster and apply these yamls:

```
$ kubectl apply -f demo1-svc.yaml dp-deployment.yaml 
$ kubectl apply -f dp-deployment.yaml 
```

We go to two separate terminals and do this on terminal 1 (note the first pod
shows "AAAAAAAA" and the second shows "BBBBBBB" to help distinguish):

```
$ m1=$(kubectl get po | grep dp-server | head -1 | cut -f 1 -d ' ')
$ kubectl exec -ti $m1 -- sh
# cat << END > index.html
> <html><head><title>Test Connectivity pod AAAAAAA ***********</title></head></html>
> END
# while [ 1 ]; do nc -l -p 8888 < index.html ; done
```

We do this on terminal 2:

```
$ m2=$(kubectl get po | grep dp-server | tail -1 | cut -f 1 -d ' ')
$ kubectl exec -ti $m2 -- sh
/ # cat << END > index.html
> <html><head><title>Test Connectivity pod BBBBBBB ***********</title></head></html>
> END
/ # while [ 1 ]; do nc -l -p 8888 < index.html ; done

```

The above creates a simple webserver and returns output so that we can
run curl and easily identify what pod got the request.

Here's a sample run using one of the IPs on the k8s cluster (using nodePort, you can
run the curl on any node IP on the k8s cluster):

```
$ anIP=x.x.x.x
$ for i in {1..20} ; do curl http://$anIP:30090 ; done
<html><head><title>Test Connectivity pod BBBBBBBB ***********</title></head></html>
<html><head><title>Test Connectivity pod BBBBBBBB ***********</title></head></html>
<html><head><title>Test Connectivity pod AAAAAAA ***********</title></head></html>
<html><head><title>Test Connectivity pod AAAAAAA ***********</title></head></html>
<html><head><title>Test Connectivity pod AAAAAAA ***********</title></head></html>
<html><head><title>Test Connectivity pod AAAAAAA ***********</title></head></html>
<html><head><title>Test Connectivity pod BBBBBBBB ***********</title></head></html>
<html><head><title>Test Connectivity pod BBBBBBBB ***********</title></head></html>
<html><head><title>Test Connectivity pod BBBBBBBB ***********</title></head></html>
<html><head><title>Test Connectivity pod BBBBBBBB ***********</title></head></html>
<html><head><title>Test Connectivity pod AAAAAAA ***********</title></head></html>
<html><head><title>Test Connectivity pod AAAAAAA ***********</title></head></html>
<html><head><title>Test Connectivity pod AAAAAAA ***********</title></head></html>
<html><head><title>Test Connectivity pod BBBBBBBB ***********</title></head></html>
<html><head><title>Test Connectivity pod BBBBBBBB ***********</title></head></html>
<html><head><title>Test Connectivity pod AAAAAAA ***********</title></head></html>
<html><head><title>Test Connectivity pod AAAAAAA ***********</title></head></html>
<html><head><title>Test Connectivity pod BBBBBBBB ***********</title></head></html>
<html><head><title>Test Connectivity pod AAAAAAA ***********</title></head></html>
<html><head><title>Test Connectivity pod AAAAAAA ***********</title></head></html>
```

Apparently, there is some level of "load balancing" on NodePort.
