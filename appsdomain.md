# Using AppsDomain spec to add alternative domain for applications

### Define the alternative domain
    APPSDOMAIN=applications.cluster-tblw7.tblw7.sandbox9.opentlc.com
    echo "Applications domain: $APPSDOMAIN"

### Generate self signed wildcard certificate

    APPSWILDCARD="*.${APPSDOMAIN}"

### Create a minimal OpenSSL config on the fly to set subjectAltName
    cat > openssl-san.cnf <<EOF
    [req]
    default_bits       = 2048
    prompt             = no
    default_md         = sha256
    req_extensions     = req_ext
    distinguished_name = dn

    [dn]
    CN = ${APPSWILDCARD}

    [req_ext]
    subjectAltName = @alt_names

    [alt_names]
    DNS.1 = ${APPSWILDCARD}
    DNS.2 = ${APPSDOMAIN}
    EOF

    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout tls.key -out tls.crt -config openssl-san.cnf -extensions req_ext

    openssl x509 -in tls.crt -noout -text | grep -E "Subject:|DNS:"

### Create/replace the secret the default router will use
    oc -n openshift-ingress delete secret default-router-custom --ignore-not-found

    oc -n openshift-ingress create secret tls applications-router-custom \
      --cert=tls.crt --key=tls.key

### update the default ingress about the alternative domain
    oc patch ingresses.config.openshift.io/cluster \
      --type=merge \
      -p '{"spec":{"appsDomain":"applications.cluster-tblw7.tblw7.sandbox9.opentlc.com"}}'

    oc edit ingresses.config/cluster -o yaml

    oc get po -n openshift-ingress

### Test app 
    oc new-project app-on-appsdomain
    oc new-app httpd~https://github.com/sclorg/httpd-ex.git --name my-httpd-app

    cat <<'EOF' | oc apply -f -
    apiVersion: route.openshift.io/v1
    kind: Route
    metadata:
      name: my-httpd-app
      namespace: app-on-appsdomain       
    spec:
      to:
        kind: Service
        name: my-httpd-app
      tls:
        termination: edge
    EOF


### rResolve locally
    curl --resolve my-httpd-app-app-on-appsdomain.applications.cluster-tblw7.tblw7.sandbox9.opentlc.com:80:122.248.209.174 \
      my-httpd-app-app-on-appsdomain.applications.cluster-tblw7.tblw7.sandbox9.opentlc.com/

### Find the shards LB hostname
    oc get svc -n openshift-ingress
    oc -n openshift-ingress get svc router-appdomain -o jsonpath='{.status.loadBalancer.ingress[0].hostname}{"\n"}{.status.loadBalancer.ingress[0].ip}{"\n"}'

### Get ip for the elb hostname
    dig +short a801b77fc6d1a4029a5a6267c44fbef3-1994312820.ap-southeast-1.elb.amazonaws.com | head -n1

### Or add to /etc/hosts

### Bypass secure
    curl -vk https://my-httpd-app-app-on-appsdomain.applications.cluster-tblw7.tblw7.sandbox9.opentlc.com