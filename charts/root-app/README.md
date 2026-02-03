# Applications and root-app
Following instructions from [this](https://www.arthurkoziel.com/setting-up-argocd-with-helm/) tutorial.  

The root-app is a Helm chart that renders [Application](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications)  manifests. 

Create the Application manifest for our root-app in charts/root-app/templates/root-app.yaml

## Let Argo CD manage itself
create an Application manifest that points to our Argo CD chart.
``` charts/root-app/templates/argo-cd.yaml```

## Installing Prometheus
create an Application manifest in charts/root-app/templates/prometheus.yaml that uses the Prometheus helm chart 

## 
