# IBM Cloud Internet Service Webhook for Cert Manager

This is a webhook solver for [IBM Cloud Internet Service](https://cloud.ibm.com/catalog/services/internet-services#about).

[The old repo](https://github.com/jb-dk/cert-manager-webhook-ibmcis) is out of support.  that's why we have the new repo.

[![Docker Repository on Quay](https://quay.io/repository/hzhihui/cert-manager-webhook-ibmcis/status "Docker Repository on Quay")](https://quay.io/repository/hzhihui/cert-manager-webhook-ibmcis)

## Prerequisites

* [cert-manager](https://github.com/jetstack/cert-manager): *tested with 1.13.2* *Kubernetes api 1.28.2*
    - [Installing on Kubernetes](https://cert-manager.io/next-docs/installation/kubernetes/)

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.yaml
 # kubectl get pods -n cert-manager

```

## Installation
_Notice: The pod will not startup until the steps under [configuration](#Issuer) is performed (there will be a secret there is not created until the steps are taken, so do not expect it just starts until that is in place )

Assuming your installation has 1. cert-manager running in name-space `cert-manager` and 2. accept this webhook will be installed into namespace `cert-manager-webhook-ibmcis`, 3. the API groups `acme.borup.work` will be used, then it is recommended to install via this pre-defined file .

```bash
git clone https://github.com/IBM/cert-manager-webhook-ibmcis.git
cd cert-manager-webhook-ibmcis
kubectl apply -f cert-manager-webhook-ibmcis.yaml
```

### How cert-manager-webhook-ibmcis.yaml is created (information)
This is just to help me remember how I created the static version of the file and for you to be inspired if you want to try to run it in a different configuration, however I will warn this is not the simplest thing to make it run in a different namespace.

```bash
helm template --name-template cert-manager-webhook-ibmcis ./deploy/cert-manager-webhook-ibmcis > cert-manager-webhook-ibmcis.yaml

```

### Customized installation
Only do the steps in this section - customized installation - if you did not do the step in installation.
```bash
helm install --name-template cert-manager-webhook-ibmcis ./deploy/cert-manager-webhook-ibmcis
```

## Issuer

0. (Optional but recommended) Generate a service id (`ibmcloud iam service-id-create cert-manager-webhook-ibmcis-sid -d "Service id that cert-manager-webhook-ibmcis uses"`), grant it "service access" level permission as `reader,writer,manager` to the relevant IBM Cloud Internet Service(s) only (example that grants access to specific instance of IBM Cloud Internet Services: `ibmcloud iam service-policy-create cert-manager-webhook-ibmcis-sid --service-instance SERVICE_INSTANCE_GUID --roles Reader,Writer,Manager `)

1. Generate API-KEY from IBM Cloud (example: `ibmcloud iam service-api-key-create cert-manager-webhook-ibmcis-sid-apikey cert-manager-webhook-ibmcis-sid -d "API key used for cert-manager-webhook-ibmcis to do the DNS01 ACME flow signed certificates"`)
2. Create a namespace to run this webhook in, recommend `cert-manager-webhook-ibmcis`. (like `kubectl create namespace cert-manager-webhook-ibmcis`)

3. Create secret to store the API Token
```bash
kubectl --namespace cert-manager-webhook-ibmcis create secret generic \
    ibmcis-credentials --from-literal=api-token='<IC_API_KEY>'
```

4. Create a staging issuer *Optional*
If you want to test and avoid rate-limit levels for production lets encryp ise this step (certificate validity is not for production though)
```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: cert-manager-webhook-ibmcis
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory

    # Email address used for ACME registration
    email: user@example.com # REPLACE THIS WITH YOUR EMAIL!!!

    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging

    solvers:
    - dns01:
        webhook:
          groupName: acme.borup.work
          solverName: ibmcis
          config:
            apiKeySecretRef:
              name: ibmcis-credentials
              key: api-token
            cisCRN:
              - "crn:v1:bluemix:public:internet-svcs:global:***::"
      selector:
        dnsZones:
        - 'example.com'

```

5. Create a production issuer
```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager-webhook-ibmcis
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory

    # Email address used for ACME registration
    email: user@example.com # REPLACE THIS WITH YOUR EMAIL!!!

    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod

    solvers:
    - dns01:
        webhook:
          groupName: acme.borup.work
          solverName: ibmcis
          config:
            apiKeySecretRef:
              name: ibmcis-credentials
              key: api-token
            cisCRN:
              - "crn:v1:bluemix:public:internet-svcs:global:***::"
      selector:
        dnsZones:
        - 'example.com'
```
Or you can create an ClusterIssuer as below :
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory

    # Email address used for ACME registration
    email: user@example.com # REPLACE THIS WITH YOUR EMAIL!!!

    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod

    solvers:
    - dns01:
        webhook:
          groupName: acme.borup.work
          solverName: ibmcis
          config:
            apiKeySecretRef:
              name: ibmcis-credentials
              key: api-token
            cisCRN:
              - "crn:v1:bluemix:public:internet-svcs:global:***::"
      selector:
        dnsZones:
        - 'example.com'
```

## Certificate

Then create the certificate which will use this issuer : https://cert-manager.io/docs/usage/certificate/

Create an certification using Issuer as below :
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: cert-manager-webhook-ibmcis
spec:
  commonName: example.com
  dnsNames:
  - example.com
  - "*.example.com"
  issuerRef:
    name: letsencrypt-staging
    kind: Issuer
  secretName: example-com-tls
```
Or create an certification using ClusterIssuer as below :
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
spec:
  commonName: example.com
  dnsNames:
  - example.com
  - "*.example.com"
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  secretName: example-com-tls
```
6. Get certificate

Namespace Issuer
```bash
kubectl get secret example-com-tls -n cert-manager-webhook-ibmcis
```
Cluster Issuer
```bash
kubectl get secret example-com-tls
```
### Automatically creating Certificates for Ingress resources

See [this](https://cert-manager.io/docs/usage/ingress/).

## Development

All DNS providers **must** run the DNS01 provider conformance testing suite,
else they will have undetermined behaviour when used with cert-manager.

**It is essential that you configure and run the test suite when creating a
DNS01 webhook.**

A Go test file for this provider is provided in [main_test.go](), and has been used for tests (via docker see below section).

Before you can run the test suite, you need to download the test binaries:

```bash
mkdir -p __main__/hack
wget -O- https://storage.googleapis.com/kubebuilder-tools/kubebuilder-tools-1.14.1-linux-amd64.tar.gz | tar xz --strip-components=1 -C __main__/hack
```

Then modify `testdata/ibmcis/config.json` to setup the configs.

Now you can run the test suite with:

```bash
TEST_ZONE_NAME=example.com. go test .
```
### Test via Docker (Mac test binaries not described in above section)

```bash
#CRN to be used in config.json as cisCRN
#ic resource service-instance borup.work-is -g default --output json | jq .[0].crn
ibmcloud resource service-instance <CIS INSTANCE NAME> -g <RESOURCE GROUP> --output json | jq .[0].crn 
docker run -it -v${PWD}:/workspace -w /workspace  --env-file .env golang:1.17 /bin/bash
apt update
apt upgrade -y
apt-get install -y bzr 
#TEST_ZONE_NAME=example.com. go test .
cat > testdata/ibmcis/config.json <<EOF
{
    "cisCRN": [ "crn:v1:bluemix:public:internet-svcs:global:xxxxxxxx::" ]
}
EOF

#export IC_API_KEY=xxxxx

TEST_ZONE_NAME=example.com. go test .

```
