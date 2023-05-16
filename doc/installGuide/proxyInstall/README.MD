# Standard Openshift IPI install

## Prepare Client Machine

### Retrieve pull secret

From https://console.redhat.com/openshift

<img src="../../../_assets/img/getPullSecret.gif" alt="Synology DS415+" width=50%; height=auto; />

### Download required binaries

``` bash
{
sudo wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-install-linux.tar.gz
sudo tar -xvf openshift-install-linux.tar.gz
##Configure CCO
sudo wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/ccoctl-linux.tar.gz
sudo tar -xvf ccoctl-linux.tar.gz
##Download OC Client
sudo wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz
sudo tar -xvf openshift-client-linux.tar.gz
##Remove Downloads
sudo rm -r *.gz
##Move Binaries
sudo mv ccoctl openshift-install oc kubectl /usr/local/bin
}
```

### Define the following variables

``` bash
PROJECT_NAME=<openshift cluster name>
NX_PC_IP="<nutanix Prism Central fqdn name>"
NX_PC_USER="<Openshift service account>"
NX_PC_PASS="<Openshift service account passowrd>"
NX_PE_NAME='<Nutanix Prism Element Cluster Name>' # this
NX_PE_IP="10.1.1.20" # this
NX_PE_UUID='000582c6-cf0d-e0a8-0000-000000016950' # this
NX_PE_NETWORK="7e6bfe2f-4433-40d3-a337-893d7131ab87" # this
OC_CP_VIP="<Openshift Control Plane Virtual IP>"
OC_IS_VIP="<Openshift Ingress Controller Virtual Ip>"
OC_SERVICE_NETWORK='<Openshift service network>'
OC_POD_NETWORK='Openshift Pod Network'
OC_SECRET='<put your pull secret here>'
OC_CP_CPU='<Control Plane CPU per node>'
OC_CP_MEM='<Control Plane Memory per node>'
OC_CP_LOCAL_DISK_SIZE='<Control Plane Disk per node>'
OC_WORKER_COUNT=<Amount of worker nodes>
OC_WORKER_CPU='<Worker CPU per node>'
OC_WORKER_MEM='<Worker Memory per node>'
OC_WORKER_LOCAL_DISK_SIZE='<Worker Disk per node>'
DOMAIN='<domain>'
NX_PC_CA='<your ca certificate here>'
PROXY_HTTP='<http://<username>:<pswd>@<ip>:<port>'
PROXY_HTTPS='<https://<username>:<pswd>@<ip>:<port>>'
NOPROXY='<excluded domains>'
```

### Create the folder structure

``` bash
{
mkdir $PROJECT_NAME
mkdir $PROJECT_NAME/credentials
mkdir $PROJECT_NAME/credrequests
mkdir $PROJECT_NAME/credoutput
mkdir $PROJECT_NAME/bck
}
```

### Create SSH Keys for OCP nodes

``` bash
mkdir $PROJECT_NAME/sshkey
ssh-keygen -t rsa -b 4096 -f $PROJECT_NAME/sshkey/id_rsa -N ""
SSH_PUB=$(cat $PROJECT_NAME/sshkey/id_rsa.pub)
```

### Download CA certs from PC

``` bash
openssl s_client -showcerts -verify 5 -connect $NX_PC_IP:9440 < /dev/null |
   awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/{ if(/BEGIN CERTIFICATE/){a++}; out="cert"a".crt"; print >out}'
sudo cp cert* /usr/local/share/ca-certificates
sudo update-ca-certificates
```

### Create the install-config.yaml

