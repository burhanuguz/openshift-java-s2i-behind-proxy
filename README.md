# Openshift s2i behind Corporate Proxy (.NET Core from Developer Catalog)

## Prerequisites
- Openshift Cluster behind proxy
- Internal Image registry (to access the images you built in cluster)

## Overview
When behind proxy, **.NET Core** build from **Developer Catalog** will give error that nuget packages can not download because there are **no secure connection to repo**. 

- To solve this problem a custom image will be built. 
- Whenever Red Hat's builder image is updated, build will be triggered. So the created builder image will always be updated  with **Red Hat's latest image**.
### Creating Build Config
1. First thing is to set certificates.
```bash
cat <<EOF > /tmp/ca-bundle.crt
# First CA to be trusted
-----BEGIN CERTIFICATE-----
................................................................
................................................................
-----END CERTIFICATE-----

# Second CA to be trusted
-----BEGIN CERTIFICATE-----
................................................................
................................................................
-----END CERTIFICATE-----

# Third CA to be trusted
-----BEGIN CERTIFICATE-----
................................................................
................................................................
-----END CERTIFICATE-----
EOF
```
2. Create a **Config Map** that contains this certificate chain.
```bash
oc create configmap user-ca-bundle --from-file=ca-bundle.crt=/tmp/ca-bundle.crt
```
3. Create the Dockerfile.
```bash
cat <<EOF > Dockerfile
FROM
# Switch to root for package installs and copying files
USER 0
# Import corporation self signed certificate
COPY ./ca-bundle.crt /etc/pki/ca-trust/source/anchors/
RUN update-ca-trust
# Run container by default as user with id 1001 (default)
USER 1001
EOF
```
4. Create **Build Config**. This **Build Config** will use **dockerstrategy** and newly created **Dockerfile** to build.

```bash
cat Dockerfile | oc new-build \
   --name dotnet-21 \
   --strategy docker \
   --image-stream dotnet:2.1 \
   --to dotnet:2.1-build \
   --build-config-map user-ca-bundle:. \
   --dockerfile -
```
5. Once **Build Config** is created, first build will be triggered and first image will be pushed to registry.
6. Builder Image should be annotated, otherwise it will not be available in **Developer Catalog** to use. 
7. Because ImageStream can not be annotated ImageStreamTag should be created as follows.
```bash
oc tag --alias=true openshift/dotnet:2.1-build openshift/dotnet:2.1-custom
```
 8. ImageStreamTag should be annotate as follows,
```bash
oc annotate imagestreamtags.image.openshift.io dotnet:2.1-custom \
supports="dotnet:2.1,dotnet" \
sampleContextDir="app" \
openshift.io/display-name=".NET Core 2.1" \
sampleRef="dotnetcore-2.1" \
version="2.1" \
tags="builder,.net,dotnet,dotnetcore,rh-dotnet21" \
sampleRepo="https://github.com/redhat-developer/s2i-dotnetcore-ex.git" \
description=\
"Build and run .NET Core 2.1 applications on RHEL 7. For more information \
about using this builder image, including OpenShift considerations, see \
https://github.com/redhat-developer/s2i-dotnetcore/tree/master/2.1/build/README.md." \
iconClass="icon-dotnet"
```
- These annotations are extracted from original dotnet:2.1 ImageStreamTag.
```bash
oc get imagestreamtags.image.openshift.io -n openshift dotnet:2.1 -o json | jq '.metadata.annotations'
```
- The result is below
```bash
{
  "description": "Build and run .NET Core 2.1 applications on RHEL 7. For more information about using this builder image, including OpenShift considerations, see https://github.com/redhat-developer/s2i-dotnetcore/tree/master/2.1/bui
  "iconClass": "icon-dotnet",
  "openshift.io/display-name": ".NET Core 2.1",
  "sampleContextDir": "app",
  "sampleRef": "dotnetcore-2.1",
  "sampleRepo": "https://github.com/redhat-developer/s2i-dotnetcore-ex.git",
  "supports": "dotnet:2.1,dotnet",
  "tags": "builder,.net,dotnet,dotnetcore,rh-dotnet21",
  "version": "2.1"
}
```
![1](https://user-images.githubusercontent.com/59168275/94267939-16e2a580-ff45-11ea-9dc8-77e4efabcebd.png)
8. After that go to Developer Tab and check if there is the custom image is there and can be used like below.
![2](https://user-images.githubusercontent.com/59168275/94268981-ab99d300-ff46-11ea-8eff-f01aa620464c.png)