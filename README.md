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



=========================================

Check status:

================ COMMAND ================

minikube status
kubectl get nodes

=========================================

2. Build the Flask Application Image

================ COMMAND ================

docker build -t flask-local-app .

=========================================

If needed for Minikube Docker environment:

================ COMMAND ================

minikube image load flask-local-app

=========================================

3. Deploy the Application

Apply the deployment and service:

================ COMMAND ================

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

=========================================

Verify:

================ COMMAND ================

kubectl get pods
kubectl get svc

=========================================

4. Install NGINX Ingress Controller

================ COMMAND ================

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

=========================================

Verify:

================ COMMAND ================

kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

=========================================

5. Configure Local DNS on Windows

Edit the Windows hosts file and add:

================ TEXT ================

127.0.0.1 flask.local

====================================

Path of the file:

================ TEXT ================

C:\Windows\System32\drivers\etc\hosts

====================================

6. Create and Apply Ingress

Apply the ingress file:

================ COMMAND ================

kubectl apply -f ingress.yaml

=========================================

Verify:

================ COMMAND ================

kubectl get ingress
kubectl describe ingress flask-ingress

=========================================

Port-forward HTTP locally:

================ COMMAND ================

kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80

=========================================

Test in browser:

================ URL ================

http://flask.local:8080

====================================

7. Verify Load Balancing

The Flask app was deployed with 2 replicas.

Check pods:

================ COMMAND ================

kubectl get pods

=========================================

Refresh the browser several times or run a benchmark test and observe alternating pod responses.

Stress test:

================ COMMAND ================

ab -n 10000 -c 10 http://flask.local:8080/

=========================================

Expected result:

requests succeed
responses alternate between replicas
load balancing works correctly
8. Enable HTTPS with a Self-Signed Certificate
8.1 Generate certificate using mkcert

If mkcert is installed:

================ COMMAND ================

mkcert -install
mkcert flask.local

=========================================

This generates files similar to:

================ TEXT ================

flask.local.pem
flask.local-key.pem

====================================

8.2 Create Kubernetes TLS secret

================ COMMAND ================

kubectl create secret tls flask-local-tls --cert="$HOME\flask.local.pem" --key="$HOME\flask.local-key.pem"

=========================================

If the secret already exists:

================ COMMAND ================

kubectl delete secret flask-local-tls
kubectl create secret tls flask-local-tls --cert="$HOME\flask.local.pem" --key="$HOME\flask.local-key.pem"

=========================================

8.3 Update Ingress for TLS

Example ingress.yaml:

================ YAML ================

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

=====================================

Apply again:

================ COMMAND ================

kubectl apply -f ingress.yaml

=========================================

8.4 Port-forward HTTPS

================ COMMAND ================

kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8443:443

=========================================

Test HTTPS:

================ URL ================

https://flask.local:8443

====================================

Or test with curl:

================ COMMAND ================

curl -k https://flask.local:8443

=========================================

9. Install Prometheus and Grafana with Helm

Add the Helm repository:

================ COMMAND ================

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

=========================================

Create namespace:

================ COMMAND ================

kubectl create namespace monitoring

=========================================

Install the monitoring stack:

================ COMMAND ================

helm install kube-prom-stack prometheus-community/kube-prometheus-stack -n monitoring

=========================================

Check pods:

================ COMMAND ================

kubectl get pods -n monitoring
kubectl get svc -n monitoring

=========================================

10. Access Grafana

Port-forward Grafana:

================ COMMAND ================

kubectl port-forward -n monitoring svc/kube-prom-stack-grafana 3000:80

=========================================

Get Grafana admin password:

================ COMMAND ================

kubectl get secret -n monitoring kube-prom-stack-grafana -o jsonpath="{.data.admin-password}" | ForEach-Object {[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_))}

=========================================

Grafana URL:

================ URL ================

http://localhost:3000

====================================

Username:

================ TEXT ================

admin

====================================

In Grafana, dashboards were checked to confirm that metrics were available.

11. Access Prometheus

Port-forward Prometheus:

================ COMMAND ================

kubectl port-forward -n monitoring svc/kube-prom-stack-kube-prome-prometheus 9091:9090

=========================================

Prometheus URL:

================ URL ================

http://localhost:9091

====================================

Targets page:

================ URL ================

http://localhost:9091/targets

====================================

Alerts page:

================ URL ================

http://localhost:9091/alerts

====================================

A few Minikube control-plane targets may remain down. This is common in local single-node clusters and does not prevent the monitoring stack from working.

