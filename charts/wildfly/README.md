# Helm Chart for WildFly

A Helm chart for building and deploying a [WildFly](https://www.wildfly.org) application on OpenShift.

# Overview

The build and deploy steps are configured in separate `build` and `deploy` values.

The input of the `build` step is a Git repository that contains the application code and the output is an `ImageStreamTag` resource that contains the built application image.

The input of the `deploy` step is an `ImageStreamTag` resource that contains the built application image and the output is a `Deployment` and related resources to access the application from inside and outside OpenShift.

To be able to install a Helm release with that chart, you must be able to provide a valid application image.

# Compatibility with WildFly S2I images

The `2.x` Helm Chart for WildFly relies on the [new source-to-image (S2I) from WildFly](https://github.com/wildfly/wildfly-s2i/) that is available at [quay.io/wildfly/wildfly-s2i-jdk11](https://quay.io/repository/wildfly/wildfly-s2i-jdk11). It is not compatible with the legacy S2I image at [quay.io/wildfly/wildfly-centos7](https://quay.io/repository/wildfly/wildfly-centos7).

You can continue to use the `1.x` Helm Chart for WildFly with the legacy S2I images by specifying a `version` when you install with `helm`. For example, to use the latest `1.x` release of the Helm Chart, you can use:

```
helm install my-legacy-app -f app.yaml wildfly/wildfly --version ^1.x
```

## Build an Application Image from Source

If the application image must be built from source, the minimal configuration is:

```yaml
build:
  uri: <git repository URL of your application>
```

The `build` step will use OpenShift `BuildConfig` to build an application image from this Git repository.

The application must be a Maven project that is configured to use the [`org.wildfly.plugins:wildfly-maven-plugin`](https://docs.wildfly.org/wildfly-maven-plugin/) to provision a WildFly server with the deployed application. The application is built during the S2I assembly by running:

```
mvn -e -Popenshift -DskipTests -Dcom.redhat.xpaas.repo.redhatga -Dfabric8.skip=true --batch-mode -Djava.net.preferIPv4Stack=true -s /tmp/artifacts/configuration/settings.xml -Dmaven.repo.local=/tmp/artifacts/m2  package
```

Any additional Maven arguments can be specified by adding the `MAVEN_ARGS_APPEND` environment variable in the `.build.env` field:

```
build:
  env:
    - name: MAVEN_ARGS_APPEND
      value: "-P my-profile"
```


## Pull an existing Application Image

If your application image already exists, you can skip the `build` step and directly deploy your application image.
In that case, the minimal configuration is:

```yaml
image:
  name: <name of the application image. e.g. "quay.io/example.org/my-app">
  tag: <tag of the applicication image. e.g. "1.3" (defaults to "latest")>
build:
  enabled: false
```

## Prerequisites
Below are prerequisites that may apply to your use case.

### Pull Secret
You will need to create a pull secret if you need to pull an image from an external registry that requires authentication. Use the following command as a reference to create your pull secret:
```bash
oc create secret docker-registry my-pull-secret --docker-server=$REGISTRY_URL --docker-username=$USERNAME --docker-password=$PASSWORD --docker-email=$EMAIL
```

You can use this secret by passing `--set build.pullSecret=my-pull-secret` to `helm install`, or you can configure this in a values file:

```yaml
build:
  pullSecret: my-pull-secret
```

### Push Secret

You will need to create a push secret if you want to push your image to an external registry. Use the following command as a reference to create your push secret:

```bash
oc create secret docker-registry my-push-secret --docker-server=$SERVER_URL --docker-username=$USERNAME --docker-password=$PASSWORD --docker-email=$EMAIL
```

You can use this secret by passing `--set build.output.pushSecret=my-push-secret` and `--set build.output.kind=DockerImage`, or you can configure these in a values file:

```yaml
build:
  output:
    kind: DockerImage
    pushSecret: my-push-secret
```

## Application Image

The configuration for the image that is built and deployed is configured in a `image` section.

| Value | Description | Default | Additional Information |
| ----- | ----------- | ------- | ---------------------- |
| `image.name` | Name of the image you want to build/deploy | Defaults to the Helm release name. | The chart will create/reference an [ImageStream](https://docs.openshift.com/container-platform/latest/openshift_images/image-streams-manage.html) based on this value. |
| `image.tag` | Tag that you want to build/deploy | `latest` | The chart will create/reference an [ImageStreamTag](https://docs.openshift.com/container-platform/latest/openshift_images/image-streams-manage.html#images-using-imagestream-tags_image-streams-managing) based on the name provided |

## Building the Application

The configuration to build the application image is configured in a `build` section.

| Value | Description | Default | Additional Information |
| ----- | ----------- | ------- | ---------------------- |
| `build.enabled` | Determines if build-related resources should be created. | `true` | Set this to `false` if you want to deploy a previously built image. Leave this set to `true` if you want to build and deploy a new image. |
| `build.uri` | Git URI that references your git repo | &lt;required&gt; | Be sure to specify this to build the application. |
| `build.ref` | Git ref containing the application you want to build | `main` | - |
| `build.contextDir` | The sub-directory where the application source code exists | - | - |
| `build.output.kind`|	Determines if the image will be pushed to an `ImageStreamTag` or a `DockerImage` | `ImageStreamTag` | [OpenShift Documentation](https://docs.openshift.com/container-platform/4.6/builds/managing-build-output.html) |
| `build.output.pushSecret` | Name of the push secret | - | The secret must exist in the same namespace or the chart will fail to install - Used only if `build.output.kind` is `DockerImage` |
| `build.pullSecret` | Name of the pull secret | - | The secret must exist in the same namespace or the chart will fail to install - [OpenShift documentation](https://docs.openshift.com/container-platform/latesst/openshift_images/managing_images/using-image-pull-secrets.html) |
| `build.mode` | Determines whether the application will be built using WildFly S2I images or Bootable Jar | `s2i` | Allowed values: `s2i` or `bootable-jar` |
| `build.env` | Freeform `env` items | - | [Kubernetes documentation](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/). These environment variables will be used when the application is _built_. If you need to specify environment variables for the running application, use `deploy.env` instead. |
| `build.resources` | Freeform `resources` items | - | [Kubernetes documentation](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) |
| `build.images`| Freeform images injected in the source during S2I build | - | [OKD API documentation](https://docs.okd.io/latest/rest_api/workloads_apis/buildconfig-build-openshift-io-v1.html#spec-source-images-2) |
| `build.triggers.githubSecret`| Name of the secret containing the WebHookSecretKey for the GitHub Webhook | - | The secret must exist in the same namespace or the chart will fail to install - [OpenShift documentation](https://docs.openshift.com/container-platform/latest/cicd/builds/triggering-builds-build-hooks.html#builds-webhook-triggers_triggering-builds-build-hooks) |
| `build.triggers.genericSecret`| Name of the secret containing the WebHookSecretKey for the Generic Webhook | - | The secret must exist in the same namespace or the chart will fail to install - [OpenShift documentation](https://docs.openshift.com/container-platform/latest/cicd/builds/triggering-builds-build-hooks.html#builds-webhook-triggers_triggering-builds-build-hooks) |
| `build.s2i` | Configuration specific to building with WildFly S2I images | - | - |
| `build.s2i.kind` | Determines the type of images for S2I Builder and Runtime images (`DockerImage`, `ImageStreamTag` or `ImageStreamImage`) | `DockerImage` | (OKD Documentation](https://docs.okd.io/latest/cicd/builds/build-strategies.html#builds-strategy-s2i-build_build-strategies) |
| `build.s2i.builderImage` | WildFly S2I Builder image | [quay.io/wildfly/wildfly-s2i-jdk11:latest](https://quay.io/repository/wildfly/wildfly-s2i-jdk11) | [WildFly S2I documentation](https://github.com/wildfly/wildfly-s2i)  |
| `build.s2i.runtimeImage` | WildFly S2I Runtime image | [quay.io/wildfly/wildfly-runtime-jdk11:latest](https://quay.io/repository/wildfly/wildfly-runtime-jdk11) | [WildFly S2I documentation](https://github.com/wildfly/wildfly-s2i) |
| `build.s2i.galleonDir` | Directory relative to the root directory for the build that contains custom content for Galleon. | - | [WildFly S2I documentation](https://github.com/wildfly/wildfly-s2i) - since WildFly 23.0.2|
| `build.s2i.featurePacks` | *Deprecated* List of Galleon feature-packs identified by Maven coordinates (`<groupId>:<artifactId>:<version>`) | - | The value can be be either a `string` with a list of comma-separated Maven coordinate or an array where each item is the Maven coordinate of a feature pack - [WildFly S2I documentation](https://github.com/wildfly/wildfly-s2i) - since WildFly 23.0.2|
| `build.s2i.galleonLayers` | *Deprecated*   A list of layer names to compose a WildFly server. If specified, `build.s2i.featurePacks` must also be specified. | - | The value can be be either a `string` with a list of comma-separated layers or an array where each item is a layer - [WildFly S2I documentation](https://github.com/wildfly/wildfly-s2i) |
| `build.bootableJar.builderImage` | JDK Builder image for Bootable Jar | [registry.access.redhat.com/ubi8/openjdk-11:latest](https://catalog.redhat.com/software/containers/ubi8/openjdk-11/5dd6a4b45a13461646f677f4?gti-tabs=unauthenticated) | - |

### Provisioning WildFly With S2I.

The recommended way to provision the WildFly server is to use the `wildfly-maven-plugin` from the application `pom.xml`.

The `build.s2i.featurePacks` and `build.s2i.galleonLayers` fields have been deprecated as they are no longer necessary with this recommendation.
For backwards compatibility, the WildFly S2I Builder image still supports these fields to delegate to the provisioning of the server to the `wildfly-maven-plugin` if it is not configured in the application `pom.xml`.
However if `build.s2i.galleonLayers` is set, `build.s2i.featurePacks` _must_ be specified (including WildFly own feature pack, e.g. `org.wildfly:wildfly-galleon-pack:26.1.1.Final`).

## Deploying the Application

The configuration to build the application image is configured in a `deploy` section.

| Value | Description | Default | Additional Information |
| ----- | ----------- | ------- | ---------------------- |
| `deploy.annotations` | Map of `string` annotations that are applied to the deployment and its pod's `template` | - | [Kubernetes documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) |
| `deploy.enabled` | Determines if deployment-related resources should be created. | `true` | Set this to `false` if you do not want to deploy an application image built by this chart. |
| `deploy.replicas` | Number of pod replicas to deploy. | `1` | [OpenShift Documentation](https://docs.openshift.com/container-platform/latest/applications/deployments/what-deployments-are.html) | 
| `deploy.route.enabled` | Determines if a `Route` should be created | `true` | Allows clients outside of OpenShift to access your application |
| `deploy.route.host` | `host` is an alias/DNS that points to the service. Optional. If not specified a route name will typically be automatically chosen | - | [OpenShift Documentation](https://docs.openshift.com/container-platform/latest/networking/routes/route-configuration.html) |
| `deploy.route.tls.enabled` | Determines if the `Route` should be TLS-encrypted. If `deploy.tls.enabled` is true, the route will use the secure service to acess to the deployment | `true`| [OpenShift Documentation](https://docs.openshift.com/container-platform/latest/networking/routes/secured-routes.html) |
| `deploy.route.tls.termination` | Determines the type of TLS termination to use | `edge`| Allowed values: `edge, reencrypt, passthrough` |
| `deploy.route.tls.insecureEdgeTerminationPolicy` | Determines if insecure traffic should be redirected | `Redirect` | Allowed values: `Allow, Disable, Redirect` |
| `deploy.tls.enabled` | Enables the creation of a secure service to access the application. If `true`, WildFly must be configured to enable HTTPS | `false`| |
| `deploy.env` | Freeform `env` items | - | [Kubernetes documentation](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/).  These environment variables will be used when the application is _running_. If you need to specify environment variables when the application is built, use `build.env` instead. |
| `deploy.envFrom` | Freeform `envFrom` items | - | [Kubernetes documentation](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/).  These environment variables will be used when the application is _running_. If you need to specify environment variables when the application is built, use `build.envFrom` instead. |
| `deploy.resources` | Freeform `resources` items | - | [Kubernetes documentation](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) |
| `deploy.labels` | Map of `string` labels that are applied to the deployment and its pod's `template` | - | [Kubernetes documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) |
| `deploy.livenessProbe` | Freeform `livenessProbe` field. | HTTP Get on `<ip>:admin/health/live` | [Kubernetes documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) |
| `deploy.readinessProbe` | Freeform `readinessProbe` field. | HTTP Get on `<ip>:admin/health/ready` | [Kubernetes documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) |
| `deploy.startupProbe` | Freeform `startupProbe` field. | HTTP Get on `<ip>:admin/health/live` | [Kubernetes documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) |
| `deploy.volumeMounts` | Freeform `volumeMounts` items| - | [Kubernetes Documentation](https://kubernetes.io/docs/concepts/storage/volumes/) |
| `deploy.volumes` | Freeform `volumes` items| - | [Kubernetes Documentation](https://kubernetes.io/docs/concepts/storage/volumes/) |
| `deploy.initContainers` | Freeform `initContainers` items | - | [Kubernetes Documentation](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) |
| `deploy.extraContainers` | Freeform extra `containers` items | - | [Kubernetes Documentation](https://kubernetes.io/docs/concepts/workloads/pods/#pod-templates) |