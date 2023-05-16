
### Validate DNS resolution

```bash 
nslookup <Prism Central hostname>.<domain name>
```

```bash 
nslookup patata.apps.<Openshift Cluster Name>.<domain name>
nslookup gazpacho.apps.<Openshift Cluster Name>.<domain name>
nslookup croqueta.apps.<Openshift Cluster Name>.<domain name>
nslookup <Openshift Ingress Controller Virtual Ip>
nslookup <Openshift Control Plane Virtual IP>
```

```bash 
nslookup api.<Openshift Cluster Name>.<domain name>
```

### Validate connectivity

#### From Openshift Cluster to Prism Central

```bash 
nc -z -v <Prism Central hostname + fqdn> 9440
```

#### From Openshift Cluster to Prism Element

```bash 
nc -z -v <Prism Element Cluster VIP> 9440
nc -z -v <Prism Element Data Services IP> 3260
nc -z -v <Prism Element Data Services IP> 3205
```

#### From Openshift Cluster to Nutanix Files

```bash 
nc -z -v <Nutanix Files IPs> 2049
nc -z -v <Nutanix Files IPs> 7508
nc -z -v <Nutanix Files IPs> 20048
nc -z -v <Nutanix Files IPs> 20049
nc -z -v <Nutanix Files IPs> 20050
```

#### From Openshift Cluster to Nutanix Objects

```bash 
nc -z -v <Nutanix Objects Public IP> 80
nc -z -v <Nutanix Objects Public IP> 443
```

### From Openshift Cluster to Internet

```bash
curl -I https://mirror.openshift.com | grep HTTP
curl -I registry.redhat.io | grep HTTP
curl -I registry.connect.redhat.com | grep HTTP
curl -I quay.io | grep HTTP
```

### From Prism Central to Internet

```bash
curl -I rhcosredirector.apps.art.xq1c.p1.openshiftapps.com | grep HTTP
```