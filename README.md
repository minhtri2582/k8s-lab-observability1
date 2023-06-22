Observability using Prometheus and Grafana
Introduction
In this lab step, you'll deploy a sample API and API client to generate data points and metrics that will be collected by Prometheus and displayed within Grafana. Using Helm, you'll be required to deploy both Prometheus and Grafana. Prometheus is an open-source systems monitoring and alerting service. Grafana is an open source analytics and interactive visualization web application, providing charts, graphs, and alerts for monitoring and observability requirements. You'll configure Grafana to connect to Prometheus as a data source. Once connected, you'll import and deploy 2 prebuilt Grafana dashboards. You'll configure Prometheus to perform automatic service discovery of both the sample API, and the EKS cluster itself. Prometheus once configured, will then automatically start collecting various metrics from both the sample API and the EKS cluster.
Notes:
•	The Prometheus web admin interface will be exposed over the Internet on port 80, allowing you to access it from your own workstation.
•	The Grafana web admin interface will be exposed over the Internet on port 80, allowing you to access it from your own workstation. 
Instructions  
1. Install tools and download the application code.
1.1 Install the helm util. In the terminal run the following command:
Copy code
{
curl -sLO https://get.helm.sh/helm-v3.7.1-linux-amd64.tar.gz
tar -xvf helm-v3.7.1-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin
}
 
1.2 Download the observability application code and scripts. In the terminal run the following commands:
Copy code
{
mkdir -p ~/cloudacademy/observability
cd ~/cloudacademy/observability
curl -sL https://api.github.com/repos/cloudacademy/k8s-lab-observability1/releases/latest | \
 jq -r '.zipball_url' | \
 wget -qi -
unzip release* && find -type f -name *.sh -exec chmod +x {} \;
cd cloudacademy-k8s-lab-observability*
tree
}
A tree view of the downloaded project directory structure is provided:
 
2. Install the sample API and API client cluster resources. 
Note: All of the source code as used within this lab is located online here:
https://github.com/cloudacademy/k8s-lab-observability1
2.1. Before installing, examine the source code that makes up the Python based API and API client. 
2.1.1. The ./code/api directory contains 3 files which have been used to build the cloudacademydevops/api-metrics container image.
Open each of the following files within your editor of choice.
•	api.py
•	Dockerfile
•	requirements.txt
The api.py file contains the Python source code which implements the example API.
In particular take note of the following:
•	Line 5 - imports a PromethusMetrics module to automatically generate Flask based metrics and provide them for collection at the default endpoint /metrics
•	Line 10-32 - implements 5 x API endpoints: 
o	/one
o	/two
o	/three
o	/four
o	/error
•	All example endpoints, except for the error endpoint, introduce a small amount of latency which will be measured and observed within both Prometheus and Grafana.
•	The error endpoint returns an HTTP 500 server error response code, which again will be measured and observed within both Prometheus and Grafana.
•	The Docker container image containing this source code has already been built using the tag cloudacademydevops/api-metrics
2.1.2 ./code/k8s directory contains the api.yaml Kubernetes manifest file used to deploy the API into the cluster. 
Open each of the following files within your editor of choice.
•	api.yaml
In particular note the following:
•	Lines 1-25: API Deployment containing 2 pods
•	Line 22: Pods are based off the container image cloudacademydevops/api-metrics
•	Lines 27-46: API Service - load balances traffic across the 2 API Deployment pods
•	Lines 34-37: API Service is annotated to ensure that the Prometheus scraper will automatically discover the API pods behind it. Prometheus will then collect their metrics from the discovered targets
2.2. The API and API client need to be deployed into the cloudacademy namespace which needs to be created. Create the new namespace. In the terminal execute the following command:
Copy code
kubectl create ns cloudacademy
2.3. Navigate into the k8s directory and perform a directory listing. In the terminal execute the following commands:
Copy code
cd ./code/k8s && ls -la
 
2.4. Deploy the sample API into the newly created namespace. In the terminal execute the following command
Copy code
kubectl apply -f ./api.yaml
2.5. Deploy the sample API client into the newly created namespace. In the terminal execute the following command
Copy code
kubectl run api-client \
 --namespace=cloudacademy \
 --image=cloudacademydevops/api-generator \
 --env="API_URL=http://api-service:5000" \
 --image-pull-policy IfNotPresent