``` bash
#Create install Files
cat > $PROJECT_NAME/install-config.yaml <<EOF
apiVersion: v1
baseDomain: $DOMAIN
credentialsMode: Manual
additionalTrustBundle: | $NX_PC_CA
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    nutanix: 
      cpus: $OC_WORKER_CPU
      coresPerSocket: 1
      memoryMiB: $OC_WORKER_MEM
      osDisk:
        diskSizeGiB: $OC_WORKER_LOCAL_DISK_SIZE
  replicas: $OC_WORKER_COUNT
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    nutanix: 
      cpus: $OC_CP_CPU
      coresPerSocket: 1
      memoryMiB: $OC_CP_MEM
      osDisk:
        diskSizeGiB: $OC_CP_LOCAL_DISK_SIZE
  replicas: 3
metadata:
  name: $PROJECT_NAME
networking:
  clusterNetwork:
  - cidr: $OC_POD_NETWORK
    hostPrefix: 24
  machineNetwork:
  - cidr: 10.68.204.0/24
  networkType: OpenShiftSDN
  serviceNetwork:
  - $OC_SERVICE_NETWORK
platform:
  nutanix:
    apiVIP: $OC_CP_VIP
    ingressVIP: $OC_IS_VIP
    prismCentral:
      endpoint:
        address: $NX_PC_IP
        port: 9440
      password: $NX_PC_PASS
      username: $NX_PC_USER
    prismElements:
    - endpoint:
        address: $NX_PE_IP
        port: 9440
      uuid: $NX_PE_UUID
    subnetUUIDs:
    - $NX_PE_NETWORK
publish: External
pullSecret: '$OC_SECRET'
sshKey: $SSH_PUB
proxy:
  httpProxy: $PROXY_HTTP
  httpsProxy: $PROXY_HTTPS
  noProxy: $NOPROXY
EOF
```

Install-config.yaml should look like:

