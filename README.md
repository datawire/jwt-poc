# JWT POC

This POC will install Ambassador, Prometheus, Grafana, and configure JWT authentication.

## Initial setup

1. If you're on GKE, make sure you have an admin cluster role binding:

```
kubectl create clusterrolebinding my-cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud info --format="value(config.account)")
```

2. Clone this repository locally:

```
git clone https://github.com/datawire/jwt-poc.git
```

4. Add your license key to the `ambassador-pro.yaml` file.

## Install Ambassador

1. Install Ambassador with the following commands, along with the demo QOTM service and a route to HTTPbin service.
   
   ```
   kubectl apply -f statsd-sink.yaml
   kubectl apply -f ambassador-pro.yaml
   kubectl apply -f ambassador-service.yaml
   kubectl apply -f httpbin.yaml
   ```

2. Get the IP address of Ambassador: `kubectl get svc ambassador`.

3. Send a request to `httpbin`:

   ```
   curl -v http://{AMBASSADOR_IP}/httpbin/ip
   {
      "origin": "108.20.119.124, 35.184.242.212, 108.20.119.124"
   }
   ```

## Metrics

Next, we'll set up metrics using Prometheus and Grafana.

1. Install the Prometheus Operator.

   ```
   kubectl apply -f monitoring/prometheus.yaml
   ```

2. Wait 30 seconds until the `prometheus-operator` pod is in the `Running` state.

3. Create the rest of the monitoring setup:

   ```
   kubectl apply -f monitoring/prom-cluster.yaml
   kubectl apply -f monitoring/prom-svc.yaml
   kubectl apply -f monitoring/servicemonitor.yaml
   kubectl apply -f monitoring/grafana.yaml
   ```

4. Send some traffic through Ambassador (metrics won't appear until some traffic is sent). You can just run the `curl` command to httpbin above a few times.

5. Get the IP address of Grafana: `kubectl get svc grafana`

6. In your browser, go to the `$GRAFANA_IP` and log in using username `admin`, password `admin`.

7. Configure Prometheus as the Grafana data source. Give it a name, choose type Prometheus, and point the HTTP URL to `http://prometheus.default:9090`. Save & Test the Data Source.

8. Import a dashboard. Click on the + button, and then choose Import. Upload the `ambassador-dashboard.json` file to Grafana. Choose the data source you created in the previous step, and click import.

9. Go to the Ambassador dashboard!

## JWT

1. Configure the JWT filter:

   ```
   kubectl apply -f filter.yaml
   ```

2. Send a valid JWT to the `jwt-httpbin` URL:

   ```
   curl -i --header "Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ." $AMBASSADOR_IP/jwt-httpbin/ip
   ```

3. Send an invalid JWT, and get a 401:

   ```
   curl -i 35.194.55.166/jwt-httpbin/ip
   HTTP/1.1 401 Unauthorized
   content-length: 58
   content-type: text/plain
   date: Thu, 28 Feb 2019 01:07:10 GMT
   server: envoy
   ```

4. Note that we've configured the `jwt-httpbin` URL to require JWTs, but the `httpbin` URL does not:

   ```
   curl -v http://{AMBASSADOR_IP}/httpbin/ip
   {
      "origin": "108.20.119.124, 35.184.242.212, 108.20.119.124"
   }
   ```

The JWT is validated using public keys supplied in a JWKS file. For the purposes of this demo, we're supplying a Datawire JWKS file. You can change the JWKS file by modifying the `filter.yaml` manifest and changing the `jwksURI` value.