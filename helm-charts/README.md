# Notes: Helm Charts

General notes around helm-charts. 

Additional related notes to see:
- Howto: Create a basic helm chart and deploy to k8s
- Howto: Create your own helm chart repository

References / documentation used to create this note:
- https://www.baeldung.com/ops/kubernetes-helm
- https://helm.sh/docs/

## helm get

To see which charts we have installed and which releases:
```sh
$ helm ls --all --all-namespaces

NAME       	NAMESPACE  	REVISION	UPDATED                                 	STATUS  	CHART            	APP VERSION
basic-todos	basic-todos	1       	2022-03-18 15:09:08.978531814 +0200 SAST	deployed	basic-todos-0.1.0	1.16.0     
hello-world	default    	1       	2022-03-18 08:02:35.337480655 +0200 SAST	deployed	hello-world-0.1.0	1.16.0     
```

## helm upgrade

Say we modify our chart and need to install the upgraded version. This helps us to upgrade a release to a specified or current version of the chart or configuration. Note that with Helm 3, the release upgrade uses a three-way strategic merge patch. Here, it considers the old manifest, cluster live state, and new when generating a patch. 

For example, I've updated `replicaCount` in our chart's `values.yaml` to use `5` replica's now instead of `1`. Use `kubectl get pods -n myapp` to check the number of pods deployed before and after running the upgrade. Also notice how the **REVISION** number is increased after the upgrade.
```sh
$ helm upgrade --namespace myapp myapp ./myapp

Release "myapp" has been upgraded. Happy Helming!
NAME: myapp
LAST DEPLOYED: Fri Mar 18 15:42:39 2022
NAMESPACE: myapp
STATUS: deployed
REVISION: 2
TEST SUITE: None
```

## helm rollback

Say we need to roll back to a previous revision / version for some reason, this is simple. Note that if we don't specify a version number to roll back to then it will roll back to the previous version.
```sh
$ helm rollback --namespace myapp myapp 1

Rollback was a success! Happy Helming!
```

## helm uninstall

To uninstall a release from Kubernetes completely, we can do the following (this removes all the resources associated with the last release of the chart and the release history):
```sh
$ helm uninstall --namespace myapp myapp

release "myapp" uninstalled
```

## helm package

If we wish to distribute our charts, we need first need to package them. This creates a versioned archive file of our chart:
```sh
$ helm package ./myapp

Successfully packaged chart and saved it to: /home/dave/helm/myapp-0.1.0.tgz
```

Now we can distribute our chart manually or through public or private chart repositories. There are also options to sign the chart archive.

## helm repo

To collaborate we first need a method to work with shared repositories. This command has several sub-commands we can use to add, remove, update, list or index chart repositories.

### Add a helm repo

Eg. adding a repo you're hosting locally with Nginx:
```sh
$ helm repo add localcharts http://localhost/charts/

"localcharts" has been added to your repositories
```

Eg. adding a github repo:
```sh
$ helm repo add githubcharts https://davekznza.github.io/charts/
"githubcharts" has been added to your repositories
```

### Fetch repo updates

```sh
$ helm repo update

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "localcharts" chart repository
Update Complete. ⎈Happy Helming!⎈
```

### Install a chart from a repo
```sh
$ helm install myapp localcharts/myapp --namespace myapp --create-namespace 

NAME: myapp
LAST DEPLOYED: Wed Mar 23 16:54:33 2022
NAMESPACE: myapp
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

### Creating a custom repo

See **Howto: Create your own helm chart repository** for more info on this (`howto-helm-charts-create-your-own-local-or-github-repo`)

## helm search

To list or search across our configured repos (add final search text or leave as is to list all):
```sh
$ helm search repo

NAME                   	CHART VERSION	APP VERSION	DESCRIPTION                
localcharts/myapp   	0.1.1        	1.16.0     	A Helm chart for Kubernetes
```

To search for charts in the Artifact Hub we can do:
```sh
$ helm search hub nginx

URL                                               	CHART VERSION   	APP VERSION              	DESCRIPTION                                       
https://artifacthub.io/packages/helm/cloudnativ...	3.2.0           	1.16.0                   	Chart for the nginx server  
...
```
