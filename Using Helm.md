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

Read the details [here](https://helm.sh/docs/intro/using_helm/#the-format-and-limitations-of---set)

The --set option takes zero or more name/value pairs. At its simplest, it is used like this: `--set name=value`.

More complex expressions 
- `--set outer.inner=value`
- `--set name={a, b, c}`: Lists can be expressed by enclosing values in { and }
- `--set name=[],a=null`:Certain name/key can be set to be null or to be an empty array []
- `--set servers[0].port=80`
- `--set servers[0].port=80,servers[0].host=example`: Multiple values can be set this way
- `--set name=value1\,value2`: You can use a backslash to escape the characters
- `--set nodeSelector."kubernetes\.io/role"=master`

### More Installation Methods

The `helm install` command can install from several sources:
- A chart repository
- A local chart archive
- An unpacked chart directory
- A full URL

## `helm upgrade` and `helm rollback`: Upgrading a Release, and Recovering on Failure

An upgrade takes an existing release and upgrades it according to the information you provide. Because Kubernetes charts can be large and complex, Helm tries to perform the least invasive upgrade. It will only update things that have changed since the last release.

```
$ helm upgrade -f panda.yaml happy-panda bitnami/wordpress
```

We can use `helm get values` to see whether that new setting took effect.

```
$ helm get values happy-panda
mariadb:
  auth:
    username: user1
```

The `helm get` command is a useful tool for looking at a release in the cluster.

Now, if something does not go as planned during a release, it is easy to roll back to a previous release using helm rollback [RELEASE] [REVISION].

```
$ helm rollback happy-panda 1
```

The above rolls back our happy-panda to its very first release version. A release version is an **incremental revision**. Every time an install, upgrade, or rollback happens, the revision number is incremented by 1. The first revision number is always 1. And we can use `helm history [RELEASE]` to see revision numbers for a certain release.

## Helpful Options for Install/Upgrade/Rollback

There are several other helpful options you can specify for customizing the behavior of Helm during an install/upgrade/rollback.
- `--timeout`: A Go duration value to wait for Kubernetes commands to complete. This defaults to 5m0s.
- `--wait`: Waits until all Pods are in a ready state, PVCs are bound, Deployments have minimum (Desired minus maxUnavailable) Pods in ready state and Services have an IP address (and Ingress if a LoadBalancer) before marking the release as successful. It will wait for as long as the --timeout value. If timeout is reached, the release will be marked as FAILED.
- `--no-hooks`: This skips running hooks for the command.
- `--recreate-pods`: (only available for upgrade and rollback): This flag will cause all pods to be recreated 

## `helm uninstall`: Uninstalling a Release

When it is time to uninstall a release from the cluster, use the helm uninstall command:

```
$ helm uninstall happy-panda
```

You can see all of your currently deployed releases with the `helm list` command.

In previous versions of Helm, when a release was deleted, a record of its deletion would remain. In Helm 3, deletion removes the release record as well. If you wish to keep a deletion release record, use helm uninstall `--keep-history`. Using `helm list --uninstalled` will only show releases that were uninstalled with the `--keep-history` flag.

The `helm list --all` flag will show you all release records that Helm has retained, including records for failed or deleted items (if --keep-history was specified).

```
$  helm list --all
```

## `helm repo`: Working with Repositories

You can see which repositories are configured using `helm repo list`. And new repositories can be added with `helm repo add`

```
$ helm repo add dev https://example.com/dev-charts
```

Because chart repositories change frequently, at any point you can make sure your Helm client is up to date by running `helm repo update`.

Repositories can be removed with `helm repo remove`.

## Creating Your Own Charts

You can create Helm chart quickly by using the `helm create` command:

```
$ helm create deis-workflow
Creating deis-workflow
```
As you edit your chart, you can validate that it is well-formed by running `helm lint`.

When it's time to package the chart up for distribution, you can run the `helm package` command:

```
$ helm package deis-workflow
deis-workflow-0.1.0.tgz
```

And that chart can now easily be installed by helm install:

```
$ helm install deis-workflow ./deis-workflow-0.1.0.tgz
...
```