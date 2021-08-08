
###Notes:
We can automate image build using cloud native build pack
and kpack. Cloud native build pack will take sourcecode
and consistently and reliabily turn it into open container initiative
compliatn container images. It has 3 parts:
- Pack cli tool & Go library
- kpack with cloud native build pack native life cycle.Life
  cycle is orchestratio logic of how container images are built.
- Build packs- proideruntimesupport for applications and modular way
  for adding content and behaviour into image
  We can do manual build using spring-boot-build-image:
```
mvnw spring-boot:build-image -DskipTests
```  
Here cloud naive build pack is being used,particular packeto buildpack.
Packeto is one version of buildpack, heroku and google have their own.
Buildpack abstracts building application.
Builder is the image that will build the application image.
Run image is the base image that provides environment
for our application image.Download pack cli from
https://buildpacks.io/docs/tools/pack/.Inspect the image create
with pack command as below:
```
pack inspect cats-service:0.0.1-SNAPSHOT
```
We can see below buildpacks were used to build our image:
```
Buildpacks:
  ID                                         VERSION        HOMEPAGE
  paketo-buildpacks/ca-certificates          2.3.2          https://github.com/paketo-buildpacks/ca-certificates
  paketo-buildpacks/bellsoft-liberica        8.2.0          https://github.com/paketo-buildpacks/bellsoft-liberica
  paketo-buildpacks/executable-jar           5.1.2          https://github.com/paketo-buildpacks/executable-jar
  paketo-buildpacks/dist-zip                 4.1.2          https://github.com/paketo-buildpacks/dist-zip
  paketo-buildpacks/spring-boot              4.4.2          https://github.com/paketo-buildpacks/spring-boot
```
We need to recreate this on kubernetes cluster and
can be done with kpack.Kpack is a k8s native build service that
build container images in k8s using cloud native build packs.
It takes source code from say github and build image and
upload it into registry of our choice.Now we need to create
stand alone resource for store and stack.Stack is what provides
the operating system layer.We can see the suggested stack by
the community with below command:
```
pack stack suggest
```
Output example:
```
Stacks maintained by the community:

    Stack ID: io.buildpacks.stacks.bionic
    Description: A minimal Paketo stack based on Ubuntu 18.04
    Maintainer: Paketo Project
    Build Image: paketobuildpacks/build:base-cnb
    Run Image: paketobuildpacks/run:base-cnb
```
Store yaml let our builder know what build packs are available to
build images.The build packs earlier seen by inpsect in below
build packs mentioned with store yaml:
```
apiVersion: kpack.io/v1alpha1
kind: ClusterStore
metadata:
  name: default
spec:
  sources:
    - image: gcr.io/paketo-buildpacks/java
    - image: gcr.io/paketo-buildpacks/nodejs
```
Here nodejs is included if we have other images to be built
in our environment which need node js. Thus we are creating a builder
that can be commonly used across our k8s cluster.Our builder
is going to use the store and stack we defined and is going
to add registry coordinates for the builder image and detection
 order. This is going to make it's own image(builder image).
This is going to be used each time an image is going to be built.
Service account is the way we are passing credentials to 
k8s to enable it to push images to container registry.
Refer this for service account creation:
```
https://blog.container-solutions.com/using-google-container-registry-with-kubernetes
```
The detection order has been specified under order section in
builder yaml. From this the build pack specified is checked to see
if it is compatible with source code under consideration. For
a matter of efficiency when we have more java projects than nodejs
apps, we can specify java build pack first in the order section.
Inorder to install kpack  in k8s cluster run the release yaml file
from kpack's official github repo. After this apply
the stack and store yaml files and builder yaml file. Once
builder is applied, the builder image can be seen in our registry chosen.
The service account will allow this push to happen.Use storage admin role or any
role that alllows image push and pull.Create secret to be mapped
for service account as below:
```
 kubectl create secret docker-registry regcred -n kpack --docker-server=gcr.io  --docker-username=_json_key  --docker-password="$(cat ./<json with service account key>.json)"  --docker-email=<emailid>
```
The builder of kind Builder is namespace scoped.If we want builder
to be namespace independent we have to use ClusterBuilder kind.
In the kpack image yaml file we are telling kpack to monitor
the git repo for any change or any change in yaml files for stack or store.
If this happens kpack will use the builder to build image.
On applying this immediately also an image build will take place.
Open container initiative has a meta data layer that points to other
layers.Rebasing is the operation by which only the layer that 
has changed will be updated ,instead of running all layer above it
as in docker daemon(Thus loosing all cached layers). Thus build pack is more
efficient.

We will follow ArgoCD methodolgy from Gitops
 for continous delivery . The core idea of gitops
is to have a repostory with decalrative description of the
desired infrastructure in production and an automated process to make sure that
the described state is matched.