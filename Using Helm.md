# Using Helm

Ref: https://helm.sh/docs/intro/using_helm/

## Three big concepts

- **Chart**: A Chart is a Helm package. It contains all of the resource definitions necessary to run an application, tool, or service inside of a Kubernetes cluster.
- **Repository**: A Repository is the place where charts can be collected and shared
- **Release**: A Release is an instance of a chart running in a Kubernetes cluster. One chart can often be installed many times into the same cluster. And each time it is installed, a new release is created. Consider a MySQL chart. If you want two databases running in your cluster, you can install that chart twice. Each one will have its own release, which will in turn have its own release name.


## helm search

Helm comes with a powerful search command. It can be used to search two different types of source:

- **helm search hub**: searches the Artifact Hub, which lists helm charts from dozens of different repositories.
- **helm search repo**: searches the repositories that you have added to your local helm client (with helm repo add). This search is done over local data, and no public network connection is needed.

With no filter, helm search hub shows you all of the available charts.

## `helm install`: Installing a Package

To install a new package, use the `helm install` command. At its simplest, it takes two arguments: 
- A release name that you pick
- The name of the chart you want to install

```
$ helm install happy-panda bitnami/wordpress
```
The release above is named happy-panda. (If you want Helm to generate a name for you, leave off the release name and use --generate-name.)

During installation, the helm client will print useful information about which resources were created, what the state of the release is, and also whether there are additional configuration steps you can or should take.

Helm does not wait until all of the resources are running before it exits. Many charts require Docker images that are over 600M in size, and may take a long time to install into the cluster.

To keep track of a release's state, or to re-read configuration information, you can use `helm status`:
```
$ helm status happy-panda
```

### Customizing the Chart Before Installing

Many times, you will want to customize the chart to use your preferred configuration.

To see what options are configurable on a chart, use helm show values. To see what options are configurable on a chart, use `helm show` values:

```
$ helm show values bitnami/wordpress
```

There are two ways to pass configuration data during install:
- `--values` (or `-f`): Specify a YAML file with overrides. This can be specified multiple times and the rightmost file will take precedence
- `--set`: Specify overrides on the command line.

If both are used, `--set` values are merged into `--values` with higher precedence. Overrides specified with --set are persisted in a ConfigMap. Values that have been --set can be viewed for a given release with `helm get values <release-name>`. Values that have been --set can be cleared by running `helm upgrade with --reset-values specified`.

### The Format and Limitations of `--set`
