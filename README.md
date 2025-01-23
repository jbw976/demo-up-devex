# demo-up-devex

This demo walks through the streamlined developer experience of Upbound, from
the perspective of a Crossplane community member that has already been building
control planes with upstream community tools. It will define a Bucket storage
API that enables end users to request storage buckets with accompanying ACLs
from our control plane.

This walkthrough accompanies the presentation from this [slide
deck](https://docs.google.com/presentation/d/1VlGXbgREX7V-oXQ_lmPdMu17jPgs2-Wwf7bFmaX_I6k/edit?usp=sharing).

## Pre-requisites

The `up` tooling is required for this demo, which can be installed from the
[Upbound
docs](https://docs.upbound.io/getstarted/control-plane-project/#install-the-up-cli).

Here are some quick installation steps to get started with just the `up` tooling
(you'll likely want to install additional native language tooling if you haven't
already):
```
curl https://cli.upbound.io/ | CHANNEL=alpha sh
sudo mv up /usr/local/bin/
up version
up login
up profile list
```

## Design the API

The first step in designing our platform API will be to initialize (scaffold) a
new project:
```
up project init demo-up-devex && cd demo-up-devex
```

Then we can generate a new blank claim with a type of our choosing, that will
serve as a blank canvas for us to design our API:
```
up example generate \
    --type claim \
    --api-group demoupdevex.upbound.io \
    --api-version v1alpha1 \
    --kind StorageBucket \
    --name example \
    --namespace default
```

We can fill in this example claim with the configuration knobs that we want to
expose to our end users. Go ahead and update the `.spec` field of `example.yaml`
with:
```yaml
spec:
  parameters:
    location: US
    versioning: true
    acl: publicRead
```

Now that we've designed our API, we can simply generate the XRD schema from it.
This saves an enormous amount of time and effort, and removes a very common
source of bugs in our control planes:
```
up xrd generate examples/storagebucket/example.yaml
```

All the OpenAPIv3 schema, types, and object structure has been inferred from the
example claim and generated for us automatically - how easy!

## Compose Resources

Next, we can start defining the logic for how resources should be composed at
runtime when our end user requests an instance of our API. Start with generating
a new composition, that will include the function pipeline with our logic we'll
define later:
```
up composition generate apis/xstoragebuckets/definition.yaml
```

This particular composition we are building will create and compose storage
buckets in GCP. Let's add the GCP storage provider to our project as a
dependency, which will automatically import and cache the models/schemas for all
of its types:
```
up dependency add 'xpkg.upbound.io/upbound/provider-gcp-storage:>=v1.9.0'
```

Now it's time to start writing our composition logic. We'll start by generating
a Python function stub:
```
up function generate storage-function apis/xstoragebuckets/composition.yaml --language=python
```

We now have everything we need to use our native Python tooling to code our
composition logic. To keep moving quickly, feel free to just copy/paste the
below Python code into `main.py`:
```python
from crossplane.function import resource
from crossplane.function.proto.v1 import run_function_pb2 as fnv1

from .model.io.upbound.gcp.storage.bucket import v1beta1 as bucketv1beta1
from .model.io.upbound.gcp.storage.bucketacl import v1beta1 as aclv1beta1
from .model.io.upbound.demoupdevex.xstoragebucket import v1alpha1

def compose(req: fnv1.RunFunctionRequest, rsp: fnv1.RunFunctionResponse):
    observed_xr = v1alpha1.XStorageBucket(**req.observed.composite.resource)
    params = observed_xr.spec.parameters

    desired_bucket = bucketv1beta1.Bucket(
        apiVersion="storage.gcp.upbound.io/v1beta1",
        kind="Bucket",
        spec=bucketv1beta1.Spec(
            forProvider=bucketv1beta1.ForProvider(
                location=params.location,
                versioning=[
                    bucketv1beta1.VersioningItem(
                        enabled=params.versioning,
                    )
                ],
            ),
        ),
    )
    resource.update(rsp.desired.resources["bucket"], desired_bucket)

    if "bucket" not in req.observed.resources:
        return

    observed_bucket = bucketv1beta1.Bucket(**req.observed.resources["bucket"].resource)
    if observed_bucket.metadata is None or observed_bucket.metadata.annotations is None:
        return
    if "crossplane.io/external-name" not in observed_bucket.metadata.annotations:
        return

    bucket_external_name = observed_bucket.metadata.annotations[
        "crossplane.io/external-name"
    ]

    desired_acl = aclv1beta1.BucketACL(
        apiVersion="storage.gcp.upbound.io/v1beta1",
        kind="BucketACL",
        spec=aclv1beta1.Spec(
            forProvider=aclv1beta1.ForProvider(
                bucket=bucket_external_name,
                predefinedAcl=params.acl,
            ),
        ),
    )
    resource.update(rsp.desired.resources["acl"], desired_acl)
```

## Test the Logic

### Testing Locally

We want to ensure the correctness of our Python composition logic. We'll first
use the `up composition render` command to locally test the Python composition
logic we've written. The `render` command works with XRs, so we first need to
create one, which is basically the same as our `example.yaml` claim file, but
using the `XStorageBucket` type and cluster scoped. Save the following as
`xr.yaml`:
```yaml
apiVersion: demoupdevex.upbound.io/v1alpha1
kind: XStorageBucket
metadata:
  name: example
spec:
  parameters:
    location: US
    versioning: true
    acl: publicRead
```

Now run the `render` command to compile our Python logic, start up the function
container and networking, ensure all the Provider dependencies are running, and
then execute the composition function pipeline for the provided input XR:
```
up composition render apis/xstoragebuckets/composition.yaml examples/storagebucket/xr.yaml
```

All of that building, configuring, and orchestration to test locally is done for
us automatically by the `up` tooling, which saves us time and effort.

### Testing on a Real Control Plane

#### Deploy to Upbound Cloud
First, check your `up` profile and make sure you are targeting the right Upbound
account:
```
up profile list
```

Then set the `up` context to an Upbound Cloud Space where you will create a
development control plane, e.g., `upbound-gcp-us-central-1`:
```
up ctx
```

Now you can build, package, push, and deploy our project to an Upbound Cloud
development control plane, all with one easy command:
```
up project run
```

After the project has been successfully deployed to a newly created control
plane, switch the context to this new development control plane:
```
up ctx ./demo-up-devex
```

#### Configure GCP Provider
In order for our control plane to be able to create resources in GCP, we need to
create a GCP credentials secret:
```
kubectl create secret generic \
    gcp-secret \
    -n crossplane-system \
    --from-file=my-gcp-secret=/Users/jared/Desktop/gcp-credentials.json
```

Then reference it with a `ProviderConfig`:
```shell
cat <<EOF | kubectl apply -f -
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: crossplane-playground
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-secret
      key: my-gcp-secret
EOF
```

#### Create real cloud resources in GCP

Let's request an instance of our `StorageBucket` claim, which will in turn
execute our composition logic on the control plane:
```
kubectl apply -f examples/storagebucket/example.yaml
```

We should start seeing cloud resources being created in GCP:
```
up alpha get storagebuckets
```

This CLI view is useful, but we can get a much more rich experience in the
Upbound Cloud Console! We can debug, inspect, and manage our control plane and
resources in a much more user-friendly way. Navigate to your control plane,
e.g., at a URL similar to the following for your organization and control plane:

https://console.upbound.io/jaredorg/spaces/upbound-gcp-us-central-1/groups/default/controlPlanes


## Build, Publish, Install

We can build our project into a reusable package and push it to the Upbound
Marketplace with some very simple commands:
```
up project build
up project push
```

This will result in a new package version being present in the Marketplace, at a
URL similar to the following depending on your organization and project names:

https://marketplace.upbound.io/account/jaredorg/demo-up-devex_storage-function


## Maintain in Production

You can get a feel for managing and maintaining a control plane in production
with Upbound Cloud by revisiting your development control plane and diving into
all the rich resource details, graphs, events, etc.  There's also functionality
for backups and disaster recovery, credential management, and observability
across your control planes! Navigate once again to a URL similar to the below
based on your organization and control plane:

https://console.upbound.io/jaredorg/spaces/upbound-gcp-us-central-1/groups/default/controlPlanes

# Clean-up

To delete all cloud resources as well as your development control plane, run
commands similar to below and ensure no resources are left behind:
```
kubectl delete -f examples/storagebucket/example.yaml
up alpha get managed

up ctx ..
up ctp delete demo-up-devex
up ctp list

cd .. && rm -rf demo-up-devex
```