2.6. Wait for the sample API and API client resources to start up. In the terminal run the following commands:
Copy code
{
kubectl wait --for=condition=available --timeout=300s deployment/api -n cloudacademy
kubectl wait --for=condition=ready --timeout=300s pod/api-client -n cloudacademy
}
2.7. Confirm that the sample API and API client cluster resources have all started successfully. In the terminal run the following commands:
Copy code
kubectl get all -n cloudacademy 
 
3. Install Prometheus into the cluster.
3.1. Prometheus will be deployed into the prometheus namespace which needs to be created. Create a new namespace named prometheus. In the terminal run the follwing command:
Copy code
kubectl create ns prometheus
3.2 Navigate into the prometheus directory and perform a directory listing. In the terminal execute the following commands:
Copy code
cd ../prometheus && ls -la
 
3.3. With Helm, install Prometheus using it's publicly available Helm Chart. Deploy Prometheus into the prometheus namespace. In the terminal execute the following commands:
Copy code
{
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus \
 --namespace prometheus \
 --set alertmanager.persistentVolume.storageClass="gp2" \
 --set server.persistentVolume.storageClass="gp2" \
 --values ./prometheus.values.yaml
}
 
3.4. Expose the Prometheus web application. In the terminal run the following commands:
Copy code
kubectl expose deployment prometheus-server \
 --namespace prometheus \
 --name=prometheus-server-loadbalancer \
 --type=LoadBalancer \
 --port=80 \
 --target-port=9090
3.5. Wait for the Prometheus application to start up. In the terminal run the following commands:
Copy code
{
kubectl wait --for=condition=available --timeout=300s deployment/prometheus-kube-state-metrics -n prometheus
kubectl wait --for=condition=available --timeout=300s deployment/prometheus-prometheus-pushgateway -n prometheus
kubectl wait --for=condition=available --timeout=300s deployment/prometheus-server -n prometheus
}
3.6. Confirm that the Prometheus cluster resources have all started successfully. In the terminal run the following commands:
Copy code
kubectl get all -n prometheus
 
