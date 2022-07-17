# Quick start for deploying SBAT

## Introduction

This repository contains a sample deployment pattern for the ForgeRock Secure Banking Access Toolkit, using minikube as the example kubernetes platform.

The sample uses docker images built and stored locally in the minikube docker repository. 

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

would require the following entry in /etc/hosts
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

### Build the spring config server

### Deploy the server

## Remote consent service

## Resource server

### Deploy MongoDB

### Build the resource server

### Deploy the resource server

## UI