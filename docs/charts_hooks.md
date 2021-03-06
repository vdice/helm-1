# Hooks

Helm provides a _hook_ mechanism to allow chart developers to intervene
at certain points in a release's life cycle. For example, you can use
hooks to:

- Load a ConfigMap or Secret during install before any other charts are
  loaded.
- Execute a Job to backup up a database before installing a new chart,
  and then execute a second job after the upgrade in order to restore
  data.
- Run a Job before deleting a release to gracefully take a service out
  of rotation before removing it.

Hooks work like regular templates, but they have special annotations
that cause Helm to utilize them differently. In this section, we cover
the basic usage pattern for hooks.

## The Available Hooks

The following hooks are defined:

- pre-install: Executes after templates are rendered, but before any
  resources are created in Kubernetes.
- post-install: Executes after all resources are loaded into Kubernetes
- pre-delete: Executes on a deletion request before any resources are
  deleted from Kubernetes.
- post-delete: Executes on a deletion request after all of the release's
  resources have been deleted.
- pre-upgrade: Executes on an upgrade request after templates are
  rendered, but before any resources are loaded into Kubernetes (e.g.
  before a kuberntes apply operation).
- post-upgrade: Executes on an upgrade after all resources have been
  upgraded.
- pre-rollback: Executes on a rollback request after templates are
  rendered, but before any resources have been rolled back.
- post-rollback: Executes on a rollback request after all resources
  have been modified.

## Hooks and the Release Lifecycle

Hooks allow you, the chart developer, an opportunity to perform
operations at strategic points in a release lifecycle. For example,
consider the lifecycle for a `helm install`. By default, the lifecycle
looks like this:

1. User runs `helm install foo`
2. Chart is loaded into Tiller
3. After some verification, Tiller renders the `foo` templates
4. Tiller loads the resulting resources into Kubernetes
5. Tiller returns the release name (and other data) to the client
6. The client exits

Helm defines two hooks for the `install` lifecycle: `pre-install` and
`post-install`. If the developer of the `foo` chart implements both
hooks, the lifecycle is altered like this:

1. User runs `helm install foo`
2. Chart is loaded into Tiller
3. After some verification, Tiller renders the `foo` templates
4. Tiller executes the `pre-install` hook (loading hook resources into
   Kubernetes)
5. Tiller waits until the hook is "Ready"
6. Tiller loads the resulting resources into Kubernetes
7. Tiller executes the `post-install` hook (loading hook resources)
8. Tiller waits until the hook is "Ready"
9. Tiller returns the release name (and other data) to the client
10. The client exits

What does it mean to wait until a hook is ready? This depends on the
resource declared in the hook. If the resources is a `Job` kind, Tiller
will wait until the job successfully runs to completion. And if the job
fails, the release will fail. This is a _blocking operation_, so the
Helm client will pause while the Job is run.

For all other kinds, as soon as Kubernetes marks the resource as loaded
(added or updated), the resource is considered "Ready". When many
resources are declared in a hook, the resources are executed serially,
but the order of their execution is not guaranteed.

### Hook resources are unmanaged

The resources that a hook creates are not tracked or managed as part of the
release. Once Tiller verifies that the hook has reached its ready state, it
will leave the hook resource alone.

Practically speaking, this means that if you create resources in a hook, you
cannot rely upon `helm delete` to remove the resources. To destroy such
resources, you need to write code to perform this operation in a `pre-delete`
or `post-delete` hook.

## Writing a Hook

Hooks are just Kubernetes manfiest files with special annotations in the
`metadata` section. Because they are template files, you can use all of
the normal template features, including reading `.Values`, `.Release`,
and `.Template`.

For example, this template, stored in `templates/post-install-job.yaml`,
declares a job to be run on `post-install`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{.Release.Name}}"
  labels:
    heritage: {{.Release.Service | quote }}
    release: {{.Release.Name | quote }}
    chart: "{{.Chart.Name}}-{{.Chart.Version}}"
  annotations:
    # This is what defines this resource as a hook. Without this line, the
    # job is considered part of the release.
    "helm.sh/hook": post-install
spec:
  template:
    metadata:
      name: "{{.Release.Name}}"
      labels:
        heritage: {{.Release.Service | quote }}
        release: {{.Release.Name | quote }}
        chart: "{{.Chart.Name}}-{{.Chart.Version}}"
    spec:
      restartPolicy: Never
      containers:
      - name: post-install-job
        image: "alpine:3.3"
        command: ["/bin/sleep","{{default "10" .Values.sleepyTime}}"]

```

What makes this template a hook is the annotation:

```
  annotations:
    "helm.sh/hook": post-install
```

One resource can implement multiple hooks:

```
  annotations:
    "helm.sh/hook": post-install,post-upgrade
```

Similarly, there is no limit to the number of different resources that
may implement a given hook. For example, one could declare both a secret
as a config map as a pre-install hook. It is important to keep in mind,
though, that there are no ordering guarantees about hooks.

When subcharts declare hooks, those are also evaluated. There is no way
for a top-level chart to disable the hooks declared by subcharts. And
again, there is no guaranteed ordering.