3.7. Retrieve the Prometheus web application URL.
3.7.1. Confirm that the Prometheus ELB FQDN has propageted and resolves. In the terminal run the following commands:
Copy code
{
PROMETHEUS_ELB_FQDN=$(kubectl get svc -n prometheus prometheus-server-loadbalancer -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
until nslookup $PROMETHEUS_ELB_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl -sD - -o /dev/null $PROMETHEUS_ELB_FQDN/graph
}
Note: DNS propagation can take up to 2-5 minutes, please be patient while the propagation proceeds - it will eventually complete.
3.7.2. Generate the Prometheus web application URL:
Copy code
echo http://$PROMETHEUS_ELB_FQDN
3.8. Using your workstation's browser, copy the Prometheus URL from the previous output and browse to it:
 
3.9. Within the Prometheus web application, click the Status top menu item and then select Service Discovery:
 
3.10. Confirm that the following service discovery targets have been configured:
 
3.11. Within Prometheus, click the Status top menu item and then select Targets:
 
3.12. Confirm that the following Targets are available:
 
4. Install Grafana into the cluster. Grafana will be deployed into the grafana namespace which needs to be created.
4.1. Navigate into the grafana directory and perform a directory listing. In the terminal execute the following commands:
Copy code
cd ../grafana && ls -la
 
4.2. Create a new namespace named grafana. In the terminal run the follwing command:
Copy code
kubectl create ns grafana
4.3. install Grafana using it's publicly available Helm Chart. Deploy Grafana into the grafana namespace. In the terminal execute the following commands:
Copy code
{
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana \
 --namespace grafana \
 --set persistence.storageClassName="gp2" \
 --set persistence.enabled=true \
 --set adminPassword="EKS:l3t5g0" \
 --set service.type=LoadBalancer \
 --values ./grafana.values.yaml
}
 
Note: The PodSecurityPolicy deprecation warnings can be safely ignored.
4.4. Wait for the Grafana application to start up. In the terminal run the following command:
Copy code
kubectl wait --for=condition=available --timeout=300s deployment/grafana -n grafana 
4.5. Confirm that the Grafana cluster resources have all started successfully. In the terminal run the following commands:
Copy code
kubectl get all -n grafana
 
4.6 Retrieve the Grafana web application URL.
4.6.1. Confirm that the Grafana ELB FQDN has propagated and resolves. In the terminal run the following commands:
Copy code
{
GRAFANA_ELB_FQDN=$(kubectl get svc -n grafana grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
until nslookup $GRAFANA_ELB_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl -I $GRAFANA_ELB_FQDN/login
}
Note: DNS propagation can take up to 2-5 minutes, please be patient while the propagation proceeds - it will eventually complete.
4.6.2. Generate the Grafana web application URL:
Copy code
echo http://$GRAFANA_ELB_FQDN
 
4.7. Using your workstation's browser, copy the Grafana URL from the previous output and browse to it:
 
4.8. Login into Grafana, using the following credentials:
Email or username: admin
Password: EKS:l3t5g0
 
4.9. Once authenticated, the Welcome to Grafana home page is displayed:
 
4.10. Hover over the Dashboards icon on the main left-hand side menu, and then select the dashboard Import option like so:
 
4.11. In the Import pane, enter the dashboard ID 3119 and click on the right hand side Load button:
 
4.12. The Import pane now provides details for the Kubernetes cluster monitoring dashboard that you are about to install. Select and set the Prometheus datasource at the bottom of the pane, then click Import to proceed:
 
4.13. Grafana now loads and displays the prebuilt Kubernetes cluster monitoring dashboard and immediately renders various visualisations using live monitoring data streams taken from Prometheus. The dashboard view automatically refreshes every 10 seconds:
 
4.14. Return to the Dashboards icon (+) on the main left-hand side menu, and again select the dashboard Import option like so:
 
4.15. Open a new browser tab and navigate to the following sample API monitoring dashboard URL:
https://raw.githubusercontent.com/cloudacademy/k8s-lab-observability1/main/code/grafana/dashboard.json
Note: The same dashboard.json file is located in the project directory (./code/grafana/dashboard.json) and can be copied directly from there if easier.
4.16 Copy the sample API monitoring dashboard JSON to the local clipboard:
 
4.17. In the Import pane, paste the contents of the local clipboard containing the sample API monitoring dashboard configuration JSON into the Import via panel json input box. Click the bottom Load button to import the dashboard:
 
4.18. The Import pane now provides details for the sample API monitoring dashboard that you are about to install. Click the Import button to proceed:
 
4.19. Grafana now loads and displays the prebuilt sample CloudAcademy DevOps 2021 API dashboard and immediately renders various visualisations using live monitoring data streams taken from Prometheus. The dashboard view automatically refreshes every 5 seconds:
 
5. Bonus steps
5.1. Try terminating the API client pod which is used to generate traffic sent to the API. Once terminated return to the API monitoring dashboard and confirm that the traffic tails off to zero.
5.2. Restart the API client pod and then confirm within the API monitoring dashboard, that traffic is again being captured and recorded.
5.3 Try scaling up the API Deployment resource. Confirm the effects of doing so within the API monitoring dashboard.
 
Summary
In this lab step, you installed a sample API and API client. Next, using Helm you installed Prometheus into the prometheus namespace within the cluster. You set up and exposed the Prometheus web admin interface using a LoadBalancer based Service resource. You then browsed to the Prometheus web admin interface and confirmed that service discovery was working correctly, and the expected targets had been registered. Using Helm you then installed Grafana into the grafana namespace. You then authenticated into the Grafana web admin interface and setup 2 dashboards, one for the cluster, and another one for the sample API you previously deployed. Grafana then loaded the dashboards and starting pulling live monitoring data and metrics directly from the Prometheus data source. 
 
![image](https://github.com/minhtri2582/k8s-lab-observability1/assets/6768446/90f7f746-601a-49ad-b8d5-9d450e12069f)
