# deployment-config

Kubernetes manifests for python-flaskapp, reconciled by Flux.

Jenkins updates the image tag in `base/deployment.yaml`, field:
`spec.template.spec.containers[0].image`

Flux watches this repo's `base/` directory and applies changes automatically.
No manifests here should be applied manually with kubectl.
