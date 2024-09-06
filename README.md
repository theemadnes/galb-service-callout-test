# galb-service-callout-test
testing service callouts 

### set up echo-server

```
kubectl apply -f echo-server/

# once pods are up, capture the LB IP
ECHO_IP="$(kubectl get service echo-server -n echo-server -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"

# and test it
curl $ECHO_IP

# test a POST with JSON body
curl --header "Content-Type: application/json" \
  --request POST \
  --data '{"username":"xYz","password":"xYz"}' \
  $ECHO_IP

# test latency too, to see how long it takes to get a response; personal testing shows ~20ms
time curl --header "Content-Type: application/json" \
  --request POST \
  --data '{"username":"xYz","password":"xYz"}' \
  $ECHO_IP 
```

### 