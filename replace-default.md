
# Replace the default certificate 

### Get cluster default domain

    DOMAIN=$(oc get ingress.config.openshift.io/cluster -o jsonpath='{.spec.domain}')
    echo "Default apps domain: $DOMAIN"

### Generate self signed wildcard certificate

    WILDCARD="*.${DOMAIN}"

### Create a minimal OpenSSL config on the fly to set subjectAltName
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
    DNS.2 = ${DOMAIN}
    EOF

    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout tls.key -out tls.crt -config openssl-san.cnf -extensions req_ext

    openssl x509 -in tls.crt -noout -text | grep -E "Subject:|DNS:"

### Create/replace the secret the default router will use
    oc -n openshift-ingress delete secret default-router-custom --ignore-not-found

    oc -n openshift-ingress create secret tls default-router-custom \
      --cert=tls.crt --key=tls.key

### Point the default ingresscontroller to the new secret 
    oc -n openshift-ingress-operator patch ingresscontroller/default --type=merge -p \
      '{"spec":{"defaultCertificate":{"name":"default-router-custom"}}}'

### Watch rollout
    oc -n openshift-ingress get pods \
      -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default -w

### For local testing

### Import tls.crt into Keychain Access → System → mark “Always Trust”.

### Test app 
    oc new-project app-on-default
    oc new-app httpd~https://github.com/sclorg/httpd-ex.git --name my-httpd-app
    oc create route edge service my-http

    cat <<'EOF' | oc apply -f -
    apiVersion: route.openshift.io/v1
    kind: Route
    metadata:
      name: my-httpd-app
      namespace: app-on-default         
    spec:
      to:
        kind: Service
        name: my-httpd-app
      tls:
        termination: edge
    EOF