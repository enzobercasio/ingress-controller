# Troubleshooting Ingress Controller Set Up

### Access the route with https:// or http:// 
`https://your-app.apps-demo.shard.example.com`

`curl -v https://your-app.apps-demo.shard.example.com`

### Check the route type 

`oc get route <your-route> -o yaml`

### For curl with self-sgned or non-public certs, add -k (insecure) to ignore SSL warnings

`curl -vk https://your-app.apps-demo.shard.example.com`