```yaml
apiVersion: v1
baseDomain: emeagso.lab
credentialsMode: Manual
additionalTrustBundle: | 
    -----BEGIN CERTIFICATE-----
    MIID7TCCAtWgAwIBAgIUOYsVJ5D80DW/vytXTM+7lw5yaPAwDQYJKoZIhvcNAQEL
    BQAwfjELMAkGA1UEBhMCTkwxEjAQBgNVBAgTCUFtc3RlcmRhbTESMBAGA1UEBxMJ
    QW1zdGVyZGFtMRkwFwYDVQQKExBDdXN0b21lciBTdWNjZXNzMRMwEQYDVQQLEwpD
    b25zdWx0aW5nMRcwFQYDVQQDDA5lbWVhZ3NvX3Jvb3RjYTAeFw0yMjA5MjcxMDAw
    NDdaFw0yNTA5MjYxMDAxMTdaMH4xCzAJBgNVBAYTAk5MMRIwEAYDVQQIEwlBbXN0
    ZXJkYW0xEjAQBgNVBAcTCUFtc3RlcmRhbTEZMBcGA1UEChMQQ3VzdG9tZXIgU3Vj
    Y2VzczETMBEGA1UECxMKQ29uc3VsdGluZzEXMBUGA1UEAwwOZW1lYWdzb19yb290
    Y2EwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC4dkkYpFK700LCXGJE
    w0gz+devHjZWgI0VICw5adX1GBwH3Lp/Np27oPCtx8XO3qKHwdqof7Oe6Oid2irL
    fKC1NWB7+0fLaPnDBiCUnsleoZuUYNxUe8s7e9VNEipsfhOhwuf5fqiY9GqbB85E
    FvnQ5Z9tdyGRLnTcOn5vddRIIWnLqu1K8wAXvB7xXYQ/QPeG+eAtwOz2GCAabHfM
    XzoNnx1XZY6S5SDMVq3/613DQEA94lcxCF1WCzjdggOc0u6feY04fT3/6o5+P6Kj
    zwr+PKE3HPESlIC64s1yhgYx/FB9ORy9mVC5jKQBbqDmLVzaIrhiZ7yh3rE8Pd8m
    44dZAgMBAAGjYzBhMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8EBTADAQH/MB0G
    A1UdDgQWBBRhH13uucIdybPXsHwLyv3yTFYMEDAfBgNVHSMEGDAWgBRhH13uucId
    ybPXsHwLyv3yTFYMEDANBgkqhkiG9w0BAQsFAAOCAQEAGfp7ROWKml8h1176uDfW
    qtZascT8qt880LLtqPwwzqPQEq9V1qULnnv6fwalmdXiFyAJ+rhnxg1PNWU6rRfk
    JdB55xocAIq8VdDrHdkOoZFOsucUvKn/79fdjKK2J1s5Mvh0jt10DFmZES9wTD/d
    1MDrA+HLDzVgyBl0hddy0ENZ7qjIpIN2+8nppDyaIrsh1TuBbX6MdtXMgR/J8Ndq
    tU+HxNMuDgBnxKLpLJXhSIWTLoEzNPx2hW50mQqKCgWKhAtpDLl//Lt1O8DpdalU
    W9IR04qCno4MFEW3GPPPRN17L+JSFYQ6DLX2djihuBuBUXmmfsya5UsU6g+VU12d
    1Q==
    -----END CERTIFICATE-----
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    nutanix: 
      cpus: 8
      coresPerSocket: 1
      memoryMiB: 8192
      osDisk:
        diskSizeGiB: 120
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    nutanix: 
      cpus: 4
      coresPerSocket: 1
      memoryMiB: 16384
      osDisk:
        diskSizeGiB: 120
  replicas: 3
metadata:
  name: drd-ocp-cl1
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/16
    hostPrefix: 24
  machineNetwork:
  - cidr: 10.68.204.0/24
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  nutanix:
    apiVIP: 10.68.204.45
    ingressVIP: 10.68.204.46
    prismCentral:
      endpoint:
        address: merlin.emeagso.lab
        port: 9440
      password: nutanix/4u
      username: david.rios@emeagso.lab
    prismElements:
    - endpoint:
        address: 10.68.97.100
        port: 9440
      uuid: 000582c6-cf0d-e0a8-0000-000000016950
    subnetUUIDs:
    - 7e6bfe2f-4433-40d3-a337-893d7131ab87
publish: External
pullSecret: '{"auths":{"cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGVDZUWjhKT1VHQ1oyRlZRS01JOUxMU0RHSQ==","email":"david.riosdelgado@nutanix.com"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2KUWjhKT1VHQ1oyRlZRS01JOUxMU0RHSQ==","email":"david.riosdelgado@nutanix.com"},"registry.connect.redhat.com":{"auth":"fHVoYy1wb29sLTIzNTRlZTIyLWUxZDEtNDMyMy05MjU4LWZmM2NlOWIyNGZmNTpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSTNaVEZrWkRNM1pqazNOakEwT1dZMFltSXdNemt5WW1GbE1UTQ3NPQW16LUN5WThsNWJ0aVlTU3dhpR3h2UkZESTEtSkt1aWFmNA==","email":"david.riosdelgado@nutanix.com"},"registry.redhat.io":{"auth":"fHVoYy1wb29sLTIzNTRlZTIyLWUxZDEtNDMyMy05MjU4LWZmM2NlOWIyNGZmNTpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSTNaVEZrWkRNM1pqazNOakEwT1dZMFltSXdNemt5WW1GbE1UTTNPRFU0TXlKOS5wMm90YUpsaEtBNkRKN1NSUFnMmp5eXlydHUtLTVVZldVLW1iX1k0elBnWE1xU2Z3R0dLcFhsVjg4SHNtWmVkb0dSX29TQnphRjNZQ3NPQW16LUN5WThsNWJ0aVlTU3dhUjAtc2t6Qmp1b05lT2xseXJTd0pXTGsyNmZ2STZHa1d4ckJKTHB4MUxfWFAxTElocEVJank5R3dkYXMzTXllbWJiQ3lidEEtQjBDa21feG53cm4taW14SmNXbi1XbVEzcW1Bd3FPSU1TOGk0VW4tZEZjaHY0bUFwbTZHcWpBTHlqbUgtZHVJbG5mcC01QUpmRHVDU0VsLWpNZVl1M2tfSzFadUpZcG5Hc3l6OVJHWWJEc1ZabmhpR3h2UkZESTEtSkt1aWFmNA==","email":"david.riosdelgado@nutanix.com"}}}'
sshKey: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQClGYDmRowWdw+Rhrf+KHZJxBTTUexlHHKTwr7/IDTxKZi84VFSu6ORox9eZ/KrUrqaI0w74MfR0+dligARG60eU6SbUciwb/5EQEXx/JtxVwPREQ+EV6IkkpyGyWZGaCfs++XyhC0PC6B8wGLjNh5trUBYY5C1tkgKXnkA8Gp7Ldzj40zUUPfN4ijSwaA2xQaOJGpFMQpoNkJqRY3i7fQaDeaZWCyNyjBT7LQSbbG4y/U3yWcRWF1PfUd7yaY+9u89GjAyQjkOR5uB4/++Y7TCeDVCDqhjoIJE2zNBq5J48h7cDxe1KobOMjlZUUyR+cfyiCEdKjxSgW+InK031B/We3TMivOb2FFzMS/ZjkDEJ/jGUu5gSVVQPQTxie51RReGWR4tDhgQRNeCQW6omhrzFiGZC82FjvijL8ILITPWT82QNJqNzQP1mYJjLbutQMDRlIifCGd34EFbzUJrSOAUVPQxpMOT9z+zZLRINcnEBkdU/bbaMJ6Hr4eFDFnkJn6BQhHwqii+YUDvPyn6nni3UuQk3TyDSsymdhCoZyW6ugmZD+4OkOOK0PfCskF7gEK0fLAcZu++iyal72Be3+LUS32vONUcMvLJdpfmpgGgJUwpBYsP4nmVVMey23t8S1w9Dy9DKVOyWO/FwjJUBtgEjM+lFqaZhW9nRFEGcasUlQ== drd@drd-dev-02
proxy:
  httpProxy: http://user:pwd@10.1.1.1:8080
  httpsProxy: https://user:pwd@10.1.1.1:80443
  noProxy: emeagso.lab

PROXY_HTTP='<http://<username>:<pswd>@<ip>:<port>'
PROXY_HTTPS='<https://<username>:<pswd>@<ip>:<port>>'
NOPROXY='<excluded domains>'
```

