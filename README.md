# kustomization-base

This repo contains a base kustomization that can be used to install Spinnaker to
a Kubernetes cluster. Please see the
[RFC](https://github.com/spinnaker/governance/blob/master/rfc/halyard-lite.md)
for an overview of the new Kubernetes-native installation pathway that is under
development.

This repo is under active development and is not ready for production (or even
development) use, but stay tuned for updates!

## Installing Spinnaker

### Prerequisites

Before you start, you will need:

- A Kubernetes cluster and `kubectl` configured to communicate with that cluster
- The [kustomize](https://github.com/kubernetes-sigs/kustomize/releases/latest)
  binary installed on your machine. Note that the version of _kustomize_ bundled
  with `kubectl` is out of date and will not work with these kustomizations.
- The [kleat](https://github.com/spinnaker/kleat) binary installed on your
  machine

### Instructions

Fork the [spinnaker-config](https://github.com/ezimanyi/spinnaker-config)
repository, and clone this fork to your local machine. This repository contains
a template kustomization that you will configure to deploy Spinnaker.

Change to the `base` directory; all further commands will assume that you are in
this directory.

At this point, you can generate the YAML needed to deploy Spinnaker by running
`kustomize build .`. As we have not configured any of the services, many would
fail on startup (particularly those that depend on some configured persistence
store), so let's add some configuration.

#### Create a `spinnaker` namespace

Create a new namespace called `spinnaker` into which each microservice will be
deployed:

```
kubectl create namespace spinnaker
```

If you would like to use a different namespace, you will need to update the
namespace in your root `kustomization.yml` and each microservice's `baseUrl` in
a fork of this repository's `service-discovery/spinnaker.yml`.

#### Configure Redis

A number of Spinnaker services require a Redis connection; this install pathway
requires there to be a service `redis` in the `spinnaker` namespace.

In the most common case, you will have an external redis; the template repo has
a `Service` configured with an `ExternalName`; you can update this to point to
the DNS name of your redis service.

For more complex cases, please refer to the following
[blog post](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-mapping-external-services)
on best practices for mapping external services. In general, the only
requirement of your solution is that you have a service named `redis` in the
`spinnaker` namespace that routes to a valid `redis` backend.

Regardless of the approach you choose, add all the relevant redis Kubernetes
objects to your customization via `kustomize edit add resource redis/*.yml`.

#### Add any secret files

The `secrets` folder is intended to hold any secret files that are referenced by
your config files. The files in this folder will be mounted in `/var/secrets` in
each pod.

To add a secret, put it in the `secrets/` folder and add it to the `files` field
of the `spinnaker-secrets` entry in the `kustomization.yaml`. In this example,
we'll add a _kubeconfig_ file called `k8s-kubeconfig`:

```shell script
cp <path to kubeconfig> secrets/k8s-kubeconfig
```

and update the secret in the `kustomization.yaml` file as:

```yaml
- behavior: merge
  name: spinnaker-secrets
  files:
    - secrets/k8s-kubeconfig
```

#### Generate Spinnaker config files

This installation pathway expects the config files for your services to be in the
`kleat` directory. As the name suggests, the expectation is that these files
will be generated by `kleat` and written to this directory.

`kleat` takes as input a
[halconfig](https://github.com/spinnaker/kleat/blob/master/docs/docs.md). This
file is in general compatible with the file consumed by Halyard; if you are
migrating from Halyard, please read the minor changes you will need to make to
your file in the [kleat readme](https://github.com/spinnaker/kleat).

Unlike Halyard, `kleat` allows you to specify only the fields that you'd like to
configure. In this case, we will assume a very simple config:

```yaml
providers:
  kubernetes:
    enabled: true
    accounts:
      - name: k8s
        kubeconfigFile: /var/secrets/k8s-kubeconfig
    primaryAccount: k8s
```

Note that the `kubeconfigFile` argument here points to a file relative to
`/var/secrets`. This is important, as the `kleat` config requires that all file
references be relative to where they will be mounted in the running container
(rather than where they happen to be on the machine running kleat).

Save this file as `halconfig.yml` to the current directory, Invoke `kleat generate` to
consume the input `halconfig.yml` and output the service configs to the `kleat/`
directory:

```shell script
kleat generate halconfig.yml kleat/
```

This will generate service configs in the `kleat/` directory, replacing the
empty configs that were there before. The `kustomization.yaml` file is already
configured to mount these configs in the correct location in each microservice.
For example, it has an entry for the clouddriver config as:

```yaml
- behavior: merge
  files:
    - kleat/clouddriver.yml
  name: clouddriver-config
```

#### Enable optional services

The two optional Spinnaker services this workflow currently supports are Fiat
and Kayenta.

In the following kustomization files, we recommend including `?ref=${KUSTOMIZATION_BASE_REF}` to pin kustomization-base to a specific version.
For further details, see [(Optional) Use a specific version of kustomization](#optional-use-a-specific-version-of-kustomization).

To enable Kayenta, ensure that `canary.enabled` is set to `true` in your hal
config, and then ensure the Kayenta kustomization is included in the `resources`
block of your `kustomization.yml`:

```
resources:
- github.com/spinnaker/kustomization-base/core?ref=${KUSTOMIZATION_BASE_REF}
- github.com/spinnaker/kustomization-base/kayenta?ref=${KUSTOMIZATION_BASE_REF}
```

Since the default
[service discovery file](/core/service-discovery/spinnaker.yml) defaults
Kayenta to disabled, you will need to configure a service discovery overrides
file in your fork of spinnaker-config with the following block:

```
services:
  kayenta:
    enabled: true
```

To enable Fiat, ensure that `security.authz` is set to `true` in your hal
config, and then ensure the Fiat kustomization is included in the `resources`
block of your `kustomization.yml`:

```
resources:
- github.com/spinnaker/kustomization-base/core?ref=${KUSTOMIZATION_BASE_REF}
- github.com/spinnaker/kustomization-base/fiat?ref=${KUSTOMIZATION_BASE_REF}
```

Since the default
[service discovery file](/core/service-discovery/spinnaker.yml) defaults
Fiat to disabled, you will need to configure a service discovery overrides file
in your fork of spinnaker-config with the following block:

```
services:
  fiat:
    enabled: true
```

#### Set the Spinnaker version

By default, this base kustomization deploys the `master-latest-validated`
version of each microservice, which is most recent version that has passed our
integration tests. To deploy a different version, you'll need to override the
version of each microservice in the `images` block in the `kustomization.yml`
file.

To deploy a specific version of Spinnaker, override each image's tag with
`spinnaker-{version-number}`. For example, to deploy Spinnaker 1.21.0, override
the tag for each microservice to be `spinnaker-1.21.0`:

```yaml
images:
  - name: us-docker.pkg.dev/spinnaker-community/docker/clouddriver
    newTag: spinnaker-1.20.5
  - name: us-docker.pkg.dev/spinnaker-community/docker/deck
    newTag: spinnaker-1.20.5
# ...
```

This workflow has only been tested with Spinnaker >= 1.21.0. Deploying an
earlier version of Spinnaker using this workflow may not work as expected,
since you may have been relying on Halyard to supply configuration defaults that
were only added as microservice-level defaults after the 1.20 release branches
were cut.

#### Replace Gate's readiness probe

If you have not enabled SSL for Gate, override Gate's readiness probe with
the following patch:

```yaml
readinessProbe:
  $patch: replace
  exec:
    command:
    - wget
    - --no-check-certificate
    - --spider
    - -q
    - http://localhost:8084/health
```

Reference the patch in your base `kustomization.yml` by adding the following to
a `patches` block:

```yaml
- target:
    kind: Deployment
    name: gate
  path: path/to/my/readiness/probe/patch.yml
```

#### (Optional) Use a specific version of kustomization

With a reference of the version, you can use a specific version with conviction, after examining if the version works well with your configurations. Without a reference, a resource link always references `master`. You can check out the available versions [here](https://github.com/spinnaker/kustomization-base/releases).

For example:

```yaml
resources:
- github.com/spinnaker/kustomization-base/core?ref=v0.1.0
```

For further details, see [kustomization.yaml · The Kubectl Book](https://kubectl.docs.kubernetes.io/pages/examples/kustomize.html).

#### (Optional) Add any -local configs

In addition to the main `service.yml` config file, each microservice also reads
in the contents of `service-local.yml` to support settings that are not
configurable in the _halconfig_.

If you would like to add a `-local.yml` config file for any service, add it to
the `local/` directory, and update that service's config in the
`kustomization.yaml` to also mount that `local.yml` file.

For example, to configure for clouddriver, add these settings to
`local/clouddriver-local.yml`, and update the `clouddriver-config` entry in the
`kustomization.yaml` to:

```yaml
- behavior: merge
  files:
    - kleat/clouddriver.yml
    - local/clouddriver-local.yml
  name: clouddriver-config
```

#### (Optional) Enable monitoring

The Spinnaker monitoring daemon runs as a sidecar in each Deployment (excluding
Deck). To enable monitoring, copy the [monitoring](/monitoring) directory from
this repository into the `base` directory of your fork of spinnaker-config.

Add the `monitoring` directory to your base kustomization.yml's `resource`
block. This will pull in the kustomization.yml that includes configuration that
each microservice's monitoring sidecar will use to discover the endpoint to poll
for metrics.

Next, copy the [example `patches` block](/monitoring/patches.yml) into your base
kustomization.yml. These patches will add the monitoring sidecar and appropriate
volumes to each Deployment.

To include custom
[metric filters](https://www.spinnaker.io/setup/monitoring/#configuring-metric-filters),
add them to the included `metric-filters` directory in your fork of
spinaker-config, and reference them in the spinnaker-monitoring-filters
`secretGenerator` entry in the root kustomization.yml.

#### Deploy Spinnaker

Now that all of the config files are in place, you can generate the YAML files
to install Spinnaker by running `kustomize build .`. You can either save this to
a file and apply it, or directly pipe it to `kubectl` via:

```shell script
kustomize build . | kubectl apply -f -
```
