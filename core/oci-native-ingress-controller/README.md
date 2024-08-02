# OCI Native Ingress Controller インストール手順
- cert-managerインストール
- OCI Native Ingress Controllerインストール
- IngressClassParametersとIngressClassのデプロイ
- Ingressのデプロイ（例）

## cert-managerインストール
```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml
```

## OCI Native Ingress Controllerインストール
レポジトリをクローン。
```bash
git clone https://github.com/oracle/oci-native-ingress-controller
```
`helm/oci-native-ingress-controller/values.yaml`を編集する。
編集箇所は、`compartment_id`と`subnet_id`。`subnet_id`にはOCI LoadBalancerで利用するサブネットのOCIDを定義する。

例↓
```yaml
#
# OCI Native Ingress Controller
#
# Copyright (c) 2023 Oracle America, Inc. and its affiliates.
# Licensed under the Universal Permissive License v 1.0 as shown at https://oss.oracle.com/licenses/upl/
#

# Default values for oci-native-ingress-controller.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

controller_class: oci.oraclecloud.com/native-ingress-controller
lease_lock_name: oci-native-ingress-controller
compartment_id: "ocid1.compartment.oc1..aaaaaaaaw4afdnpdddddddddcnamvmrzgdq"
subnet_id: "ocid1.subnet.oc1.eu-frankfurt-1.aaaaaaaa4hwibdtrczy7mhddddddd6mx74j2kcnzaoleka"
====
```
編集後、helmインストール。
```bash
helm install oci-native-ingress-controller helm/oci-native-ingress-controller
```
インストールされたかを確認。
```bash
kubectl get pods -n native-ingress-controller-system --selector='app.kubernetes.io/name in (oci-native-ingress-controller)' -o wide

NAME                                             READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
oci-native-ingress-controller-58fc779658-8d658   1/1     Running   1          25h   10.0.10.97    10.0.10.105   <none>           <none>
```

## IngressClassParametersとIngressClassのデプロイ
この二つのリソースを作成することでOCI上にロードバランサがプロビジョニングされる。バックエンドセットは、イングレスを作成することで作成される。

### IngressClassParameters.yaml
```yaml
apiVersion: "ingress.oraclecloud.com/v1beta1"
kind: IngressClassParameters
metadata:
  name: native-ic-params
spec:
  compartmentId: "ocid1.compartment.oc1..aaaaaaaaw4afdnptbjyff7ojizzddddddrzgdq"
  subnetId: "ocid1.subnet.oc1.eu-frankfurt-1.aaaaaaaa4hwibdtrczy7mhdhiwtxsddddkcnzaoleka"
  loadBalancerName: "native-ic-lb"
  isPrivate: false
  maxBandwidthMbps: 10
  minBandwidthMbps: 10
```

### IngressClass.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: native-ic-ingress-class
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: oci.oraclecloud.com/native-ingress-controller
  parameters:
    scope: Namespace
    namespace: default
    apiGroup: ingress.oraclecloud.com
    kind: ingressclassparameters
    name: native-ic-params
```

これらを`kubectl apply -f `で作成。