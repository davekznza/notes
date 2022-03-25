# Howto: Create your own helm chart repository

Additional related notes to see:
- [Notes: Helm Charts](https://github.com/davekznza/notes/blob/main/helm-charts/README.md)
- [Howto: Create a basic helm chart and deploy to k8s](https://github.com/davekznza/notes/blob/main/helm-charts/howto-helm-charts-create-and-deploy-to-k8s.md)

References / documentation used to create this note:
- https://www.baeldung.com/ops/kubernetes-helm
- https://helm.sh/docs/howto/chart_releaser_action/
- https://helm.sh/docs/topics/chart_repository/#github-pages-example
- https://helm.sh/docs/topics/chart_repository/#ordinary-web-servers

## Using your own GitHub repository

We can use a GitHub repository to host our helm charts and use it as our repo. Our repo will contain 2 branches called `main` and `gh-pages`. We push our charts to `main` and use a github workflow which will publish our charts to the `gh-pages` branch, where helm will use to pull the chart packages from. I followed hte process as per the helm documentation here: https://helm.sh/docs/howto/chart_releaser_action/#github-actions-workflow.

### 1. Create the GitHub repository

First start by creating a new public repo in GitHub. I created one called https://github.com/davekznza/charts.git

### 2. Initial commit and create branches main and gh-pages

Do an initial commit and push a simple `README.md` to the `main` branch, also create the `gh-pages` branch and push the same `README.md` for now:

```sh
mkdir ~/charts
cd ~/charts

git init

echo "# charts" >> README.md
git add README.md
git commit -m "initial commit"

git branch -M main
git branch gh-pages

git remote add origin https://github.com/davekznza/charts.git

git push -u origin main
git push -u origin gh-pages
```

### 3. Add our GitHub workflow

For `main` we'll add a new git workflow:

```sh
mkdir -p .github/workflows
vim .github/workflows/release.yml
```

.github/workflows/release.yml:

```yaml
name: Release Charts

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "$GITHUB_ACTOR"
          git config user.email "$GITHUB_ACTOR@users.noreply.github.com"

      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.1.0
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
```

Commit and push our new github workflow to `main` branch:

```sh
git add .github/
git commit -m "add new github workflow"
git push -u origin main
```

Now when we push our charts to `main` (they go into a subfolder called `charts`), the workflow action will check the chart and whenever there is a new chart version it will create a corresponding GitHub release name for the chart version in the `gh-pages` branch, add Helm artifacts to the release and create or update the `index.yaml` with metadata about the release.

### 4. Add a chart to the main branch

Create the `charts` subdirectory in `main` first and then add one of our charts to push:

```sh
mkdir charts
cp -Rp /some/path/myapp charts/
git push origin -u main
```

I also mainually push the chart archive package to the `gh-pages` branch as well (after manually building it with `helm package` first), at this point I'm not sure if there is an easier way to automate including your chart archive package as well, this part is a bit of a work in progress till I learn a better way.

```sh
git checkout gh-pages
git pull
cp /some/path/myapp-0.1.0.tgz ./
git add myapp-0.1.0.tgz
git commit -m "add myapp archive package"
git push origin -u gh-pages
```

### 5. Summary / Overview

For info here are is what the layout of each of the branches looks like:

```sh
$ git branch
* main

$ tree
.
├── charts
│   └── myapp
│       ├── Chart.yaml
│       ├── templates
│       │   ├── deployment.yaml
│       │   ├── _helpers.tpl
│       │   └── service.yaml
│       └── values.yaml
└── README.md

3 directories, 6 files

$ git checkout gh-pages
Branch 'gh-pages' set up to track remote branch 'gh-pages' from 'origin'.
Switched to a new branch 'gh-pages'

$ tree
.
├── myapp-0.1.0.tgz
├── index.yaml
└── README.md

0 directories, 3 files
```

## Using a self-hosted repo with Nginx

Assuming you already have nginx running locally and set up with `/var/www/html` configured as the root and accessible via http://localhost/:

```sh
$ cd ~/mycharts

$ helm package myapp/

Successfully packaged chart and saved it to: ~/mycharts/myapp-0.1.0.tgz

$ mkdir /var/www/html/charts

$ cp -Rp ~/mycharts/myapp /var/www/html/charts/
$ cp ~/mycharts/myapp-0.1.0.tgz /var/www/html/charts/

$ cd /var/www/html
$ helm repo index charts/ --url http://localhost/charts/

$ cat /var/www/html/charts/index.yaml

apiVersion: v1
entries:
  myapp:
  - apiVersion: v2
    appVersion: 1.16.0
    created: "2022-03-23T15:24:24.195686003+02:00"
    description: A Helm chart for Kubernetes
    digest: 26efe02018e7c98b7f6d729c1f6e7410a02f0be4df9ad23c78c871e1ff148250
    name: myapp
    type: application
    urls:
    - http://localhost/charts/myapp-0.1.0.tgz
    version: 0.1.1
generated: "2022-03-23T15:24:24.194185885+02:00"
```

## Next Steps

See additional note **Notes: Helm Charts** on how to work with helm repos using `helm repo`.
