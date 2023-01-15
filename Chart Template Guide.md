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

When you want to test the template rendering, but not actually install anything, you can use `helm install --debug --dry-run goodly-guppy ./mychart`. This will render the templates. But instead of installing the chart, it will return the rendered template to you so you can see the output.

#
## Built-in Objects

Objects are passed into a template from the template engine. And your code can pass objects around. There are even a few ways to create new objects within your templates.

Objects can be simple, and have just one value. Or they can contain other objects or functions. For example. the  `Release` object contains several objects (like Release.Name) and the Files object has a few functions.

Release is one of the top-level objects that you can access in your templates.
- Release: This object describes the release itself. It has several objects inside of it:
  - Release.Name
  - Release.Namespace
  - Release.IsUpgrade
  - Release.IsInstall
  - Release.Revision
  - Release.Service
- Values: Values passed into the template from the values.yaml file and from user-supplied files. By default, Values is empty.
- Chart: The contents of the Chart.yaml file. Any data in Chart.yaml will be accessible here. For example {{ .Chart.Name }}-{{ .Chart.Version }} will print out the mychart-0.1.0.
- Files: This provides access to all non-special files in a chart. While you cannot use it to access templates, you can use it to access other files in the chart.
- Capabilities: This provides information about what capabilities the Kubernetes cluster supports.
  - Capabilities.APIVersions is a set of versions.
  - Capabilities.KubeVersion and Capabilities.KubeVersion.Version is the Kubernetes version.
- Template: Contains information about the current template that is being executed
  - `Template.Name`: A namespaced file path to the current template
  - `Template.BasePath`: The namespaced path to the templates directory of the current chart

#
## Values Files

One of the built-in objects is Values. This object provides access to values passed into the chart.

Values files are plain YAML files. `values.yaml` can be overriden by a user-supplied values file, which in turn be overriden by `--set` parameters.

Removing the defaults in values.yaml, we'll set just one parameter:
```
favoriteDrink: coffee
```

Now we can use this inside of a template:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favoriteDrink }}
```

Values files can contain more structured content, too. For example, we could create a favorite section in our values.yaml file, and then add several keys there:

```
favorite:
  drink: coffee
  food: pizza
```

Now we would have to modify the template slightly:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink }}
  food: {{ .Values.favorite.food }}
```

### Deleting a default key

If you need to delete a key from the default values, you may override the value of the key to be null, in which case Helm will remove the key from the overridden values merge.

#
## Template Functions and Pipelines

Sometimes we want to transform the supplied data in a way that makes it more useable to us.

Let's start with a best practice: When injecting strings from the .Values object into the template, we ought to quote these strings. We can do that by calling the quote function in the template directive:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```

Template functions follow the syntax `functionName arg1 arg2....` In the snippet above, quote .Values.favorite.drink calls the quote function and passes it a single argument.

Helm has over 60 available functions.

Helm is actually a combination of the Go template language, some extra functions, and a variety of wrappers to expose certain objects to the templates. Many resources on Go templates may be helpful as you learn about templating.

## Pipelines

Drawing on a concept from UNIX, pipelines are a tool for chaining together a series of template commands to compactly express a series of transformations. In other words, pipelines are an efficient way of getting several things done in sequence. Let's rewrite the above example using a pipeline.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | quote }}
```

In this example, instead of calling quote ARGUMENT, we inverted the order. We "sent" the argument to the function using a pipeline (|): .Values.favorite.drink | quote. Using pipelines, we can chain several functions together:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

When pipelining arguments like this, the result of the first evaluation (.Values.favorite.drink) is sent as the last argument to the function.

### Using the `default` function

One function frequently used in templates is the default function: default DEFAULT_VALUE GIVEN_VALUE. This function allows you to specify a default value inside of the template, in case the value is omitted. Let's use it to modify the drink example above:
```
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```

### Using the `lookup` function

The `lookup` function can be used to look up resources in a running cluster. The synopsis of the lookup function is lookup apiVersion, kind, namespace, name -> resource or resource list.

### Logic and Flow control functions

Helm includes numerous logic and control flow functions including `and`, `coalesce`, `default`, `empty`, `eq`, `fail`, `ge`, `gt`, `le`, `lt`, `ne`, `not`, and `or`.

- The `coalesce` function takes a list of values and returns the first non-empty one.
- The `ternary` function takes two values, and a test value. If the test value is true, the first value will be returned. If the test value is empty, the second value will be returned.
- `lt` Returns a boolean true if the first argument is less than the second.
- `le` Returns a boolean true if the first argument is less than or equal to the second. False is returned otherwise 

#
## Flow Control

Control structures (called "actions" in template parlance) provide you, the template author, with the ability to control the flow of a template's generation. Helm's template language provides the following control structures:
- `if/else` for creating conditional blocks
- `with` to specify a scope
- `range`, which provides a "for each"-style loop

### If/Else

The first control structure we'll look at is for conditionally including blocks of text in a template. This is the if/else block. Control structures can execute an entire pipeline, not just evaluate a value.

```
{{ if PIPELINE }}
  # Do something
{{ else if OTHER PIPELINE }}
  # Do something else
{{ else }}
  # Default case
{{ end }}
```

A pipeline is evaluated as false if the value is:
- a boolean false
- a numeric zero
- an empty string
- a nil (empty or null)
- an empty collection (map, slice, tuple, dict, array)
Under all other conditions, the condition is true.

Let's add a simple conditional to our ConfigMap. We'll add another setting if the drink is set to coffee:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}mug: "true"{{ end }}
```