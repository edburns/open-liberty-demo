# Deploy Open Liberty Application on Azure Red Hat OpenShift
This guide demonstrates how to run your Java EE application on the Open Liberty runtime and then deploy the containerized application to an Azure Red Hat OpenShift (ARO) cluster using the Open Liberty Operator. This article will walk you through preparing Open Liberty Application, building the application image and running the containerized application on ARO.
- [Open Liberty](https://openliberty.io/): Open Liberty is an IBM Open Source project which implements the Eclipse MicroProfile specifications and is also Jakarta EE compatible. Open Liberty is fast to start up with a low memory footprint and live reload for quick iterative development. It is simple to add and remove features from the latest versions of MicroProfile and Jakarta EE. Zero migration lets you focus on what's important, not the APIs changing under you. 
- [Azure Red Hat OpenShift](https://azure.microsoft.com/en-us/services/openshift/): Azure Red Hat OpenShift provides a flexible, self-service deployment of fully managed OpenShift clusters. Maintain regulatory compliance and focus on your application development, while your master, infrastructure, and application nodes are patched, updated, and monitored by both Microsoft and Red Hat.

## Prerequisites
- Install JDK per your needs (e.g., [AdoptOpenJDK OpenJDK 8 LTS/OpenJ9](https://adoptopenjdk.net/?variant=openjdk8&jvmVariant=openj9))
- Install [Maven](https://maven.apache.org/download.cgi)
- Install [Docker](https://docs.docker.com/get-docker/) for your OS
- Register a [Docker Hub](https://id.docker.com/) account
- Install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
- Register an Azure subscription. If you don't have one, you can get one for free for one year [here](https://azure.microsoft.com/en-us/free)
- Clone this repo to your local directory

## Set up Azure Red Hat OpenShift cluster

Follow the instructions in these two tutorials and then return here to continue.

1. Create the cluster by following the steps in [Create an Azure Red Hat OpenShift 4 cluster](https://docs.microsoft.com/en-us/azure/openshift/tutorial-create-cluster).
2. Connect to the cluster by following the steps in [Connect to an Azure Red Hat OpenShift 4 cluster](https://docs.microsoft.com/en-us/azure/openshift/tutorial-connect-cluster).

After creating and connecting to the cluster, install the Open Liberty Operator.

1. Log in to the OpenShift web console from your browser.
2. Navigate to "Operators > OperatorHub" and search for "Open Liberty Operator"
3. Select "Open Liberty Operator" from the search results.
4. Select "Install".
   ![install-operator](pictures/install-operator.png)
5. Select "Subscribe" and wait until "Open Liberty Operator" is listed as one of Installed Operators.

## Prepare your Open Liberty application

Open Liberty is a Java EE 8/Jakarta EE 8 full profile compatible app server.  If you already have a Java EE application running on an existing app server (for example, IBM WebSphere Network Deployment, Oracle WebLogic Server, JBoss EAP, etc.), only minimal changes are necessary to make it run on Open Liberty.

### Get a Quick Start with A basic Java EE App

Navigate to [`<path-to-repo>/javaee-cafe/1-start`](https://github.com/majguo/open-liberty-demo/tree/master/javaee-cafe/1-start) of your local clone to see the sample app.  It uses Maven and Java EE 8 (JAX-RS, EJB, CDI, JSON-B, JSF, Bean Validation). Here is the project structure:

```bash
├── pom.xml                                         # Maven POM file
└── src
    ├── main
    │   ├── java
    │   │   └── cafe
    │   │       ├── model
    │   │       │   ├── CafeRepository.java         # Cafe CRUD repository (in-memory)
    │   │       │   └── entity
    │   │       │       └── Coffee.java             # Coffee entity
    │   │       └── web
    │   │           ├── rest
    │   │           │   └── CafeResource.java       # Cafe CRUD REST APIs
    │   │           └── view
    │   │               └── Cafe.java               # Cafe bean in JSF client
    │   ├── resources
    │   │   ├── META-INF
    │   │   └── cafe
    │   │       └── web
    │   │           ├── messages.properties         # Resource bundle in EN
    │   │           └── messages_es.properties      # Resource bundle in ES
    │   └── webapp
    │       ├── WEB-INF
    │       │   ├── faces-config.xml                # JSF configuration file specifying resource bundles and supported locales
    │       │   └── web.xml                         # Deployment descriptor for a Servlet-based Java web application
    │       └── index.xhtml                         # Home page of JSF client
    └── test
        └── java                                    # Placeholder for tests
```

1. Run `mvn clean package`.  This will generate a war package `javaee-cafe.war` in the directory `./target`.

### Prepare app to run on Open Liberty server

We will prepare the app to run on Open Liberty by adding the mandatory `server.xml` file with the required settings.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<server description="defaultServer">
    <!-- Enable features -->
    <featureManager>
        <feature>cdi-2.0</feature>
        <feature>jaxb-2.2</feature>
        <feature>jsf-2.3</feature>
        <feature>jaxrs-2.1</feature>
        <feature>ejbLite-3.2</feature>
    </featureManager>

    <!-- Define http & https endpoints -->
    <httpEndpoint id="defaultHttpEndpoint" host="*" httpPort="9080" httpsPort="9443" />

    <!-- Automatically expand WAR files and EAR files -->
    <applicationManager autoExpand="true" />

    <!-- Define web application with its context root and location -->
    <webApplication id="javaee-cafe" contextRoot="/" location="${server.config.dir}/apps/javaee-cafe.war">
    </webApplication>
</server>
```

Add this configuration file to `<path-to-repo>/javaee-cafe/1-start/src/main/liberty/config`. The [liberty-maven-plugin](https://github.com/OpenLiberty/ci.maven#liberty-maven-plugin) looks in this directory when packaging the application for deployment.

The `liberty-maven-plugin` provides a number of goals for managing an Open Liberty server and applications.  We will use dev mode to get a look at the sample application running locally.

1. Run `mvn clean liberty:dev` in a console.
2. Wait until the server starts. You will output similar to the following in your console.

```bash
[INFO] Listening for transport dt_socket at address: 7777
[INFO] Launching defaultServer (Open Liberty 20.0.0.6/wlp-1.0.41.cl200620200528-0414) on Java HotSpot(TM) 64-Bit Server VM, version 1.8.0_251-b08 (en_US)
[INFO] [AUDIT   ] CWWKE0001I: The server defaultServer has been launched.
[INFO] [AUDIT   ] CWWKG0093A: Processing configuration drop-ins resource: 
[INFO]   Property location will be set to ${server.config.dir}/apps/javaee-cafe.war.
[INFO] 
[INFO] [AUDIT   ] CWWKZ0058I: Monitoring dropins for applications.
[INFO] [AUDIT   ] CWWKT0016I: Web application available (default_host): http://localhost:9080/
[INFO] [AUDIT   ] CWWKZ0001I: Application javaee-cafe started in 3.453 seconds.
[INFO] [AUDIT   ] CWWKF0012I: The server installed the following features: [cdi-2.0, ejbLite-3.2, el-3.0, jaxb-2.2, jaxrs-2.1, jaxrsClient-2.1, jndi-1.0, jsf-2.3, jsonp-1.1, jsp-2.3, servlet-4.0].
[INFO] [AUDIT   ] CWWKF0011I: The defaultServer server is ready to run a smarter planet. The defaultServer server started in 6.447 seconds.
[INFO] CWWKM2015I: Match number: 1 is [6/10/20 10:26:09:517 CST] 00000022 com.ibm.ws.kernel.feature.internal.FeatureManager            A CWWKF0011I: The 
defaultServer server is ready to run a smarter planet. The defaultServer server started in 6.447 seconds..
[INFO] Press the Enter key to run tests on demand. To stop the server and quit dev mode, use Ctrl-C or type 'q' and press the Enter key.
[INFO] Source compilation was successful.
```

Open [http://localhost:9080/](http://localhost:9080/) in your browser to visit the application home page.

The application will look similar to the following.

   ![javaee-cafe-web-ui](pictures/javaee-cafe-web-ui.png)

For reference, these have already been applied in [`<path-to-repo>/javaee-cafe/2-simple`](https://github.com/majguo/open-liberty-demo/tree/master/javaee-cafe/2-simple).

## Deploy application on ARO cluster

To deploy and run your Open Liberty Application on Azure Red Hat OpenShift cluster, you need to containerize your app as a Docker image using [Open Liberty container images](https://github.com/OpenLiberty/ci.docker).

### Build application image

Here is the [Dockerfile](https://github.com/majguo/open-liberty-demo/tree/master/javaee-cafe/2-simple/Dockerfile) for building the application image:

```Dockerfile
# open liberty base image
FROM openliberty/open-liberty:kernel-java8-openj9-ubi

# Add config and app
COPY --chown=1001:0 src/main/liberty/config/server.xml /config/server.xml
COPY --chown=1001:0 target/javaee-cafe.war /config/apps/

# This script will add the requested XML snippets, grow image to be fit-for-purpose and apply interim fixes
RUN configure.sh
```

1. cd to [`<path-to-repo>/javaee-cafe/2-simple`](https://github.com/majguo/open-liberty-demo/tree/master/javaee-cafe/2-simple).
1. Run the following commands to build application image and push to your Docker Hub repository.

```bash
# Build project and generate war package
mvn clean package

# Build and tag application image
docker build -t javaee-cafe-simple --pull .

# Create a new tag with your Docker Hub account info that refers to source image
# Note: replace "${Your_DockerHub_Account}" with your valid Docker Hub account name
docker tag javaee-cafe-simple docker.io/${Your_DockerHub_Account}/javaee-cafe-simple

# Login to Docker Hub
docker login

# Push image to your Docker Hub repositories
# Note: replace "${Your_DockerHub_Account}" with your valid Docker Hub account name
docker push docker.io/${Your_DockerHub_Account}/javaee-cafe-simple
```

### Prepare OpenLibertyApplication yaml file

Because we use the "Open Liberty Operator" to manage Open Liberty applications, we just need to create an instance of its *Custom Resource Definition* "OpenLibertyApplication". The Operator will then take care all the aspects to manage built-in OpenShift resources required for deployment.

Here is the resource definition of the [Open Liberty Application](https://github.com/majguo/open-liberty-demo/tree/master/javaee-cafe/2-simple/openlibertyapplication.yaml) used in the guide:

```yaml
apiVersion: openliberty.io/v1beta1
kind: OpenLibertyApplication
metadata:
  name: javaee-cafe-simple
  namespace: open-liberty-demo
spec:
  replicas: 1
  # Note: replace "${Your_DockerHub_Account}" with your valid Docker Hub account name
  applicationImage: docker.io/${Your_DockerHub_Account}/javaee-cafe-simple:latest
  expose: true
```

Now we can deploy the sample Open Liberty Application to the Azure Red Hat OpenShift cluster [you created earlier in the article](#set-up-azure-red-hat-openshift-cluster).

### Deploy from GUI

1. Login into OpenShift web console from your browser.
2. Navigate to "Administration > Namespaces > Create Namespace".
   ![create-namespace](pictures/create-namespace.png)
3. For `Name > Create`,  Fill in "open-liberty-demo".
4. Navigate to "Operators > Installed Operators > Open Liberty Operator > Open Liberty Application > Create OpenLibertyApplication"
   ![create-openlibertyapplication](pictures/create-openlibertyapplication.png)
5. Copy and paste the contents of above yaml file or start from the default yaml auto-genreated, the final yaml should look like the following.

    ```yaml
      apiVersion: openliberty.io/v1beta1
      kind: OpenLibertyApplication
      metadata:
        name: javaee-cafe-simple
        namespace: open-liberty-demo
      spec:
        replicas: 1
        # Note: replace "${Your_DockerHub_Account}" with your valid Docker Hub account name
        applicationImage: docker.io/${Your_DockerHub_Account}/javaee-cafe-simple
        expose: true
    ```

6. Select "Create".
7. Navigate to "javaee-cafe-simple > Resources > javaee-cafe-simple (Route) > Click link below Location".
8. You will see the same application home page opened in the browser, which was mentioned before

### Deploy from CLI

You can also login to the OpenShift cluster via CLI with a token retrieved from its web console:

1. At right-top of web console, expand the context menu of the logged-in user (e.g. "kube:admin"), then select "Copy Login Command".
2. Log in into the new tab window if required.
3. Click "Display Token" > Copy value listed below "Log in with this token" > Paste and run the copied command in a console  
4. Change directory to [`<path-to-repo>/javaee-cafe/2-simple`](https://github.com/majguo/open-liberty-demo/tree/master/javaee-cafe/2-simple), and run the following commands to deploy your Open Liberty Application to ARO cluster.

```bash
# Create new namespace where resources of demo app will belong to
oc new-project open-liberty-demo

# Create an ENV variable which will subsitute the one defined in openlibertyapplication.yaml
# Note: replace "<Your_DockerHub_Account>" with your valid Docker Hub account name
export Your_DockerHub_Account=<Your_DockerHub_Account>

# Subsitute "Your_DockerHub_Account" in openlibertyapplication.yaml and then create resource
envsubst < openlibertyapplication.yaml | oc apply -f -

# Check if OpenLibertyApplication instance created
oc get openlibertyapplication -n open-liberty-demo

# Check if route created by Operator
oc get route -n open-liberty-demo
```

Once the route of the Open Liberty Application is created, you can visit the application home page by opening the URL copied from value of its "HOST/PORT" in your browser.

## Integrate with Azure services
### Enable Signle-Sign-On with Azure Active Directory OpenID Connect

### Persist data with Azure Database for PostgreSQL

### Ship application log to managed Elasticsearch service on Azure

## References
- [Open Liberty](https://openliberty.io/)
- [Azure Red Hat OpenShift](https://azure.microsoft.com/en-us/services/openshift/)
- [Open Liberty Operator](https://github.com/OpenLiberty/open-liberty-operator)
- [Open Liberty Server Configuration](https://openliberty.io/docs/ref/config/)
- [Liberty Maven Plugin](https://github.com/OpenLiberty/ci.maven#liberty-maven-plugin)
- [Open Liberty Container Images](https://github.com/OpenLiberty/ci.docker)