# certificate-expiry-monitor
Utility that exposes the expiry of TLS certificates as Prometheus metrics

## Building
To build the Docker image, simply run `docker build`:
```
docker build . -t muxinc/certificate-expiry-monitor:latest
```

## Running
Run the Docker image using the executable at `/app`:
```
→ docker run muxinc/certificate-expiry-monitor:latest /app --help
Usage of /app:
  -frequency duration
    	Frequency at which the certificate expiry times are polled (default 1m0s)
  -hostnames string
    	Comma-separated SNI hostnames to query
  -insecure
    	If true, then the InsecureSkipVerify option will be used with the TLS connection, and the remote certificate and hostname will be trusted without verification (default true)
  -kubeconfig string
    	Path to kubeconfig file if running outside the Kubernetes cluster
  -labels string
    	Label selector that identifies pods to query
  -logformat string
    	Log format (text or json) (default "text")
  -loglevel string
    	Log-level threshold for logging messages (debug, info, warn, error, fatal, or panic) (default "error")
  -metricsPort int
    	TCP port that the Prometheus metrics listener should use (default 8888)
  -namespaces string
    	Comma-separated Kubernetes namespaces to query (default "default")
  -port int
    	TCP port to connect to each pod on (default 443)

```

### Kubernetes Manifest
You're probably going to want to run the certificate-expiry monitor in a Kubernetes cluster. The following manifest shows how you might monitor a set of ingress pods matching the label `k8s-app=my-ingresses` in the `default` namespace for the `foobar.example.com` certificate hostname:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: certificate-expiry-monitor
  namespace: default
spec:
  minReadySeconds: 5
  revisionHistoryLimit: 3
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: certificate-expiry-monitor
    spec:
      containers:
      - command:
        - /app
        - -labels
        - k8s-app=my-ingresses
        - -namespaces
        - default
        - -frequency
        - 1m
        - -hostnames
        - foobar.example.com
        image: muxinc/certificate-expiry-monitor:latest
        imagePullPolicy: Always
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8888
          initialDelaySeconds: 5
          timeoutSeconds: 5
        name: certificate-expiry-monitor
        resources:
          limits:
            cpu: 200m
            memory: 200Mi
          requests:
            cpu: 200m
            memory: 200Mi
```

## Monitoring
A Prometheus endpoint is available at `/metrics` on TCP port `:8888` (customizable with `metricsPort`). 

### Gauges
| Name  | Description  |
|---|---|
| `certificate_expiry_monitor_matching_pods`  | Number of pods that match the label filter in a namespace  |
| `certificate_expiry_monitor_certificate_is_expired`  | Number of expired certificates  |
| `certificate_expiry_monitor_certificate_is_not_yet_valid`  | Number of certificates that are not yet valid  |
| `certificate_expiry_monitor_certificate_is_valid`  | Number of valid certificates  |
| `certificate_expiry_monitor_seconds_since_cert_issued`  | Seconds since the certificate was issued  |
| `certificate_expiry_monitor_seconds_until_cert_expires`  | Seconds until the certificate expires  |

### Counters
| Name  | Description  |
|---|---|
| `certificate_expiry_monitor_tls_open_connection_error`  | Number of times an error occurred while opening a TLS connection to a pod |
| `certificate_expiry_monitor_tls_close_connection_error`  | Number of times an error occurred while closing a TLS connection to a pod |
| `certificate_expiry_monitor_certificate_not_found`  | Number of times an applicable TLS certificate could not be found for a pod and hostname combination |

## Healthcheck
A simple healthcheck is available at `/healthz` on the TCP port `:8888` (customizable with `metricsPort`):

```
→ curl -v http://localhost:8888/healthz
*   Trying ::1...
* TCP_NODELAY set
* Connection failed
* connect to ::1 port 8888 failed: Connection refused
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8888 (#0)
> GET /healthz HTTP/1.1
> Host: localhost:8888
> User-Agent: curl/7.52.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Mon, 04 Mar 2019 17:56:45 GMT
< Content-Length: 7
< Content-Type: text/plain; charset=utf-8
<
* Curl_http_done: called premature == 0
* Connection #0 to host localhost left intact
Healthy
```
