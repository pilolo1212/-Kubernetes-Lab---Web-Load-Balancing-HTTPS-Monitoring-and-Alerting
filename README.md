# Kubernetes Lab - Web Load Balancing, HTTPS, Monitoring, and Alerting

This project is the Kubernetes version of Lab 1.  
The goal is to keep the same logical architecture as the original lab:

- an external entry point
- reverse proxying
- load balancing across two backend servers
- HTTPS with a self-signed certificate
- monitoring with Prometheus and Grafana
- alerting with Alertmanager

## 1. Environment

- **OS:** Windows
- **Cluster:** Minikube
- **Driver / Runtime:** Docker Desktop
- **Shell used:** PowerShell and CMD
- **Ingress access:** `kubectl port-forward`
- **Local host mapping:** Windows `hosts` file updated to map:

```txt
127.0.0.1 flask.local

Because Windows blocked direct use of ports 80/443 in our setup, we used local forwarded ports:

HTTP: 8080
HTTPS: 8443
2. Project Architecture

The final Kubernetes architecture is:

Flask app
Deployment with 2 replicas
ClusterIP Service for internal load balancing
NGINX Ingress Controller
Ingress resource for host flask.local
TLS Secret for HTTPS
Prometheus + Grafana for monitoring
Alertmanager for alert routing

Logical flow:

Client
  |
  v
NGINX Ingress
  |
  v
flask-service
  |
  +---- Pod 1
  |
  +---- Pod 2

Monitoring flow:

Prometheus ---> scrapes cluster/node/app-related metrics
Grafana -----> visualizes metrics
Alertmanager -> handles alerts
3. Application Files

Main files used in the project:

app.py
requirements.txt
Dockerfile
deployment.yaml
service.yaml
ingress.yaml
test-alert-rule.yaml
alertmanager-email.yaml
4. Flask Application

The application is a simple Flask web app that returns the pod identity so load balancing can be verified.

Example idea:

return a message from the current pod
refresh multiple times to observe alternation between replicas
5. Build Docker Image

Example Docker build command:

docker build -t flask-app .

If using Minikube local Docker environment, use the Minikube Docker daemon first if needed.

Example:

minikube docker-env

Then evaluate the environment if needed in the shell you use.

6. Kubernetes Deployment
6.1 Deployment

We deployed the Flask application with 2 replicas.

Apply:

kubectl apply -f deployment.yaml

Check:

kubectl get pods
6.2 Service

We created a Service to expose the deployment internally and provide load balancing between the two pods.

Apply:

kubectl apply -f service.yaml

Check:

kubectl get svc
7. Ingress Controller

The NGINX Ingress Controller was used as the reverse proxy.

Check ingress namespace resources:

kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
8. Ingress Resource

The app was exposed using an Ingress resource for:

flask.local

Example apply:

kubectl apply -f ingress.yaml

Check:

kubectl get ingress
kubectl describe ingress flask-ingress
9. Local Access with Port Forwarding
HTTP
kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80

Application URL:

http://flask.local:8080
HTTPS
kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8443:443

Application URL:

https://flask.local:8443
10. HTTPS Configuration

A self-signed certificate was created for flask.local.

Generated files used during the lab included:

flask.local.crt
flask.local.key or flask.local-key.pem
flask.local.pem
flask.local.pfx

The TLS secret was created in Kubernetes:

kubectl create secret tls flask-local-tls --cert="$HOME\flask.local.pem" --key="$HOME\flask.local-key.pem"

If the secret already existed:

kubectl delete secret flask-local-tls
kubectl create secret tls flask-local-tls --cert="$HOME\flask.local.pem" --key="$HOME\flask.local-key.pem"
Ingress TLS configuration

The Ingress was updated to use TLS and force HTTPS redirect.

Example:

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flask-ingress
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - flask.local
      secretName: flask-local-tls
  rules:
    - host: flask.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: flask-service
                port:
                  number: 80

Apply:

kubectl apply -f ingress.yaml
11. Load Balancing Verification

Load balancing was verified by sending repeated requests and observing different pod IDs / responses.

We used browser refresh and ApacheBench stress tests.

Example:

ab -n 10000 -c 10 http://flask.local:8080/

Later HTTPS was also used:

ab -n 500 -c 10 https://flask.local:8443/

Result:

requests were distributed across the replicas
the application remained available
no failures were observed in the successful test phase
12. Monitoring with Helm, Prometheus, and Grafana
12.1 Add Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
12.2 Create monitoring namespace
kubectl create namespace monitoring
12.3 Install kube-prometheus-stack
helm install kube-prom-stack prometheus-community/kube-prometheus-stack -n monitoring

If the namespace did not exist yet, another valid form is:

helm install kube-prom-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
12.4 Check monitoring pods
kubectl get pods -n monitoring
kubectl get pods -n monitoring -w
13. Grafana Access

Port-forward Grafana:

kubectl port-forward -n monitoring svc/kube-prom-stack-grafana 3000:80

Get the admin password in PowerShell:

kubectl get secret -n monitoring kube-prom-stack-grafana -o jsonpath="{.data.admin-password}" | ForEach-Object {[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_))}

Open:

http://localhost:3000

Login:

username: admin
password: result of the previous command

In Grafana, dashboards such as Kubernetes node/pod resource dashboards were used to observe metrics.

14. Prometheus Access

List services first if needed:

kubectl get svc -n monitoring

Prometheus service was accessed by port-forwarding.
If port 9090 was already used locally, another local port was used such as 9091.

Example:

kubectl port-forward -n monitoring svc/kube-prom-stack-kube-prome-prometheus 9091:9090

Open:

http://localhost:9091

Useful pages:

http://localhost:9091/alerts
http://localhost:9091/rules

Useful query:

ALERTS

or:

ALERTS{alertname="TestEmailAlert"}
15. Alertmanager Access

Port-forward Alertmanager:

kubectl port-forward -n monitoring svc/kube-prom-stack-kube-prome-alertmanager 9094:9093

Open:

http://localhost:9094

This was used to verify that the test alert reached Alertmanager.

16. Alertmanager Email Configuration

An email receiver was configured using AlertmanagerConfig.

Example file:

apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: email-alerts
  namespace: monitoring
  labels:
    alertmanagerConfig: email-alerts
spec:
  route:
    receiver: email-me
    groupBy: ["alertname"]
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 1h
  receivers:
    - name: email-me
      emailConfigs:
        - to: "RECEIVER_EMAIL"
          from: "SENDER_GMAIL"
          smarthost: "smtp.gmail.com:587"
          authUsername: "SENDER_GMAIL"
          authIdentity: "SENDER_GMAIL"
          authPassword:
            name: alertmanager-email-secret
            key: smtp-password
          requireTLS: true

Apply:

kubectl apply -f alertmanager-email.yaml
Email password secret

A Gmail App Password was stored in a Kubernetes secret:

kubectl create secret generic alertmanager-email-secret -n monitoring --from-literal=smtp-password="YOUR_GMAIL_APP_PASSWORD"

If it already existed:

kubectl delete secret alertmanager-email-secret -n monitoring
kubectl create secret generic alertmanager-email-secret -n monitoring --from-literal=smtp-password="YOUR_GMAIL_APP_PASSWORD"
Make Alertmanager select the config

Because the selector was empty, Helm was used to update the Alertmanager selector:

helm upgrade kube-prom-stack prometheus-community/kube-prometheus-stack `
  -n monitoring `
  --reuse-values `
  --set alertmanager.alertmanagerSpec.alertmanagerConfigSelector.matchLabels.alertmanagerConfig=email-alerts `
  --set alertmanager.alertmanagerSpec.alertmanagerConfigNamespaceSelector.matchLabels.kubernetes\.io/metadata\.name=monitoring

Check:

kubectl get alertmanager -n monitoring -o yaml | Select-String alertmanagerConfigSelector
kubectl get alertmanager -n monitoring -o yaml | Select-String alertmanagerConfigNamespaceSelector
17. Test Alert Rule

A custom Prometheus rule was created to trigger a test alert.

test-alert-rule.yaml:

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: test-email-alert
  namespace: monitoring
  labels:
    release: kube-prom-stack
spec:
  groups:
    - name: test-email-alert-group
      rules:
        - alert: TestEmailAlert
          expr: vector(1)
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "Test email alert"
            description: "This is a test alert from kube-prometheus-stack."

Apply:

kubectl apply -f test-alert-rule.yaml

Check rules:

kubectl get prometheusrule -n monitoring
kubectl describe prometheusrule test-email-alert -n monitoring

Prometheus query used to verify:

ALERTS{alertname="TestEmailAlert"}

Observed states:

pending
later visible in Alertmanager UI
18. Useful Debug Commands
General Kubernetes
kubectl get pods
kubectl get svc
kubectl get ingress
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl get events -n monitoring --sort-by=.metadata.creationTimestamp
Check specific monitoring pods
kubectl describe pod -n monitoring -l app.kubernetes.io/name=grafana
kubectl describe pod -n monitoring prometheus-kube-prom-stack-kube-prome-prometheus-0
Alertmanager logs
kubectl logs -n monitoring statefulset/alertmanager-kube-prom-stack-kube-prome-alertmanager
Prometheus logs
kubectl logs -n monitoring statefulset/prometheus-kube-prom-stack-kube-prome-prometheus
Check generated Alertmanager config
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl get secret -n monitoring alertmanager-kube-prom-stack-kube-prome-alertmanager-generated -o jsonpath="{.data.alertmanager\.yaml}")))
Check local port conflicts on Windows
netstat -ano | findstr :9090
taskkill /PID <PID> /F
19. Problems Encountered and Fixes
1. OpenSSL not recognized on Windows
Cause: OpenSSL was not installed or not in PATH
Fix: used Windows/PowerShell certificate generation workflow and other local certificate handling tools
2. monitoring namespace not found
Cause: Helm install attempted before namespace creation
Fix:
kubectl create namespace monitoring
3. Grafana pod in Pending / ContainerCreating
Cause: stack was still starting
Fix: waited for pods to become running and checked pod status
4. Wrong Prometheus service name
Cause: tried to port-forward a non-existing service name
Fix: listed monitoring services first and used the correct one
5. Port 9090 already in use
Cause: local port conflict
Fix: used another local port such as 9091
6. AlertmanagerConfig version issue
Cause: v1beta1 did not match installed CRD
Fix: used:
apiVersion: monitoring.coreos.com/v1alpha1
7. Test alert not visible at first
Cause: rule needed correct label for Helm release selection
Fix: added:
labels:
  release: kube-prom-stack
8. Email not received
Cause: Alertmanager selector/config merge and email delivery validation still required
Fix attempts included:
verifying selector
verifying generated config
checking logs
using a Gmail App Password
20. Final Result

The lab was successfully reproduced in Kubernetes with the same logical architecture as the original VM-based lab:

reverse proxy via NGINX Ingress
load balancing with a Service across 2 replicas
HTTPS enabled with TLS secret
monitoring using Prometheus and Grafana
alerting configured with Alertmanager and custom alert rules

Main achievements:

application deployed successfully
load balancing verified
HTTPS working
Prometheus and Grafana working
alert rule visible in Prometheus and Alertmanager
21. Commands Summary
Deployment
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
Ingress forwarding
kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80
kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8443:443
Monitoring install
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install kube-prom-stack prometheus-community/kube-prometheus-stack -n monitoring
Grafana
kubectl port-forward -n monitoring svc/kube-prom-stack-grafana 3000:80
kubectl get secret -n monitoring kube-prom-stack-grafana -o jsonpath="{.data.admin-password}" | ForEach-Object {[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_))}
Prometheus
kubectl port-forward -n monitoring svc/kube-prom-stack-kube-prome-prometheus 9091:9090
Alertmanager
kubectl port-forward -n monitoring svc/kube-prom-stack-kube-prome-alertmanager 9094:9093
Alerting
kubectl apply -f alertmanager-email.yaml
kubectl apply -f test-alert-rule.yaml
helm upgrade kube-prom-stack prometheus-community/kube-prometheus-stack `
  -n monitoring `
  --reuse-values `
  --set alertmanager.alertmanagerSpec.alertmanagerConfigSelector.matchLabels.alertmanagerConfig=email-alerts `
  --set alertmanager.alertmanagerSpec.alertmanagerConfigNamespaceSelector.matc