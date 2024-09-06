# galb-service-callout-test
testing service callouts with a regional ALB

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

### start setting up GALB

```
# verify that NEGs have been created for echo_server
export PROJECT=e2m-private-test-01 # replace with your own project
export REGION=us-central1 # replace with your own region
gcloud compute network-endpoint-groups list --project=$PROJECT

# reserve a static IP address
gcloud compute addresses create callout-demo-01  \
   --region=$REGION \
   --project=$PROJECT \
   --network-tier=STANDARD

# create health check
gcloud compute health-checks create http callout-demo-01-basic-check \
   --region=$REGION \
   --project=$PROJECT \
   --request-path='/' \
   --use-serving-port

gcloud compute backend-services create callout-demo-01-backend-service \
  --load-balancing-scheme=EXTERNAL_MANAGED \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=callout-demo-01-basic-check \
  --health-checks-region=$REGION \
  --project=$PROJECT \
  --region=$REGION

# adding backend service NEGs - in my setup, zones used are a, c and f
gcloud compute backend-services add-backend callout-demo-01-backend-service \
  --balancing-mode=RATE \
  --project=$PROJECT \
  --region=$REGION \
  --network-endpoint-group=echo-server \
  --network-endpoint-group-zone=${REGION}-a \
  --max-rate-per-endpoint=100

gcloud compute backend-services add-backend callout-demo-01-backend-service \
  --balancing-mode=RATE \
  --project=$PROJECT \
  --region=$REGION \
  --network-endpoint-group=echo-server \
  --network-endpoint-group-zone=${REGION}-c \
  --max-rate-per-endpoint=100

gcloud compute backend-services add-backend callout-demo-01-backend-service \
  --balancing-mode=RATE \
  --project=$PROJECT \
  --region=$REGION \
  --network-endpoint-group=echo-server \
  --network-endpoint-group-zone=${REGION}-f \
  --max-rate-per-endpoint=100

# create the URL map
gcloud compute url-maps create callout-demo-01-regional-l7-xlb-map \
  --default-service=callout-demo-01-backend-service \
  --region=$REGION \
  --project=$PROJECT

# create target proxy
gcloud compute target-http-proxies create callout-demo-01-regional-l7-xlb-proxy \
  --url-map=callout-demo-01-regional-l7-xlb-map \
  --url-map-region=$REGION \
  --project=$PROJECT \
  --region=$REGION

# create forwarding rule
gcloud compute forwarding-rules create l7-xlb-forwarding-rule \
  --load-balancing-scheme=EXTERNAL_MANAGED \
  --network-tier=STANDARD \
  --address=callout-demo-01 \
  --ports=80 \
  --region=$REGION \
  --project=$PROJECT \
  --target-http-proxy=callout-demo-01-regional-l7-xlb-proxy \
  --target-http-proxy-region=$REGION

# export the IP address
GALB_IP="$(gcloud compute addresses describe callout-demo-01 --region=$REGION --format='value(address)' --project $PROJECT)"

# test & time call to GALB
time curl --header "Content-Type: application/json" \
  --request POST \
  --data '{"username":"xYz","password":"xYz"}' \
  $GALB_IP 
```