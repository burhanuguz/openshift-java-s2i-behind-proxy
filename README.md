# Openshift 4.5 s2i behind Corporate Proxy (JBoss EAP 7.2 from Developer Catalog)

## Prerequisites
- Openshift Cluster behind proxy
- Internal Image registry (to access the images you built in cluster)

## Overview
When starting JAVA image build from **Developer Catalog (e.g JBoss EAP 7.2)** you will probably face with error from maven build that says there are **no secure connection to repo**. 
- To solve this problem you can make **your own builder image** putting your company's proxy certificate in it, but in the end you will not get to use **Red Hat's official images** and updates to those images. You will have to update your own images always . 

### Initial Script to trust company's CA certificates
In this explanation you will have chance to use **Red Hat's own images**. These images have **s2i scripts** in it. **Red Hat gives option to change the s2i scripts**. 
- The idea is to put an initial script first to **inject those company or other certificates to image** before anything actually starts. With that JVM will securely connect through proxy to maven repos and building operation will be success.

> In this explanation JBoss EAP 7.2 from Developer Catalog will be used.
 
 In this example build will be done in **example project**(namespace).
 - Switch to **Developer Tab** and click to **From Catalog**
 
![https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/1.png](https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/1.png)
 - Select **JBoss EAP 7.2** and then **Instantiate Template**
 
![https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/2.png](https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/2.png)
![https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/3.png](https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/3.png)
 - Scroll down and put **-Djavax.net.ssl.trustStore=/tmp/cacerts** argument to **Maven Additional Arguments** part and click **Create**.
 
![https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/4.png](https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/4.png)
``` java
It should be like that 
-Dcom.redhat.xpaas.repo.jbossorg  -Djavax.net.ssl.trustStore=/tmp/cacerts
```
- The first build will start but it will fail because of proxy. 
- In the building stage image will call for the script in this path **/usr/local/s2i/assemble**. 
  - What will we do is creating our own script and name it as **assemble**
  - In this script **cacerts** file that JVM's trust will be copied to **/tmp** path.
  - **New CA certs will be placed in this new cacerts file** and with **-Djavax.net.ssl.trustStore=/tmp/cacerts** command JVM's will trust our own CA's.
- The **assemble** script will look like this

```bash
#!/bin/sh

cat <<EOF > /tmp/cert1.crt
-----BEGIN CERTIFICATE-----
................................................................
................................................................
-----END CERTIFICATE-----
EOF

cat <<EOF > /tmp/cert2.crt
-----BEGIN CERTIFICATE-----
................................................................
................................................................
-----END CERTIFICATE-----
EOF

cp $JAVA_HOME/jre/lib/security/cacerts /tmp/cacerts

chmod +w /tmp/cacerts

keytool -importcert -noprompt -storepass changeit -file /tmp/cert1.crt -alias cert1 -keystore /tmp/cacerts
keytool -importcert -noprompt -storepass changeit -file /tmp/cert2.crt -alias cert2 -keystore /tmp/cacerts

/usr/local/s2i/assemble
 
```
- The newly created **cacerts** file will have our **CA certs** in it.
- **The assemble script** should be put somewhere reachable from your Cluster. In this example **the assemble script** can be accessed via URL. Here is the official link for [s2i scripts and accessing them](https://docs.openshift.com/container-platform/4.5/builds/build-strategies.html#images-create-s2i-scripts_build-strategies)
- To put the link select **Build tab** and click the **build** that created.
 
![https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/5.png](https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/5.png)
- Select **YAML tab** and put the link of the directory contains the script to **spec.strategy.sourceStrategy.scripts** part.
 
![https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/6.png](https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/6.png)
- After all those steps click **Start Build** and in the opened page select **Logs** and watch. You will see that maven won't give error and build process will be finished with success after that.
 
![https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/7.png](https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/7.png)
![https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/8.png](https://github.com/burhanuguz/openshift-java-s2i-behind-proxy/blob/master/pictures/8.png)


Using this method you get to use Red Hat's updated and supported image always. There is no need create a custom image for certificate and updating it always.

---
##### Resources
1. [Build Strategies](https://docs.openshift.com/container-platform/4.5/builds/build-strategies.html)
