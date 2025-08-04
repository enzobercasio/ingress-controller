

# create cert
# Get cluster default domain

SHARDDOMAIN=shard.openshiftdemo.local
echo "Shard apps domain: $SHARDDOMAIN"

# Generate self signed wildcard certificate

WILDCARD="*.${SHARDDOMAIN}"

# Create a minimal OpenSSL config on the fly to set subjectAltName
cat > openssl-san.cnf <<EOF
[req]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[dn]
CN = ${WILDCARD}

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = ${WILDCARD}
DNS.2 = ${SHARDDOMAIN}
EOF

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -config openssl-san.cnf -extensions req_ext

openssl x509 -in tls.crt -noout -text | grep -E "Subject:|DNS:"

# Create the secret the shard router will use

oc -n openshift-ingress create secret tls shard-router-custom \
  --cert=tls.crt --key=tls.key

# create the sharded ingress controller

#shard-ingress-controller.yml 
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: sharded
  namespace: openshift-ingress-operator
spec:
  domain: shard.cluster-bbnvh.bbnvh.sandbox2226.opentlc.com
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/worker: ""
  routeSelector:
    matchLabels:
      type: sharded


oc apply -f shard-ingress-controller.yml 

oc -n openshift-ingress get pods -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=sharded -w


# test app 

oc new-project app-on-shard
oc new-app httpd~https://github.com/sclorg/httpd-ex.git --name my-httpd-app

# edge tls
cat <<'EOF' | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-httpd-app
  namespace: app-on-shard
  labels:
    type: sharded            
spec:
  host: my-httpd-app.shard.cluster-bbnvh.bbnvh.sandbox2226.opentlc.com
  to:
    kind: Service
    name: my-httpd-app
  tls:
    termination: edge
EOF

# no tls
cat <<'EOF' | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-httpd-app
  namespace: app-on-shard
  labels:
    type: sharded            
spec:
  host: my-httpd-app.shard.cluster-bbnvh.bbnvh.sandbox2226.opentlc.com
  to:
    kind: Service
    name: my-httpd-app
EOF

# resolve locally
curl --resolve my-httpd-app.shard.cluster-bbnvh.bbnvh.sandbox2226.opentlc.com:80:18.136.26.246 \
  http://my-httpd-app.shard.cluster-bbnvh.bbnvh.sandbox2226.opentlc.com/

# find the shards LB hostname
oc -n openshift-ingress get svc router-sharded -o jsonpath='{.status.loadBalancer.ingress[0].hostname}{"\n"}{.status.loadBalancer.ingress[0].ip}{"\n"}'

# get ip for the elb hostname
dig +short afa6dac7608c644768e485c854a4e3b9-1074462894.ap-southeast-1.elb.amazonaws.com | head -n1

# or add to /etc/hosts
