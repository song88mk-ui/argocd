# ArgoCD

The repository contains Helm charts and configuration files used by the application deployed on the
EKS clusters.

- The `apps` directory contains environment specific values for each application, grouped by their namespace.
- The `charts` directory contain generic Helm charts that the apps use. The goal is to try to make
  them as generic as possible so we can keep duplication to a minimum.

## How it works

In the [infrastructure][infra_repo] repo we create ArgoCD [`Applications`][argocd_apps]. Those are
custom Kubernetes resources ([CRDS][crds]) which specify how an application is configured and how it
will be deployed by ArgoCD. In essence they are referencing one of the Helm charts on this repo and
the corresponding values file for the environment we're deploying.

Once this resource is created, ArgoCD will be able to manage it and deploy any new changes that
happen on this repository. Those changes can be either about a change on the chart or a change on the
values. Either one will trigger a new deployment.

The most common change is regarding the image tag which is done automatically by [Image
Updater][image_updater]. When a new image is pushed by the corresponding application repo the image
updater will be notified about the change and create a new commit on this repo updating the values
file with the new image tag. This will trigger a new deployment by ArgoCD.

[infra_repo]: https://github.com/FlyingCockatoo/infrastructure
[crds]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
[argocd_apps]: https://argo-cd.readthedocs.io/en/latest/user-guide/application-specification/
[image_updater]: https://argocd-image-updater.readthedocs.io/en/stable/
