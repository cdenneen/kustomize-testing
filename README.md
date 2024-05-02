The ingress-nginx-internal-env-values is missing from the resulting HelmRelease

`cd env/dev/addons/ingress-nginx-internal; kustomize build .`

```yaml
apiVersion: v1
data:
  values.yaml: |
    ---
    foo: bar
kind: ConfigMap
metadata:
  labels:
    toolkit.fluxcd.io/tenant: sre-team
  name: ingress-nginx-internal-env-values
  namespace: flux-system
---
apiVersion: v1
data:
  values.yaml: |
    controller:
      watchIngressWithoutClass: true
      service:
        loadBalancerClass: service.k8s.aws/nlb
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-name: k8s-nginx-int-${clusterShortName}
          service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
          service.beta.kubernetes.io/aws-load-balancer-ip-address-type: ipv4
          service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
          service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: true
          service.beta.kubernetes.io/aws-load-balancer-scheme: internal
          service.beta.kubernetes.io/aws-load-balancer-subnets: ${nlb_subnets}
      ingressClassResource:
        name: nginx-internal
        enabled: true
        default: true
        controllerValue: "k8s.io/ingress-nginx-internal"
kind: ConfigMap
metadata:
  labels:
    toolkit.fluxcd.io/tenant: sre-team
  name: ingress-nginx-internal-values
  namespace: flux-system
---
apiVersion: v1
data:
  values.yaml: |
    controller:
      networkPolicy:
        enabled: true
      allowSnippetAnnotations: true
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: "topology.kubernetes.io/zone"
        whenUnsatisfiable: DoNotSchedule
      - maxSkew: 1
        topologyKey: "kubernetes.io/hostname"
        whenUnsatisfiable: DoNotSchedule
      replicaCount: 3
      metrics:
        enabled: true
        serviceMonitor:
          enabled: true
          namespace: platform-ingress-nginx
          additionalLabels:
            release: kube-prometheus-stack
      podAnnotations:
        prometheus.io.scrape: 'true'
        prometheus.io.port: '10254'
      service:
        annotations:
          prometheus.io/scrape: 'true'
          prometheus.io/port: '10254'
      resources:
        limits:
          memory: 360Mi
        requests:
          cpu: 100m
          memory: 360Mi
      autoscaling:
        enabled: true
        minReplicas: 3
        maxReplicas: 50
        targetCPUUtilizationPercentage: 70
        targetMemoryUtilizationPercentage: 98
kind: ConfigMap
metadata:
  labels:
    toolkit.fluxcd.io/tenant: sre-team
  name: ingress-nginx-values
  namespace: flux-system
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  labels:
    toolkit.fluxcd.io/tenant: sre-team
  name: ingress-nginx-internal
  namespace: flux-system
spec:
  chart:
    spec:
      chart: ingress-nginx
      interval: 1m
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
      version: 4.10.1
  dependsOn:
  - name: cilium
    namespace: flux-system
  - name: aws-load-balancer-controller
    namespace: flux-system
  - name: kube-prometheus-stack
    namespace: flux-system
  driftDetection:
    ignore:
    - paths:
      - /spec/replicas
    mode: enabled
  install:
    crds: CreateReplace
    createNamespace: true
    remediation:
      retries: 3
  interval: 20m
  releaseName: ingress-nginx-internal
  serviceAccountName: helm-controller
  storageNamespace: platform-ingress-nginx
  targetNamespace: platform-ingress-nginx
  timeout: 10m
  upgrade:
    crds: CreateReplace
    remediation:
      remediateLastFailure: false
  valuesFrom:
  - kind: ConfigMap
    name: ingress-nginx-values
    valuesKey: values.yaml
  - kind: ConfigMap
    name: ingress-nginx-internal-values
    valuesKey: values.yaml
```
