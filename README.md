# helm
Helm related stuff

# How-to get up and running with helm
1. Download and install helm [client](https://github.com/kubernetes/helm)

```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
The current local state of Helm is kept in your environment in the home location

The Helm command defaults to discovering the host already set in ~/.kube/config. There is a way to change or override the host.

```
helm env
```

#### Search For Chart

You can now start deploying applications to Kubernetes based on public charts. To find available charts use the search command.

For example, to deploy Redis search the Helm Hub for that chart by name.

```
helm search hub redis
```

To add the Google & other repositories 

```
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo add fabric8 https://fabric8.io/helm
```

#### Install a Chart

```
kubectl create namespace redis
helm install my-redis stable/redis --namespace redis
```

With the install command Helm will launch the required deployments, ReplicaSets, Pods, Services, ConfigMaps or any other Kubernetes resource the chart defines. View all the installed charts.

```
helm list
watch kubectl get deployments,pods,services -n redis
```

#### Remove Chart

```
helm delete my-redis
```

#### Explore Repositories
```
helm repo list
helm search repo stable | sed -E "s/(.{27}).*$/\1/"
helm search hub --max-col-width=80 | sed -E "s/(.{70}).*$/\1/"
```

#### Incubator Charts
```
helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
```

#### Create Chart
Charts are helful when creating your own solutions. Application charts are often a combination on 3rd party public charts as well as your own. The first step is to create your own chart.

This will create the directory my-app-chart as the skeleton for your chart. All chart directories will have these standard files and directories.

```
helm create app-chart
```

All of your Kubernetes resource definitions in YAML files are located in the templates directory. Take a look at the top of deployments.yaml.

```
cat app-chart/templates/deployment.yaml | grep 'kind:' -n -B1 -A5
```

Notice it looks like a normal deployment yaml with the kind: Deployment defined. However, there is new syntax sugar using double braces {{ .. }}. The is the templating mechanism that Helm uses to inject values into this template. Instead of hard coding in values instead this templating injects values. The templating language has many features by leveraging the Go templating API.

What about defining the container image for the deployment? That is an injected value as well.

```
cat app-chart/templates/deployment.yaml | grep 'image:' -n -B3 -A3
```

Notice the `{{ .Values.image.repository }}` this is where the container name gets injected. All of these values have defaults typically found in the values.yaml file in the chart directory.

Notice the templating key uses the dot ('.') notation to navigate and extract the values from the hierarchy in the values.yaml.

In this case the Helm create feature defaulted the deployed container to be the ubiquitous demonstration application nginx.

As is, this chart is ready to be deployed since all the defaults have been supplied. A complete set of sensible defaults is a good practice for any chart you author. A good README for your chart should also have a table to reflect these defaults, options and descriptions.

Before deploying to Kubernetes, the dry-run feature will list out the resources to the console. This allows to you inspect the injection of the values into the template without committing an installation, a helpful development technique. Observe how the container image name is injected into the template.

```
helm install my-app ./app-chart --dry-run --debug | grep 'image: "' -n -B3 -A3
```

Notice the `ImagePullPolicy` is set to the default of `IfNotPreset`. Before we deploy the chart we could modify the values.yaml file and change the policy value in there, but perhaps we would like to locally modify a different policy setting first to verify it works. Use the --set command to override a default value. Here we change the Nginx container image `ImagePullPolicy` from `IfNotPreset` to `Always`.

```
helm install my-app ./app-chart --dry-run --debug --set image.pullPolicy=Always | grep 'image: "' -n -B3
```
With the version injecting correctly, install it.

```
helm install my-app ./app-chart --set image.pullPolicy=Always
helm list
kubectl get deployments,service
```

#### Update Chart

Look at the service. Notice the service type is ClusterIP. To see the Nginx default page we would like to instead expose it as a NodePort. A kubectl patch could be applied, but it would be best to change the values.yaml file. Perhaps this is just to verify. We could simply change the installed application with a new value. Use the Helm upgrade command to modify the deployment.

```
helm upgrade my-app ./app-chart --install --reuse-values --set service.type=NodePort
```

Well, this demonstration chart is a bit deficient as it does not allow the values for the NodePort to be assigned. Right now it's a random value. We could modify the chart template to accept a nodePort value, but for this exercise apply this quick patch.

```
kubectl patch service my-app-app-chart --type='json' --patch='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value":31111}]'
```