12. Create a Test Alert Rule

Example test-alert-rule.yaml:

================ YAML ================

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

=====================================

Apply it:

================ COMMAND ================

kubectl apply -f test-alert-rule.yaml

=========================================

Verify in Prometheus query page:

================ QUERY ================

ALERTS{alertname="TestEmailAlert"}

======================================

Possible states:

pending
firing
13. Configure Alertmanager Email Receiver

Create the secret containing the Gmail app password:

================ COMMAND ================

kubectl create secret generic alertmanager-email-secret -n monitoring --from-literal=smtp-password="YOUR_APP_PASSWORD"

=========================================

If the secret already exists:

================ COMMAND ================

kubectl delete secret alertmanager-email-secret -n monitoring
kubectl create secret generic alertmanager-email-secret -n monitoring --from-literal=smtp-password="YOUR_APP_PASSWORD"

=========================================

Example alertmanager-email.yaml:

================ YAML ================

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
        - to: "YOUR_RECEIVER_EMAIL"
          from: "YOUR_GMAIL_ADDRESS"
          smarthost: "smtp.gmail.com:587"
          authUsername: "YOUR_GMAIL_ADDRESS"
          authIdentity: "YOUR_GMAIL_ADDRESS"
          authPassword:
            name: alertmanager-email-secret
            key: smtp-password
          requireTLS: true

=====================================

Apply it:

================ COMMAND ================

kubectl apply -f alertmanager-email.yaml

=========================================

Make Alertmanager select the config:

================ COMMAND ================

helm upgrade kube-prom-stack prometheus-community/kube-prometheus-stack `
  -n monitoring `
  --reuse-values `
  --set alertmanager.alertmanagerSpec.alertmanagerConfigSelector.matchLabels.alertmanagerConfig=email-alerts `
  --set alertmanager.alertmanagerSpec.alertmanagerConfigNamespaceSelector.matchLabels.kubernetes\.io/metadata\.name=monitoring

=========================================

Verify selector:

================ COMMAND ================

kubectl get alertmanager -n monitoring -o yaml | Select-String alertmanagerConfigSelector
kubectl get alertmanager -n monitoring -o yaml | Select-String alertmanagerConfigNamespaceSelector

=========================================

14. Access Alertmanager

Port-forward Alertmanager:

================ COMMAND ================

kubectl port-forward -n monitoring svc/kube-prom-stack-kube-prome-alertmanager 9094:9093

=========================================

Open:

================ URL ================

http://localhost:9094

====================================

When TestEmailAlert becomes firing, it should appear in Alertmanager.

15. Useful Debug Commands
General Kubernetes checks

================ COMMAND ================

kubectl get pods
kubectl get svc
kubectl get ingress
kubectl get pods -n monitoring
kubectl get svc -n monitoring

=========================================

Check monitoring stack

================ COMMAND ================

kubectl get pods -n monitoring -w
kubectl get events -n monitoring --sort-by=.metadata.creationTimestamp

=========================================

Check Prometheus rules

================ COMMAND ================

kubectl get prometheusrule -n monitoring
kubectl describe prometheusrule test-email-alert -n monitoring

=========================================

Check Alertmanager logs

================ COMMAND ================

kubectl logs -n monitoring statefulset/alertmanager-kube-prom-stack-kube-prome-alertmanager

=========================================

Check generated Alertmanager config

================ COMMAND ================

[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl get secret -n monitoring alertmanager-kube-prom-stack-kube-prome-alertmanager-generated -o jsonpath="{.data.alertmanager\.yaml}")))

=========================================

Check Prometheus query manually

================ QUERY ================

ALERTS
ALERTS{alertname="TestEmailAlert"}

======================================

Find port conflicts on Windows

================ COMMAND ================

netstat -ano | findstr :9090
taskkill /PID <PID> /F

=========================================

16. Results

The following objectives were achieved:

Flask application deployed successfully on Kubernetes
Deployment scaled to 2 replicas
Service used as internal load balancer
NGINX Ingress Controller exposed the application
HTTPS enabled with a self-signed certificate
Prometheus installed and scraping metrics
Grafana installed and visualizing metrics
Alertmanager configured and tested with a custom alert rule
17. Notes
On Windows, port-forwarding was used because direct port 80/443 access may be blocked.
On Minikube, some control-plane monitoring targets may stay down. This is normal in local lab environments.
For Gmail SMTP, an App Password is required instead of the normal Gmail password.