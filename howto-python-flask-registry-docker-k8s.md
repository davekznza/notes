This doc contains some notes on how to deploy a python flask app as a container to Kubernetes and utilize a private docker repository for local testing (Ubuntu).

# Prep app file structure

```sh
$ mkdir myapp
$ mkdir myapp/templates
$ touch myapp/app.py
$ touch myapp/templates/index.html
$ touch myapp/requirements.txt
$ touch myapp/Dockerfile
$ touch myapp/myapp-deploy.yaml
$ touch myapp/myapp-service.yaml
```

# Create Image (eg. python flask app)

app.py
```python
# basic todo app

from flask import Flask, render_template, request
from flask_wtf import FlaskForm
from wtforms import StringField, SubmitField

app = Flask(__name__)
app.config['SECRET_KEY'] = 'mysecret'

todos = ["Create basic python flask app", "Deploy with k8s", "Test open telementry"]

class TodoForm(FlaskForm):
    todo = StringField("Todo")
    submit = SubmitField("Add Todo")

@app.route('/', methods=["GET", "POST"])
def index():
    if 'todo' in request.form:
        todos.append(request.form['todo'])
    return render_template("index.html", todos=todos, template_form=TodoForm())
    
if __name__ == "__main__":
    app.run()
```

templates/index.html
```html
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <h1>Todos</h1>

    {% for todo in todos %}
        <li>{{ todo }}</li>
    {% endfor %}

    <form method="POST">
        {{ template_form.hidden_tag() }}
        <p>
            {{ template_form.todo.label }}
            {{ template_form.todo() }}
        </p>
        <p>
            {{ template_form.submit() }}
        </p>
    </form>
</body>
</html>
```

Setup python virtual environment, install required python dependencies and create a `requirements.txt` file:
```sh
$ python3 -m venv venv
$ source venv/bin/activate

(venv) $ pip3 install flask flask_wtf		# only required if you wish to do a test run before moving onto building the image
(venv) $ pip3 freeze > requirements.txt
```

Test run the app and verify access to it at http://localhost:5555/, then exit the venv:
```sh
(venv) python3 -m flask run -h 127.0.0.1 -p 5555

 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:5555/ (Press CTRL+C to quit)
 
127.0.0.1 - - [16/Mar/2022 14:25:19] "GET / HTTP/1.1" 200 -

^C

(venv) deactivate
```

Dockerfile
```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.8-slim-buster
WORKDIR /myapp
COPY requirements.txt requirements.txt
RUN pip3 install --upgrade pip
RUN pip3 install -r requirements.txt
COPY . .
EXPOSE 5555
CMD [ "python3", "-m" , "flask", "run", "--host=0.0.0.0", "--port=5555" ]
```

Build our docker image `myapp:latest`:
```sh
$ docker build --no-cache=true --tag myapp:latest .
..
Successfully tagged myapp:latest
```

Check your image list:
```sh
$ docker images

REPOSITORY                    TAG               IMAGE ID       CREATED          SIZE
myapp                         latest            f83ba412ee1e   21 minutes ago   148MB
```

Test run our docker image in docker and verify access to it at http://localhost:5555/, then stop / remove it:
```sh
$ docker run -d -p 5555:5555 --name myapp myapp:latest
..
$ docker stop myapp
$ docker rm myapp
```

# Start / Enable Registry

Start the registry using minikube:
```sh
$ minikube addons enable registry

    ▪ Using image registry:2.7.1
    ▪ Using image gcr.io/google_containers/kube-registry-proxy:0.4
🔎  Verifying registry addon...
🌟  The 'registry' addon is enabled
```

