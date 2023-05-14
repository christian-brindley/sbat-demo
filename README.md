# Quick start for deploying ForgeRock SBAT in minikube

## Introduction

This repository contains a sample deployment pattern for the [ForgeRock Secure Banking Access Toolkit](https://github.com/SecureBankingAccessToolkit), using minikube as the example kubernetes platform.

The sample uses docker images built and stored locally in the minikube docker repository.

A high level view of the deployment steps is as follows

- Prepare environment
- Configure ForgeRock Platform
- Deploy IG with SBAT configuration
- Deploy Spring config server
- Deploy Remote Consent Service
- Deploy Mock Resource Server along with MongoDB
- Deploy UI apps (Swagger and Remote Consent)
- Test

Please note that this repository is for reference only, and is work in progress. The samples and documentation here will be absorbed into the main SBAT repository, at which time this repository will be retired.

## Preparation

### Repos

Clone the following repos into your base directory for this deployment. For the purposes of these instructions, we will use the base directory `~/sbat`.

```
mkdir ~/sbat
cd ~/sbat
git clone https://github.com/christian-brindley/sbat-demo
git clone https://github.com/forgerock/forgeops
git clone https://github.com/SecureApiGateway/secure-api-gateway-ob-uk
git clone https://github.com/SecureApiGateway/secure-api-gateway-ob-uk-rcs
git clone https://github.com/SecureApiGateway/secure-api-gateway-ob-uk-spring-config-server
git clone https://github.com/SecureApiGateway/secure-api-gateway-ob-uk-rs
git clone https://github.com/SecureApiGateway/secure-api-gateway-ob-uk-ui
```

### kubernetes

The following instructions are specific to minikube, but can be adapted for other environments such as Docker Desktop kubernetes.

If you are using minikube, make sure you are using the local minikube docker daemon

`eval $(minikube docker-env)`

The ingress charts included with this demo repo assume that nginx is deployed in your k8s environment. If you don't already have nginx, you can deploy it via `minikube` as follows:

```
minikube addons enable ingress
```

Alternatively, you can install nginx directly from the public repo:

```
helm install --namespace kube-system nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx
```

### DNS

The sample configuration uses the domain `sbat.demo` for test purposes. This requires host entries for your minikube ip - e.g.

```
$ minikube ip
192.168.133.133
```

This would require the following entry in /etc/hosts

```
192.168.133.133 tpp.sbat.demo consent.sbat.demo swagger.sbat.demo rcs.sbat.demo
```

## Self signed TLS certificate

You'll need to generate a self signed TLS certificate for the `sbat.demo` domain. A sample openssl config file is provided in this repo - e.g. under the `certs/secrets` directory:

```
openssl req -x509 -newkey rsa:4096 -keyout sbat-demo.key -out sbat-demo.crt -sha256 -days 365 -subj '/CN=sbat.demo' -nodes  -config ../conf/openssl.cnf
```

This needs to be trusted by your workstation browser to avoid TLS issues during testing.

## Adjust config for your environment

### Platform configuration

Create a `configmap.yaml` from the sample in this repo:

```
cd ~/sbat/sbat-demo
cd configmap
cp configmap.yaml.sample configmap.yaml
```

Edit the configmap.yaml for your platform configuration, or keep the defaults for testing purposes.

### IG secrets

Create a `secrets.yaml` file from the sample provided in this repo:

```
cd ~/sbat/sbat-demo/kustomize/sbat
cp secrets.yaml.sample secrets.yaml
```

Edit `secrets.yaml` with your own deployment secrets, or keep the defaults for testing purposes.

## Namespace configuration

### Create the namespace

Create a dedicated namespace for sbat

```
$ kubectl create namespace sbat
$ kubens sbat
```

### TLS secrets

Load the server certificate you created previously as a TLS secret in your namespace - e.g. under the `certs/secrets` directory

```
cd ~/sbat/sbat-demo/certs/secrets
kubectl create secret tls server.tls --key=sbat-demo.key --cert=sbat-demo.crt
```

Load the test trusted CA certs from the `certs/conf` directory

```
cd ~/sbat/sbat-demo/certs/conf
kubectl create secret generic obri-ca --from-file=ob-ca.crt=ob-ca.crt
```

### SBAT config map

Load the SBAT configmap into the namespace

```
cd ~/sbat/sbat-demo/configmap
kubectl apply -f configmap.yaml
```

## Platform configuration

SBAT requires specific configuration of the ForgeRock platform to support the Open Banking use case. This configuration consists of the following main areas:

- Access Management
  - Base URL Source
  - Authentication Journeys
  - OAuth2 Provider Settings
  - Policy Configuration
  - Service accounts
  - System OAuth2 Clients
- Identity Management
  - Open Banking Managed Objects

There are various ways to configure these settings, including

- Manually via the platform administration portal
- Step by step using the [sample Postman collections](postman)
- Automatically via the [SBAT initialiser task](https://github.com/SecureBankingAccessToolkit/securebanking-openbanking-uk-fidc-initializer)

## Identity Gateway

### Build IG config

Build the required custom IG extensions, and then run the config builder script from the SBAT repo.

```
cd ~/sbat/secure-api-gateway-ob-uk
mvn clean install
bin/config.sh -e dev -igm development init
```

Note: if you cannot run the mvn build for any reason, you can copy the version of the target jar file from this repository into the `config/7.1.0/securebanking/ig/lib` before running the config.sh script.

This creates a full IG docker image directory within the repo under `docker/7.1.0/ig`.

### Customise local routing

In order to make things easier in a dev environment, the demo deployment uses kubernetes internal DNS to route from IG to the mock resource server.

To enable this, we use the internal service name for the RS URL (`securebanking-openbanking-uk-rs`) and disable TLS. Disabling TLS requires an update to each of the IG routes that sit in front of the RS, from

`"baseURI" : "https://&{rs.fqdn}"`

to

`"baseURI" : "http://&{rs.fqdn}"`

We are also going to use an internal route for IG to talk to itself for test JWK services.

Finally, we are going to parameterise the IDM object names so that we can use custom versions for a shared environment.

The routes sit in the directory built in the previous step - e.g. (sed example has a blank paramter so that it works on MacOS)

```
cd ~/sbat/sbat/securebanking-openbanking-uk-gateway/docker/7.1.0/ig/config/routes
sed -i '' 's/https:\/\/\&{rs\.fqdn}/http:\/\/\&{rs\.fqdn}:8080/g' *
sed -i '' 's/https:\/\/\&{ig\.fqdn}/http:\/\/\localhost:8080/g' 03-ob-dcr-register-tpp.json
sed -i '' 's/"apiClient"/"\&{apiclient.object}"/g' *
sed -i '' 's/"apiClientOrg"/"\&{apiclientorg.object}"/g' *
sed -i '' 's/"accountAccessIntent"/"\&{accountaccessintent.object}"/g' *
```

Update RepoConsent.groovy with your own managed object names/

### Clone the forgeops repository

The ForgeRock forgeops repository has a sample deployment of Identity Gateway which may be used to deploy the SBAT IG instance.

### Build the IG image

Follow the forgeops documentation to build a custom IG image with the SBAT configuration and deploy it into the namespace. During the build step, copy the contents of the config directory you built previously in the `securebanking-openbanking-uk-gateway` repo.

Note that there is an additional step to add an instruction to the standard forgeops Dockerfile for IG: this ensures that the additional libraries are in the standard IG classpath.

```
cd ~/sbat/forgeops
git checkout release/7.2.0
git checkout -b sbat
cp -R ~/sbat/secure-api-gateway-ob-uk/docker/7.1.0/ig docker/ig/config-profiles/sbat
cp -R ~/sbat/sbat-demo/certs/keystores docker/ig/config-profiles/sbat/.
echo 'COPY --chown=forgerock:root config-profiles/${CONFIG_PROFILE}/lib /opt/ig/lib' >> docker/ig/Dockerfile
bin/forgeops build ig --config-profile sbat
```

You should see your new IG image if you run a `docker images`

### IG config overlay

Before deploying, we are going to copy in some patches for SBAT so that

- We set the image pull policy to Never to make it easy for minikube
- We load in all the IG environment variables from the SBAT config map
- We create an ingress rule to direct everything to IG
- We add some SBAT specific secrets to the deployment

Switch to the forgeops repo base directory and generate the deployment manifest.

```

cd ~/sbat/forgeops
bin/forgeops generate ig --cdk
cp -R ~/sbat/sbat-demo/kustomize/sbat kustomize/deploy/ig/.

```

You will now have an IG deployment like this

```

kustomize/deploy/ig
├── kustomization.yaml
└── sbat
├── ig.yaml
├── ingress.yaml
└── secret.yaml

```

Update the deployment to patch the standard IG deployment and create an ingress:

```

cd ~/sbat/forgeops
cat >> kustomize/deploy/ig/kustomization.yaml <<EOF

# Added for SBAT

- sbat/ingress.yaml
patches:
- sbat/ig.yaml
- sbat/secrets.yaml
EOF

```

`~/sbat/forgeops/kustomize/deploy/ig/kustomization.yaml` should end up looking like this

```

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
components:

- ../image-defaulter
resources:
- ../../../kustomize/base/ig

# Added for SBAT

- sbat/ingress.yaml
patches:
- sbat/ig.yaml
- sbat/secrets.yaml

```

### Deploy

We are now ready to deploy IG. From the forgeops repo base directory:

```

cd ~/sbat/forgeops
kubectl apply -k kustomize/deploy/ig

```

If you run a `kubectl get pods` you should see something like

```

ig-54b866d6c5-9z6mn 1/1 Running 0 3m

```

## Spring config

### Create a spring config repo

Create your own mirror of the SBAT Spring config repo at https://github.com/SecureBankingAccessToolkit/securebanking-openbanking-spring-config.

Rename the `docker` directory to `minikube`. You should now have a repo structure like this

```

minikube
├── securebanking-openbanking-uk-rcs.yml
└── securebanking-openbanking-uk-rs.yml

```

There are a few changes you'll need to make to both of these files to work with your own deployment. If you are using the `sbat.demo` domain, you can just mirror the demo config repo at https://github.com/christian-brindley/sbat-config.

### Build the spring config server

Clone and build the spring config server from the SBAT repo

```

cd ~/sbat/securebanking-spring-config-server

```

Before you build the server, adjust the pom.xml to adjust the repository and disable push in the `dockerfile-maven-plugin` section

```

<configuration>
    <skipPush>true</skipPush>
    <repository>${project.artifactId}</repository>
    <buildArgs>
        <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
    </buildArgs>
    <tag>${tag}</tag>
</configuration>
```

Now build it

```
mvn package -DskipTests=true -Dtag=latest -file pom.xml
```

If you run `docker images` you should see the image `securebanking-spring-config-server` that you just built.

### Deploy the server

You can use the helm charts in this repo to deploy the config server. Under the `helm` directory:

```
helm install securebanking-spring-config-server securebanking-spring-config-server
```

If you run a `kubectl get pods` you should see the config server running - e.g.

```
securebanking-spring-config-server-5d7bd8778-pvmxd   1/1     Running   0          48s
```

## Remote consent service

### Build the remote consent service

Build the remote consent service from the SBAT repo

```
cd ~/sbat/securebanking-openbanking-uk-rcs
```

Before you build the service, adjust `securebanking-openbanking-uk-rcs-sample/pom.xml` to adjust the repository and disable push in the `dockerfile-maven-plugin` section

```
<configuration>
    <dockerfile>docker/Dockerfile</dockerfile>
    <skipPush>true</skipPush>
    <repository>securebanking-openbanking-uk-rcs</repository>
    <buildArgs>
        <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
    </buildArgs>
    <tag>${tag}</tag>
</configuration>
```

Now build it

```
mvn package -DskipTests=true -Dtag=latest -file securebanking-openbanking-uk-rcs-sample/pom.xml
```

If you run `docker images` you should see the image `securebanking-openbanking-uk-rcs` that you just built.

### Deploy the server

You can use the helm charts in this repo to deploy the remote consent service. Under the `helm` directory:

```
cd ~/sbat/sbat-demo/helm
helm install securebanking-openbanking-uk-rcs securebanking-openbanking-uk-rcs
```

If you run a `kubectl get pods` you should see the config server running - e.g.

```
securebanking-openbanking-uk-rcs-754f6758c8-phx97    1/1     Running   0          21s
```

## Resource server

### Deploy MongoDB

Deploy MongoDB into the sbat namespace. For simplicity, you can deploy with authentication switched off.

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install mongodb bitnami/mongodb --set auth.enabled=false
```

If you now run a `kubectl get pods` you should see mongo running

```
mongodb-7df9799c65-rdlxs                             1/1     Running   0          38s
```

### Build the resource server

Clone and build the mock resource server from the SBAT repo

```
cd ~/sbat/securebanking-openbanking-uk-rs
```

Before you build the service, adjust `securebanking-openbanking-uk-rs-simulator-sample/pom.xml` to adjust the repository and disable push in the `dockerfile-maven-plugin` section

```
<configuration>
    <dockerfile>docker/Dockerfile</dockerfile>
    <skipPush>true</skipPush>
    <repository>securebanking-openbanking-uk-rs</repository>
    <buildArgs>
        <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
    </buildArgs>
    <tag>${tag}</tag>
</configuration>
```

Now build it

```
mvn package -DskipTests=true -Dtag=latest -file securebanking-openbanking-uk-rs-simulator-sample/pom.xml
```

If you run `docker images` you should see the image `securebanking-openbanking-uk-rs` that you just built.

### Deploy the resource server

You can use the helm charts in this repo to deploy the resource server:

```
cd ~/sbat/sbat-demo/helm
helm install securebanking-openbanking-uk-rs securebanking-openbanking-uk-rs
```

If you run a `kubectl get pods` you should see the config server running - e.g.

```
securebanking-openbanking-uk-rs-5699575c9d-tb4sx     1/1     Running   0          46s
```

## UI

### Build the UI apps

Go to the SBAT UI repo

```
cd ~/sbat/securebanking-ui
```

Build the RCS UI

```
cd securebanking-rcs-ui
docker build -t securebanking-rcs-ui:latest -f projects/rcs/docker/Dockerfile .
```

Build the Swagger UI

```
cd ../securebanking-swagger-ui
docker build -t securebanking-swagger-ui:latest -f projects/swagger/docker/Dockerfile .
```

If you run `docker images` you should see the UI images `securebanking-rcs-ui` and `securebanking-swagger-ui` that you just built.

### Deploy the UIs

You can use the helm charts in this repo to deploy the UIs.

```
cd ~/sbat/sbat-demo/helm
helm install securebanking-ui securebanking-ui
```

If you run a `kubectl get pods` you should see the UI pods running - e.g.

```
securebanking-ui-consent-65dbb4c96f-rxgpk            1/1     Running   0          32s
securebanking-ui-swagger-7f67f7cf6d-55f66            1/1     Running   0          58s
```

## Testing

### Create a test user and associated account data

Create a test user in the ForgeRock platform to act as an Open Banking PSU.
Next, create test data in the mock resource server for that user.
The demonstration Postman collection inludes sample REST calls for creating the user via the user management API, and for creating account data via the resource server administration API.

### Run the TPP account information and payment flows

Run the main TPP flows to make sure that all the SBAT services are deployed correctly, and that the ForgeRock platform is configured correctly.
The demonstration Postman collection includes tests for the following flows

- TPP onboarding via dynamic client registration
- Account Information consent and API authorisation
- Domestic Paymment consent and API authorisation
