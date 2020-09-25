# Openshift s2i behind Corporate Proxy (JBoss EAP 7.2 from Developer Catalog)

## Prerequisites
- Openshift Cluster behind proxy
- Internal Image registry (to access the images you built in cluster)

## Overview
When starting JAVA image build from **Developer Catalog (e.g JBoss EAP 7.2)** you will probably face with error from maven build that says there are **no secure connection to repo**. 

- To solve this problem a custom image will be built. 
- Whenever Red Hat's builder image is updated, build will be triggered. So the created builder image will always be updated  with **Red Hat's latest image**.

### Creating Build Config
1. First thing is to set certificates.
```bash
cat <<EOF > /tmp/cert1.crt
# First CA to be trusted
-----BEGIN CERTIFICATE-----
................................................................
................................................................
-----END CERTIFICATE-----
EOF

cat <<EOF > /tmp/cert1.crt
# Second CA to be trusted
-----BEGIN CERTIFICATE-----
................................................................
................................................................
-----END CERTIFICATE-----
EOF

cat <<EOF > /tmp/cert1.crt
# Third CA to be trusted
-----BEGIN CERTIFICATE-----
................................................................
................................................................
-----END CERTIFICATE-----
EOF
```
2. Create a **Config Map** that contains this certificate chain.
```bash
oc create configmap user-ca-bundle --from-file=/tmp/cert1.crt --from-file=/tmp/cert2.crt --from-file=/tmp/cert3.crt
```
3. Create the Dockerfile.
```bash
cat <<EOF > Dockerfile
FROM
# Switch to root for package installs and copying files
USER 0
# Import corporation self signed certificate
COPY ./*.crt /etc/pki/ca-trust/source/anchors/
RUN keytool -importcert -noprompt -storepass changeit -keystore \$JAVA_HOME/jre/lib/security/cacerts -file /etc/pki/ca-trust/source/anchors/cert1.crt -alias cert1 && \\
    keytool -importcert -noprompt -storepass changeit -keystore \$JAVA_HOME/jre/lib/security/cacerts -file /etc/pki/ca-trust/source/anchors/cert2.crt -alias cert2 && \\
    keytool -importcert -noprompt -storepass changeit -keystore \$JAVA_HOME/jre/lib/security/cacerts -file /etc/pki/ca-trust/source/anchors/cert3.crt -alias cert3 && \\
    update-ca-trust
# Run container by default as user with id 1001 (default)
USER 1001
EOF
```

4. Create **Build Config**. This **Build Config** will use **dockerstrategy** and newly created **Dockerfile** to build.

```bash
cat Dockerfile | oc new-build \
   --name jboss-eap72-openshift-12 \
   --strategy docker \
   --image-stream jboss-eap72-openshift:1.2 \
   --to jboss-eap72-openshift:1.2-build \
   --build-config-map user-ca-bundle:. \
   --dockerfile -
```
5. Once **Build Config** is created, first build will be triggered and first image will be pushed to registry.

6. Builder Image should be annotated, otherwise it will not be available in **Developer Catalog** to use. 

7. Because ImageStream can not be annotated ImageStreamTag should be created as follows.

```bash
oc tag --alias=true openshift/jboss-eap72-openshift:1.2-build openshift/jboss-eap72-openshift:1.2-custom
````
 8. ImageStreamTag should be annotate as follows,
```bash
oc annotate imagestreamtags.image.openshift.io jboss-eap72-openshift:1.2-custom \
supports='eap:7.2,javaee:7,java:8' \
sampleContextDir='kitchensink' \
openshift.io/display-name='Red Hat JBoss EAP 7.2' \
sampleRef='openshift' \
version='1.2' \
tags='builder,eap,javaee,java,jboss,hidden' \
sampleRepo='https://github.com/jbossas/eap-quickstarts/openshift' \
description='Red Hat JBoss EAP 7.2 S2I Image' \
iconClass='icon-eap'
```
- These annotations are extracted from original dotnet:2.1 ImageStreamTag.
```bash
oc get imagestreamtags.image.openshift.io -n openshift jboss-eap72-openshift:1.2 -o json | jq '.metadata.annotations'
```
- The result is below
```bash
{
  "description": "Red Hat JBoss EAP 7.2 S2I Image",
  "iconClass": "icon-eap",
  "openshift.io/display-name": "Red Hat JBoss EAP 7.2",
  "sampleContextDir": "kitchensink",
  "sampleRef": "openshift",
  "sampleRepo": "https://github.com/jbossas/eap-quickstarts/openshift",
  "supports": "eap:7.2,javaee:7,java:8",
  "tags": "builder,eap,javaee,java,jboss,hidden",
  "version": "1.2"
}
```
![1](https://user-images.githubusercontent.com/59168275/94281082-da1faa00-ff56-11ea-8a77-412ca48634b0.png)
9. It is not a Builder Image, so it is needed to create a new Template and include the newly created image in that Template.