### Create the credentials files

``` bash
cat > $PROJECT_NAME/credentials/credentials.yaml <<EOF
credentials:
- type: basic_auth 
  data:
    prismCentral: 
      username: $NX_PC_USER
      password: $NX_PC_PASS
    prismElements: 
    - name: $NX_PE_NAME
      username: $NX_PC_USER
      password: $NX_PC_PASS
EOF
```

### Create the cloud credentials template

``` bash
cat > $PROJECT_NAME/credrequests/credrequests.yaml <<EOF
apiVersion: cloudcredential.openshift.io/v1
kind: CredentialsRequest
metadata:
  annotations:
    include.release.openshift.io/self-managed-high-availability: "true"
  labels:
    controller-tools.k8s.io: "1.0"
  name: openshift-machine-api-nutanix
  namespace: openshift-cloud-credential-operator
spec:
  providerSpec:
    apiVersion: cloudcredential.openshift.io/v1
    kind: NutanixProviderSpec
  secretRef:
    name: nutanix-credentials
    namespace: openshift-machine-api
EOF
```

### Create cloud credentials secret

``` bash
sudo ccoctl nutanix create-shared-secrets \
--credentials-requests-dir=$PROJECT_NAME/credrequests/ \
--output-dir=$PROJECT_NAME/credoutput \
--credentials-source-filepath=$PROJECT_NAME/credentials/credentials.yaml
```

### Generate install manifest and backup files

``` bash
cp $PROJECT_NAME/install-config.yaml $PROJECT_NAME/bck/
openshift-install create manifests --dir $PROJECT_NAME
cp -r $PROJECT_NAME/credoutput/manifests/*credentials.yaml $PROJECT_NAME/manifests
```

## Install Openshift

``` bash
openshift-install create cluster --dir $PROJECT_NAME --log-level=debug
```

<img src="../../../_assets/img/ocpInstall.gif" alt="Synology DS415+" width=50%; height=auto; />

Once cluster is deployed UI console should be available at https://console-openshift-console.apps.<cluster name>.<domain name>

<img src="../../../_assets/img/openshiftLogin.png" alt="Synology DS415+" width=100%; height=auto; />

kubeconfig is available at:

  export KUBECONFIG=$(pwd)/$PROJECT_NAME/auth/kubeconfig

kubeadmin credentials are available at:

  $PROJECT_NAME/auth/kubeadmin-password