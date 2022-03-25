# Howto: Create a basic helm chart and deploy to k8s

Quick example which deploys a container called **myapp** which runs on `http://0.0.0.0:5555`.
- The image used here is hosted on my local docker registry (`localhost:5000`) which k8s can already access (note I'm using minikube)
- This uses a simple python flask app container created with docker: `localhost:5000/myapp:latest`
- There are examples for both **NodePort** and **LoadBalancer** deployment options

Additional related notes to see:
- [Notes: Helm Charts](https://github.com/davekznza/notes/blob/main/helm-charts/README.md)
- [Howto: Create your own helm chart repository](https://github.com/davekznza/notes/blob/main/helm-charts/howto-helm-charts-create-custom-chart-repo.md)

References / documentation used to create this note:
- https://www.baeldung.com/ops/kubernetes-helm
- https://helm.sh/docs/

## 1. Create chart

```sh
$ helm create myapp
Creating myapp
```

Delete some templates which I don't think we require right now, just to keep things smiple / lean:

```sh
$ rm -fr myapp/templates/{hpa.yaml,ingress.yaml,serviceaccount.yaml,NOTES.txt,tests}
```

## 2. Create deployment & service templates  

See template options below for both **NodePort** and **LoadBalancer** type deployments.

### NodePort templates

For this **NodePort** example, a random port will be exposed and traffic will be forwarded to the container's port `5555`.

myapp/templates/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "myapp.name" . }}
    helm.sh/chart: {{ include "myapp.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "myapp.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ include "myapp.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 5555
              protocol: TCP
```

myapp/templates/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "myapp.name" . }}
    helm.sh/chart: {{ include "myapp.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: {{ include "myapp.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
```

myapp/values.yaml:
```yaml
replicaCount: 1
image:
  repository: "localhost:5000/myapp"
  tag: "latest"
  pullPolicy: IfNotPresent
service:
  type: NodePort
  port: 5555
```

### LoadBalancer templates

For our **LoadBalancer** example we are exposing port `5556` which forwards to the deployment `containerPort` of `5555`.

myapp/templates/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "myapp.name" . }}
    helm.sh/chart: {{ include "myapp.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "myapp.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ include "myapp.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.deployment.containerPort }}
              protocol: TCP
```

myapp/templates/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "myapp.name" . }}
    helm.sh/chart: {{ include "myapp.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: {{ include "myapp.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
```

myapp/values.yaml:
```yaml
replicaCount: 1
image:
  repository: "localhost:5000/myapp"
  tag: "latest"
  pullPolicy: IfNotPresent
service:
  type: LoadBalancer
  port: 5556
deployment:
  containerPort: 5555
```

##  3. Check for errors & validate template data

Ensure our chart is well formed and error free:
```sh
$ helm lint ./myapp
==> Linting myapp
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

Next, render the template to verify that our values were correctly injected into the template definitions.

NodePort example:
```sh
$ helm template myapp

---
# Source: myapp/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: release-name-myapp
  labels:
    app.kubernetes.io/name: myapp
    helm.sh/chart: myapp-0.1.0
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
spec:
  type: NodePort
  ports:
    - port: 5555
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: release-name
---
# Source: myapp/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: release-name-myapp
  labels:
    app.kubernetes.io/name: myapp
    helm.sh/chart: myapp-0.1.0
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
      app.kubernetes.io/instance: release-name
  template:
    metadata:
      labels:
        app.kubernetes.io/name: myapp
        app.kubernetes.io/instance: release-name
    spec:
      containers:
        - name: myapp
          image: "localhost:5000/myapp:latest"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 5555
              protocol: TCP
```

LoadBalancer example:
```sh
$ helm template myapp

---
# Source: myapp/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: release-name-myapp
  labels:
    app.kubernetes.io/name: myapp
    helm.sh/chart: myapp-0.1.0
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
spec:
  type: LoadBalancer
  ports:
    - port: 5556
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: release-name
---
# Source: myapp/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: release-name-myapp
  labels:
    app.kubernetes.io/name: myapp
    helm.sh/chart: myapp-0.1.0
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
      app.kubernetes.io/instance: release-name
  template:
    metadata:
      labels:
        app.kubernetes.io/name: myapp
        app.kubernetes.io/instance: release-name
    spec:
      containers:
        - name: myapp
          image: "localhost:5000/myapp:latest"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 5555
              protocol: TCP
```

## 4. Install / deploy our chart

Install / deploy the chart, make sure to specify specify a namespace called `myapp` and specify to have it created:
```sh
$ helm install myapp ./myapp --namespace myapp --create-namespace

NAME: myapp
LAST DEPLOYED: Fri Mar 18 10:41:53 2022
NAMESPACE: myapp
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

## 5. Check status of deployment

Check with `helm status`:
```sh
$ helm status --namespace myapp myapp

NAME: myapp
LAST DEPLOYED: Fri Mar 18 10:41:53 2022
NAMESPACE: myapp
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Check k8s service, deployment and pod status with `kubectl`:

NodePort example:
```sh
$ kubectl get service -n myapp

NAME    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
myapp   NodePort   10.109.102.138   <none>        5555:32560/TCP   16s

$ kubectl get deploy -n myapp

NAME    READY   UP-TO-DATE   AVAILABLE   AGE
myapp   1/1     1            1           22s

$ kubectl get pod -n myapp

NAME                     READY   STATUS    RESTARTS   AGE
myapp-5f87849596-dtj4w   1/1     Running   0          29s
```

LoadBalancer example:
```sh
$ kubectl get service -n myapp

NAME     TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
myapp    LoadBalancer   10.105.177.94   <pending>     5556:32717/TCP   26s

$ kubectl get deploy -n myapp

NAME     READY   UP-TO-DATE   AVAILABLE   AGE
myapp    1/1     1            1           53s

$ kubectl get pod -n myapp

NAME                     READY   STATUS    RESTARTS   AGE
myapp-cb5cdd46b-nbrlj    1/1     Running   0          76s
```

## 6. Access deployment

Access `myapp` via browser to verify it's up and running.

### NodePort deployment

Use `minikube service list` to find the exposed endpoint for **myapp** and open that in a browser - http://192.168.49.2:32560
```sh
$ minikube service list
|----------------------|---------------------------|--------------|---------------------------|
|      NAMESPACE       |           NAME            | TARGET PORT  |            URL            |
|----------------------|---------------------------|--------------|---------------------------|
| ...                  | ...                       | ...          |                           |
| myapp                | myapp                     | http/5555    | http://192.168.49.2:32560 |
|----------------------|---------------------------|--------------|---------------------------|
```

**OR** .. create a temporary port fwd **127.0.0.1:5555 -> myapp:5555** and access it via http://127.0.0.1:5555:
```sh
$ k port-forward -n myapp service/myapp 5555:5555

Forwarding from 127.0.0.1:5555 -> 5555
Forwarding from [::1]:5555 -> 5555
^C
```

### LoadBalancer deployment

Start a tunnel with minikube in order to create a routable IP for our deployment which we can access it via (**note:** If your tunnel shuts down abruptly then remember to run `minikube tunnel --cleanup` to clean up any orphaned routes which may be left over).
```sh
$ minikube tunnel

Status:	
	machine: minikube
	pid: 259738
	route: 10.96.0.0/12 -> 192.168.49.2
	minikube: Running
	services: [myapp-svc]
    errors: 
		minikube: no errors
		router: no errors
		loadbalancer emulator: no errors

```

Once we see an `EXTERNAL-IP` for the service, then you can access it via browser at http://EXTERNAL-IP:5556
```sh
$ kubectl get service -n myapp

NAME     TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
myapp    LoadBalancer   10.105.177.94   10.105.177.94   5556:32717/TCP   2m47s
```

## 7. Cleanup

Cancel the tunnel and then go ahead and delete the service, deployment and namespace:
```sh
$ kubectl delete service -n myapp myapp
service "myapp-svc" deleted

$ kubectl delete deploy -n myapp myapp
deployment.apps "myapp-deploy" deleted

$ kubectl delete namespace myapp
namespace "myapp" deleted
```