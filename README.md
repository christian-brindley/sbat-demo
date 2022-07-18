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

### minikube

Make sure you are using the local minikube docker daemon 

```eval $(minikube docker-env)```

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

## Wildcard certificate

You'll need to generate a self signed wildcard certificate for the `sbat.demo` domain. A sample openssl config file is provided in this repo:

```
$ cd certs
$ openssl req -x509 -newkey rsa:4096 -keyout sbat-demo.key -out sbat-demo.crt -sha256 -days 365 -subj '/CN=sbat.demo' -nodes  -config openssl.cnf 
```

This needs to be trusted by your workstation browser to avoid TLS issues during testing.

## Namespace configuration

### Create the namespace
Create a dedicated namespace for sbat

```
$ kubectl create namespace sbat
$ kubens sbat
```

### TLS secrets
Load the server certificate you created previously as a TLS secret in your namespace

```
$ cd certs
$ kubectl create secret tls server.tls --key=sbat-demo.key --cert=sbat-demo.crt
```
Load the test trusted CA certs
```
$  kubectl create secret generic obri-ca --from-file=ca.crt=ca.crt
```

### Config map
Load the SBAT configmap into the namespace
```
$ cd overlay/namespace
$ cp configmap.yaml.sample configmap.yaml
```
Edit the configmap.yaml for your platform configuration, then apply it
```
$ kubectl apply -f configmap.yaml
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
- Step by step using the sample Postman collection
- Automatically via the SBAT initialiser task

## Identity Gateway

### Clone the gateway repo
```
$ git clone https://github.com/SecureBankingAccessToolkit/securebanking-openbanking-uk-gateway
```

### Build IG config
Run the config builder script from the SBAT repo.
```
$ cd securebanking-openbanking-uk-gateway
$ bin/config.sh -e dev -igm development init
```
This creates a full IG docker image directory within the repo under `docker/7.0/ig`. 

### Clone the forgeops repository
The ForgeRock forgeops repository has a sample deployment of Identity Gateway which may be used to deploy the SBAT IG instance.

### Build and deploy the IG image
Follow the forgeops documentation to build a custom IG image with the SBAT configuration and deploy it into the namespace. During the build step, copy the contents of the config directory you built previously in the  `securebanking-openbanking-uk-gateway` repo.

When the gateway deploys, it will fail because of missing environment variables which you set in the next step.

### IG config overlay
Apply the SBAT configuration to the IG deployment.

```
$ cd overlay/ig
$ kubectl apply -f ingress.yaml
$ kubectl apply -f configmap.yaml
$ kubectl apply -f deployment.yaml
```

Delete the IG pod to pick up the config changes.

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
$ git clone https://github.com/SecureBankingAccessToolkit/securebanking-spring-config-server
$ cd securebanking-spring-config-server
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
$ mvn package -DskipTests=true -Dtag=latest -file pom.xml
```
If you run `docker images` you should see the image `securebanking-spring-config-server` that you just built.

### Deploy the server
You can use the helm charts in this repo to deploy the config server.
```
$ cd helm
$ helm install securebanking-spring-config-server securebanking-spring-config-server
```
If you run a `kubectl get pods` you should see the config server running - e.g.
```
securebanking-spring-config-server-5d7bd8778-pvmxd   1/1     Running   0          48s
```

## Remote consent service
### Build the remote consent service
Clone and build the remote consent service from the SBAT repo
```
$ git clone https://github.com/SecureBankingAccessToolkit/securebanking-openbanking-uk-rcs
$ cd ssecurebanking-openbanking-uk-rcs
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
$ mvn package -DskipTests=true -Dtag=latest -file securebanking-openbanking-uk-rcs-sample/pom.xml
```
If you run `docker images` you should see the image `securebanking-openbanking-uk-rcs` that you just built.

### Deploy the server
You can use the helm charts in this repo to deploy the remote consent service.
```
$ cd helm
$ helm install securebanking-openbanking-uk-rcs securebanking-openbanking-uk-rcs
```
If you run a `kubectl get pods` you should see the config server running - e.g.
```
securebanking-openbanking-uk-rcs-754f6758c8-phx97    1/1     Running   0          21s
```
## Resource server

### Deploy MongoDB
Deploy MongoDB into the sbat namespace. For simplicity, you can deploy with authentication switched off.
```
$ helm repo add bitnami https://charts.bitnami.com/bitnami 
$ helm install mongodb bitnami/mongodb --set auth.enabled=false
```
If you now run a `kubectl get pods` you should see mongo running
```
mongodb-7df9799c65-rdlxs                             1/1     Running   0          38s
```
### Build the resource server
Clone and build the mock resource server from the SBAT repo
```
$ git clone https://github.com/SecureBankingAccessToolkit/securebanking-openbanking-uk-rs
$ cd securebanking-openbanking-uk-rs
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
$ mvn package -DskipTests=true -Dtag=latest -file securebanking-openbanking-uk-rs-simulator-sample/pom.xml
```
If you run `docker images` you should see the image `securebanking-openbanking-uk-rs` that you just built.
### Deploy the resource server
You can use the helm charts in this repo to deploy the resource server.
```
$ cd helm
$ helm install securebanking-openbanking-uk-rs securebanking-openbanking-uk-rs
```
If you run a `kubectl get pods` you should see the config server running - e.g.
```
securebanking-openbanking-uk-rs-5699575c9d-tb4sx     1/1     Running   0          46s
```
## UI
### Build the UI apps
Clone and build the UI apps from the SBAT repo
```
$ git clone https://github.com/SecureBankingAccessToolkit/securebanking-ui
$ cd securebanking-ui
$ cd ecurebanking-rcs-ui
$ docker build -t securebanking-rcs-ui:latest -f projects/rcs/docker/Dockerfile .
$ cd ../securebanking-swagger-ui
$ docker build -t securebanking-swagger-ui:latest -f projects/swagger/docker/Dockerfile .
```
If you run `docker images` you should see the UI images `securebanking-rcs-ui` and `securebanking-swagger-ui` that you just built.
### Deploy the UIs
You can use the helm charts in this repo to deploy the UIs.
```
$ cd helm
$     - helm install securebanking-ui securebanking-ui
```
If you run a `kubectl get pods` you should see the UI pods running - e.g.
```
securebanking-ui-consent-65dbb4c96f-rxgpk            1/1     Running   0          32s
securebanking-ui-swagger-7f67f7cf6d-55f66            1/1     Running   0          58s
```