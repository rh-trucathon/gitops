# Central Cluster configuration

## Let's Encrypt

Je déploie des certificats TLS avec **Let's Encrypt** :

```sh
# Cluster DNS domain
export DOMAIN=central.sandbox2463.opentlc.com

# Get a valid certificate
sudo dnf install -y golang-github-acme-lego
lego -d "api.$DOMAIN" -d "*.apps.$DOMAIN" -a -m nmasse@redhat.com --dns route53 run

# Install it on the router
cert_dn="$(openssl x509 -noout -subject -in .lego/certificates/api.$DOMAIN.crt)"
cert_cn="${cert_dn#subject=CN = }"
kubectl create secret tls router-certs-$(date "+%Y-%m-%d") --cert=".lego/certificates/api.$DOMAIN.crt" --key=".lego/certificates/api.$DOMAIN.key" -n openshift-ingress --dry-run -o yaml > router-certs.yaml
kubectl apply -f "router-certs.yaml" -n openshift-ingress
kubectl patch ingresscontroller default -n openshift-ingress-operator --type=merge --patch-file=/dev/fd/0 <<EOF
{"spec": { "defaultCertificate": { "name": "router-certs-$(date "+%Y-%m-%d")" }}}  
EOF
```

> [!CAUTION]
> Sur le réseau Red Hat, il faut ajouter l'option `--dns.disable-cp`

## ACM Configuration

Installation de **Red Hat Advanced Cluster Management (ACM)** :

```sh
# Create the open-cluster-management namespace
oc create namespace open-cluster-management
oc project open-cluster-management

# Install the operator
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: open-cluster-management
  namespace: open-cluster-management
spec:
  targetNamespaces:
  - open-cluster-management
EOF
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: acm-operator-subscription
  namespace: open-cluster-management
spec:
  sourceNamespace: openshift-marketplace
  source: redhat-operators
  channel: release-2.11
  installPlanApproval: Automatic
  name: advanced-cluster-management
EOF

# Deploy ACM
oc apply -f - <<EOF
apiVersion: operator.open-cluster-management.io/v1
kind: MultiClusterHub
metadata:
  name: multiclusterhub
  namespace: open-cluster-management
spec: {}
EOF

# Wait for the installation to complete
while [ "$(oc get mch -o=jsonpath='{.items[0].status.phase}')" != "Running" ]; do echo "Deploying..."; sleep 2; done ; echo 'Done !'
```

Activer l'observabilité :

```sh
# Create the open-cluster-management-observability namespace
oc create namespace open-cluster-management-observability

# Copy the pull secret from the openshift namespace
DOCKER_CONFIG_JSON=`oc extract secret/pull-secret -n openshift-config --to=-`
echo $DOCKER_CONFIG_JSON
oc create secret generic multiclusterhub-operator-pull-secret \  
   -n open-cluster-management-observability \  
   --from-literal=.dockerconfigjson="$DOCKER_CONFIG_JSON" \  
   --type=kubernetes.io/dockerconfigjson

# Create an S3 bucket
BUCKET_NAME="redhat-summit-connect-2024-france-nmasse-observability"
aws s3api create-bucket --bucket "$BUCKET_NAME" --create-bucket-configuration LocationConstraint=eu-west-3 --region eu-west-3 --output json

# Deploy the observability add-on
oc apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: thanos-object-storage
  namespace: open-cluster-management-observability
type: Opaque
stringData:
  thanos.yaml: |
    type: s3
    config:
      bucket: $BUCKET_NAME
      endpoint: s3.eu-west-3.amazonaws.com
      insecure: false
      access_key: REDACTED
      secret_key: REDACTED
EOF
oc apply -f - <<EOF
apiVersion: observability.open-cluster-management.io/v1beta2
kind: MultiClusterObservability
metadata:
  name: observability
  namespace: open-cluster-management-observability
spec:
  observabilityAddonSpec: {}
  storageConfig:
    metricObjectStorage:
      name: thanos-object-storage
      key: thanos.yaml
EOF
```

## Leaderboard configuration

TODO