**Important:** Redirect port 5000 on the docker virtual machine over to port 5000 on the minikube machine, we must do this before we can use docker to push to our registry:
```sh
$ docker run --rm -it --network=host alpine ash -c "apk add socat && socat TCP-LISTEN:5000,reuseaddr,fork TCP:$(minikube ip):5000"

Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
59bf1c3509f3: Already exists 
Digest: sha256:21a3deaa0d32a8057914f36584b5288d2e5ecc984380bc0118285c70fa8c9300
Status: Downloaded newer image for alpine:latest
fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/community/x86_64/APKINDEX.tar.gz
(1/4) Installing ncurses-terminfo-base (6.3_p20211120-r0)
(2/4) Installing ncurses-libs (6.3_p20211120-r0)
(3/4) Installing readline (8.1.1-r0)
(4/4) Installing socat (1.7.4.2-r0)
Executing busybox-1.34.1-r3.trigger
OK: 7 MiB in 18 packages

^C
```

For more info see https://minikube.sigs.k8s.io/docs/handbook/registry/

# Push Image to Registry

Tag our image and push it to our registry:
```sh
$ docker tag myapp:latest localhost:5000/myapp:latest

$ docker push localhost:5000/myapp:latest

The push refers to repository [localhost:5000/myapp]
ae2bb3ec38d0: Pushed 
17cb5184baa0: Pushed 
cab5ef7f4188: Pushed 
20c83a207145: Pushed 
4d4dcbbb93d9: Pushed 
6fafa3f64716: Pushed 
e41449df8276: Pushed 
469c8dbcb08a: Pushed 
5d5d684babe6: Pushed 
e5baccb54724: Pushed 
latest: digest: sha256:c435a4215e2f5a53d93588c9455a537f61ad731257abb4e0a037e7bbe0f81b62 size: 2417
```

# Deploy to k8s

Add new namespace for `myapp`:
```sh
$ kubectl create namespace myapp

namespace/myapp created
```

Create a deployment manifest file `myapp-deploy.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: localhost:5000/myapp:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 5555
```

Deploy to our `myapp` namespace:
```sh
$ kubectl apply -f myapp-deploy.yaml -n myapp

deployment.apps/myapp-deploy created
```

Check if deployment is running with:
```sh
$ kubectl get deploy -n myapp

NAME           READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deploy   1/1     1            1           9s
```

Quick check to see if we can reach the app via http://localhost:5555 (we run a temporary local port forward to the pod’s container port):
```sh
$ kubectl port-forward deployment/myapp-deploy -n myapp 5555:5555

Forwarding from 127.0.0.1:5555 -> 5555
Forwarding from [::1]:5555 -> 5555
Handling connection for 5555

^C
```

Now we will create a Kubernetes service to create a stable network for the running pod.
Create a service manifest file `myapp-service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  labels:
    app: myapp
spec:
  type: LoadBalancer 
  ports:
  - port: 5555
    targetPort: 5555
    protocol: TCP
  selector:
    app: myapp
```

Create the service (this may take a brief moment):
```sh
$ kubectl apply -f myapp-service.yaml -n myapp

service/myapp-svc created
```

At this point we need to start a tunnel with minikube since we are exposing a service of `type=LoadBalancer`, this will create a routable IP for our deployment which we can access it via.
```sh
$ minikube tunnel

Status:	
	machine: minikube
	pid: 259738
	route: 10.96.0.0/12 -> 192.168.49.2
	minikube: Running
	services: [basic-todos-svc, myapp-svc]
    errors: 
		minikube: no errors
		router: no errors
		loadbalancer emulator: no errors

```

Once we see an `EXTERNAL-IP` for the service, then you can access it via browser at http://EXTERNAL-IP:5555
```sh
$ kubectl get svc -w -n myapp

NAME        TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
myapp-svc   LoadBalancer   10.97.113.108   <pending>     5555:31748/TCP   31s
```

Remove orphaned routes (eg. if minikube tunnel shut down abruptly)
```sh
$ minikube tunnel --cleanup
```

# Cleanup

Cancel the tunnel and then go ahead and delete the service, deployment and namespace:
```sh
$ kubectl delete service -n myapp myapp-svc
service "myapp-svc" deleted

$ kubectl delete deploy -n myapp myapp-deploy
deployment.apps "myapp-deploy" deleted

$ kubectl delete namespace myapp
namespace "myapp" deleted
```