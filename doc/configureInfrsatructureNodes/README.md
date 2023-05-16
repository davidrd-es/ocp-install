# Configure Infrastructure nodes

Before create any additional machineset, run the following coomands to create and configure infrastructure nodes

``` bash
oc get $(oc get machineset -A --output=name) -n openshift-machine-api -o json | sed 's/-worker/-infra/g' | jq '.spec.template.spec +={"metadata":{"labels":{"node-role.kubernetes.io/infra":""}},"taints":[{"effect":"NoSchedule","key":"node-role.kubernetes.io/infra"}]}' | oc create -f -
oc patch ingresscontroller default -n openshift-ingress-operator  --type merge --patch '{"spec":{"nodePlacement":{"nodeSelector":{"matchLabels":{"node-role.kubernetes.io/infra":""}},"tolerations":[{"effect":"NoSchedule","operator":"Exists"}]}}}'
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec": {"nodeSelector": {"node-role.kubernetes.io/infra": ""},"affinity": {"podAntiAffinity": {"preferredDuringSchedulingIgnoredDuringExecution": [{"podAffinityTerm": {"namespaces": ["openshift-image-registry"],"topologyKey": "kubernetes.io/hostname"},"weight": 100}]}},"logLevel": "Normal"}}'
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"tolerations":[{"effect":"NoSchedule","key":"node-role.kubernetes.io/infra","operator":"Exists"}]}}'
cat <<EOF | oc create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |+
    alertmanagerMain:
      tolerations:
      - key: "node-role.kubernetes.io/infra"
        operator: "Exists"
        value: ""
        effect: "NoSchedule"    
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    prometheusK8s:
      tolerations:
      - key: "node-role.kubernetes.io/infra"
        operator: "Exists"
        value: ""
        effect: "NoSchedule"
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    prometheusOperator:
      tolerations:
      - key: "node-role.kubernetes.io/infra"
        operator: "Exists"
        value: ""
        effect: "NoSchedule"
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    grafana:
      tolerations:
      - key: "node-role.kubernetes.io/infra"
        operator: "Exists"
        value: ""
        effect: "NoSchedule"
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    k8sPrometheusAdapter:
      tolerations:
      - key: "node-role.kubernetes.io/infra"
        operator: "Exists"
        value: ""
        effect: "NoSchedule"
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    kubeStateMetrics:
      tolerations:
      - key: "node-role.kubernetes.io/infra"
        operator: "Exists"
        value: ""
        effect: "NoSchedule"
    telemeterClient:
      tolerations:
      - key: "node-role.kubernetes.io/infra"
        operator: "Exists"
        value: ""
        effect: "NoSchedule"
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    openshiftStateMetrics:
      tolerations:
      - key: "node-role.kubernetes.io/infra"
        operator: "Exists"
        value: ""
        effect: "NoSchedule"
    thanosQuerier:
      tolerations:
      - key: "node-role.kubernetes.io/infra"
        operator: "Exists"
        value: ""
        effect: "NoSchedule"
      nodeSelector:
        node-role.kubernetes.io/infra: ""
EOF
```