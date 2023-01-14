# Chart Template Guide

Helm charts are structured like this:

```
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
  ...
```

The `templates/` directory is for template files. When Helm evaluates a chart, it will send all of the files in the templates/ directory through the template rendering engine. It then collects the results of those templates and sends them on to Kubernetes.

The `values.yaml` file is also important to templates. This file contains the default values for a chart. These values may be overridden by users during helm install or helm upgrade.

The `Chart.yaml` file contains a description of the chart. You can access it from within a template.

The `charts/` directory may contain other charts (which we call subcharts).

## A Starter Chart

We'll create a simple chart called mychart:

```
$ helm create mychart
Creating mychart
```

If you take a look at the `mychart/templates/` directory, you'll notice a few files already there.

- NOTES.txt: The "help text" for your chart. This will be displayed to your users when they run helm install.
- deployment.yaml: A basic manifest for creating a Kubernetes deployment
- service.yaml: A basic manifest for creating a service endpoint for your deployment
- _helpers.tpl: A place to put template helpers that you can re-use throughout the chart

Let's begin by creating a file called mychart/templates/configmap.yaml:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

In virtue of the fact that this file is in the mychart/templates/ directory, it will be sent through the template engine. It is just fine to put a plain YAML file like this in the mychart/templates/ directory. When Helm reads this template, it will simply send it to Kubernetes as-is.

With this simple template, we now have an installable chart. And we can install it like this:
```
$ helm install full-coral ./mychart
```

Using Helm, we can retrieve the release and see the actual template that was loaded.

```
$ helm get manifest full-coral

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

The `helm get manifest` command takes a release name (full-coral) and prints out all of the Kubernetes resources that were uploaded to the server. Each file begins with --- to indicate the start of a YAML document, and then is followed by an automatically generated comment line that tells us what template file generated this YAML document.

Now we can uninstall our release: `helm uninstall full-coral`

### Adding a Simple Template Call

Hard-coding the `name:` into a resource is usually considered to be bad practice. Names should be unique to a release. So we might want to generate a name field by inserting the release name.

Let's alter configmap.yaml accordingly.
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```

The big change comes in the value of the name: field, which is now {{ .Release.Name }}-configmap.

The template directive `{{ .Release.Name }}` injects the release name into the template. The values that are passed into a template can be thought of as namespaced objects, where a dot (.) separates each namespaced element.

The leading dot before Release indicates that we start with the top-most namespace for this scope. So we could read .Release.Name as "start at the top namespace, find the Release object, then look inside of it for an object called Name.

Now when we install our resource, we'll immediately see the result of using this template directive:

```
$ helm install clunky-serval ./mychart
```

You can run `helm get manifest clunky-serval` to see the entire generated YAML.