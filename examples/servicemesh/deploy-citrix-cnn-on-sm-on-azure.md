# Deploy Citrix cloud native solution on Azure Kubernetes Service

You can deploy Citrix cloud native solution in a service mesh on Azure Kubernetes Service (AKS) cluster. To deploy Citrix cloud native solution on Azure Kubernetes Service cluster, you must perform the following tasks:

1. Deploy Citrix ADC VPX on Azure
1. Install Istio service mesh
1. Certifcate generation for applications
1. Configure Citrix ADM Agent and Citrix ADC Observability Exporter
1. Deploy ingress gateway
1. Deploy Citrix ADC CPX Sidecar injector
1. Deploy Applications

**Prerequisites**:

  You must have installed Azure Kubernetes Service cluster. For information about installing AKS, see [Deploy an Azure Kubernetes Service cluster using the Azure CLI](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough).

  Perform the following tasks:

## Deploy Citrix ADC VPX on Azure

1. Deploy Citrix ADC VPX in High Availability INC mode as ingress for Azure Kubernetes Services. For information about deployment, see [Citrix ADC VPX in High Availability INC mode as ingress for Azure Kubernetes Services](https://github.com/citrix/citrix-k8s-ingress-controller/blob/master/docs/deploy/deploy-vpx-ha-inc-on-azure.md).

2. Set up inbound and outbound rules on Azure. Ensure that the appropriate inbound and outbound rules are configured on the Azure portal before deploying Citrix ADC VPX Ingress Gateway. For information on configuring inbound and outbound rules, see [AKS with Istio](https://info.citrite.net/pages/viewpage.action?pageId=1505704742).

## Install Istio service mesh

To install the Istio service mesh, perform the following steps:

1. Install the Istio v1.8.3 or later in AKS using the following commands:

        curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.8.3 TARGET_ARCH=x86_64 sh -
        cd istio-1.8.3
        sudo cp ./bin/istioctl /usr/local/bin/istioctl
        sudo chmod +x /usr/local/bin/istioctl
        istioctl install --set profile=demo -y

    For uninstalling Istio, use the following command:

        istioctl manifest generate --set profile=demo | kubectl delete -f -

2. Install Grafana and Prometheus in the `istio-system` namespace using the following commands:

        kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/addons/prometheus.yaml

        kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/addons/grafana.yaml

    Verify that Prometheus and Grafana are running and operational. For information, see [Prometheus and Grafana](https://github.com/citrix/citrix-helm-charts/tree/master/examples/servicemesh_with_coe_and_adm#h-deploy-gateway-for-prometheus-and-grafana).

3. Expose Prometheus and Grafana through Citrix ADC VPX Ingress Gateway using the following commands:

        kubectl apply -f https://raw.githubusercontent.com/citrix/citrix-helm-charts/master/examples/servicemesh_with_coe_and_adm/manifest/prometheus_gateway.yaml

        kubectl apply -f https://raw.githubusercontent.com/citrix/citrix-helm-charts/master/examples/servicemesh_with_coe_and_adm/manifest/grafana_gateway.yaml

## Certifcate generation for applications

  The following steps shows certificate generation for sample applications: `bookinfo` and `httpbin`.

  For `bookinfo` application;

      kubectl create ns bookinfo
 
      kubectl label ns bookinfo cpx-injection=enabled
 
      openssl genrsa -out bookinfo_key.pem 2048
 
      openssl req -new -key bookinfo_key.pem -out bookinfo_csr.pem -subj "/CN=www.bookinfo.com"
 
      openssl x509 -req -in bookinfo_csr.pem -sha256 -days 365 -extensions v3_ca -signkey bookinfo_key.pem -CAcreateserial -out bookinfo_cert.pem
 
      kubectl create -n citrix-system secret tls citrix-ingressgateway-certs --key bookinfo_key.pem --cert bookinfo_cert.pem

  Secret for `httpbin`:

      kubectl create ns httpbin
      kubectl label ns httpbin cpx-injection=enabled
      openssl genrsa -out httpbin_key.pem 2048

      openssl req -new -key httpbin_key.pem -out httpbin_csr.pem -subj "/CN=www.httpbin.com"
 
      openssl x509 -req -in httpbin_csr.pem -sha256 -days 365 -extensions v3_ca -signkey httpbin_key.pem -CAcreateserial -out httpbin_cert.pem
 
      kubectl create -n citrix-system secret tls httpbin-ingressgateway-certs --key httpbin_key.pem --cert httpbin_cert.pem

## Configure Citrix ADM Agent and Citrix ADC Observability Exporter

1. To set up Citrix ADC Observability Exporter, run the following commands:

        helm repo add citrix https://citrix.github.io/citrix-helm-charts/
 
        helm install coe citrix/citrix-observability-exporter --namespace citrix-system --set timeseries.enabled=true

2. To onboard ADM agent, run the following commands:

        wget  https://raw.githubusercontent.com/citrix/citrix-helm-charts/master/generate_token.py

        python generate_token.py --accessID=<accessID> --accessSecret=<accessSecret>

        helm repo add citrix https://citrix.github.io/citrix-helm-charts
        helm install citrix-adm citrix/adm-agent-onboarding --namespace citrix-system --set token=<Token>

3. To disable mTLS for Citrix ADC Observability Exporter and ADMIP, run the following command:

        kubectl apply -n citrix-system -f https://raw.githubusercontent.com/citrix/citrix-helm-charts/master/examples/servicemesh_with_coe_and_adm/manifest/destinationrule_agent_coe.yaml

## Deploy Ingress Gateway

  **Deploy Citrix ADC CPX as Ingress Gateway**:

To deploy Citrix ADC CPX Ingress Gateway, use the following command:

    helm repo add citrix https://citrix.github.io/citrix-helm-charts/

     helm install cpx-ingress-gateway citrix/citrix-adc-istio-ingress-gateway --namespace citrix-system --set ingressGateway.EULA=YES,citrixCPX=true,ingressGateway.label=cpx_ingressgateway --set ingressGateway.secretVolumes[0].name=httpbin-ingressgateway-certs,ingressGateway.secretVolumes[0].secretName=httpbin-ingressgateway-certs,ingressGateway.secretVolumes[0].mountPath=/etc/istio/httpbin-ingressgateway-certs --set ADMSettings.ADMIP=10.42.145.25

  To include Citrix ADC Observability Exporter configuration in the deployment, specify the following:

    --set coe.coeURL=coe.citrix-system

  To include Prometheus and Grafana configurations in the deployment, specify the following:

    --set ingressGateway.tcpPort[0].name=tcp1,ingressGateway.tcpPort[0].nodePort=30900,ingressGateway.tcpPort[0].port=9090,ingressGateway.tcpPort[0].targetPort=9090 --set ingressGateway.tcpPort[1].name=tcp2,ingressGateway.tcpPort[1].nodePort=30300,ingressGateway.tcpPort[1].port=3000,ingressGateway.tcpPort[1].targetPort=3000

  To include Citrix ADM configuration in the deployment, specify the following:

    --set ADMSettings.ADMIP=<ADM POD IP>

  **Deploy Citrix ADC VPX as Ingress Gateway**

To deploy Citrix ADC CPX Ingress Gateway, use the following command:

    kubectl create secret generic nslogin --from-literal=username=<citrix-adc-user> --from-literal=password=<citrix-adc-password> -n citrix-system

     helm repo add citrix https://citrix.github.io/citrix-helm-charts/

     helm install citrix-adc-istio-ingress-gateway citrix/citrix-adc-istio-ingress-gateway --namespace citrix-system --set ingressGateway.EULA=YES --set ingressGateway.netscalerUrl=https://10.106.172.146 --set ingressGateway.vserverIP=10.106.172.148 --set secretName=nslogin --set ingressGateway.secretVolumes[0].name=httpbin-ingressgateway-certs,ingressGateway.secretVolumes[0].secretName=httpbin-ingressgateway-certs,ingressGateway.secretVolumes[0].mountPath=/etc/istio/httpbin-ingressgateway-certs --set ingressGateway.EULA=YES

  To include the Citrix ADC Observability Exporter configuration in the deployment, specify the following:

    --set coe.coeURL=coe.citrix-system

  To include the Prometheus and Grafana configuration in the deployment, specify the following:

    --set ingressGateway.tcpPort[0].name=tcp1,ingressGateway.tcpPort[0].port=9090,ingressGateway.tcpPort[0].targetPort=9090 --set ingressGateway.tcpPort[1].name=tcp2,ingressGateway.tcpPort[1].port=3000,ingressGateway.tcpPort[1].targetPort=3000

  To include the Analytics configuration in the deployment, specify the following:

    en ns mode ulfd
  
    en ns feature appflow
  
    add appflow collector logproxy_lstreamd -IPAddress <ADM-AGENT-POD-IP> -port 5557 -Transport logstream

    set appflow param -templateRefresh 3600 -httpUrl ENABLED -httpCookie ENABLED -httpReferer ENABLED -httpMethod ENABLED -httpHost ENABLED -httpUserAgent ENABLED -httpContentType ENABLED -httpAuthorization ENABLED -httpVia ENABLED -httpXForwardedFor ENABLED -httpLocation ENABLED -httpSetCookie ENABLED -httpSetCookie2 ENABLED -httpDomain ENABLED -httpQueryWithUrl ENABLED  metrics ENABLED -events ENABLED -auditlogs ENABLED
  
    add appflow action logproxy_lstreamd -collectors logproxy_lstreamd
  
    add appflow policy logproxy_policy true logproxy_lstreamd
  
    bind appflow global logproxy_policy 10 END -type REQ_DEFAULT 
  
    bind appflow global logproxy_policy 10 END -type OTHERTCP_REQ_DEFAULT

## Deploy Citrix ADC CPX Sidecar injector

To deploy Citrix ADC CPX Sidecar injector, run the following commands:

    helm install cpx-sidecar-injector citrix/citrix-cpx-istio-sidecar-injector --namespace citrix-system --set cpxProxy.EULA=YES --set ADMSettings.ADMIP=10.42.145.25

  To include the Citrix ADC Observability Exporter configuration in the deployment, specify the following:

    --set coe.coeURL=coe.citrix-system

  To include the Citrix ADM configuration in the deployment, specify the following:

    --set ADMSettings.ADMIP=<ADM POD IP>

## Deploy Applications

Deploy the Sidecar in application namespace using the YAML file. See the YAML in this location: [Resource](https://raw.githubusercontent.com/priyankash-citrix/citrix-helm-charts/master/examples/servicemesh_with_coe_and_adm/manifest/resource.yaml).

1. Deploy sample applications `bookinfo` and `httpbin` using the following commands:

    kubectl apply -n bookinfo -f https://raw.githubusercontent.com/citrix/citrix-helm-charts/master/examples/servicemesh_with_coe_and_adm/manifest/bookinfo.yaml  

    kubectl apply -n httpbin -f https://raw.githubusercontent.com/citrix/citrix-helm-charts/master/examples/servicemesh_with_coe_and_adm/manifest/httpbin.yaml

    kubectl apply -n bookinfo -f https://raw.githubusercontent.com/citrix/citrix-helm-charts/master/examples/citrix-adc-in-istio/bookinfo/deployment-yaml/productpage_vs.yaml

    kubectl  apply -n bookinfo -f https://raw.githubusercontent.com/citrix/citrix-helm-charts/master/examples/citrix-adc-in-istio/bookinfo/deployment-yaml/bookinfo_https_gateway.yaml

    kubectl apply -n bookinfo -f https://raw.githubusercontent.com/citrix/citrix-helm-charts/master/examples/citrix-adc-in-istio/bookinfo/deployment-yaml/bookinfo_http_gateway.yaml

    kubectl  apply -n httpbin -f https://raw.githubusercontent.com/priyankash-citrix/citrix-helm-charts/master/examples/servicemesh_with_coe_and_adm/manifest/httpbin_secure_gateway.yaml

2. Expose `httpbin` and `bookinfo` application through Citrix ADC VPX Ingress Gateway using the following commands:

    kubectl apply -n httpbin https://raw.githubusercontent.com/priyankash-citrix/citrix-helm-charts/master/examples/servicemesh_with_coe_and_adm/manifest/httpbin_gateway.yaml

    kubectl apply -n bookinfo https://raw.githubusercontent.com/priyankash-citrix/citrix-helm-charts/master/examples/servicemesh_with_coe_and_adm/manifest/bookinfo_gateway.yaml

2. Expose `httpbin` and `bookinfo` application through Citrix ADC CPX Ingress Gateway using the following yaml files:

```yml
  apiVersion: networking.istio.io/v1alpha3
  kind: Gateway
  metadata:
    name: cpx-bookinfo-gateway
  spec:
    selector:
      app: cpx_ingressgateway
    servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
        privateKey: /etc/istio/ingressgateway-certs/tls.key
      hosts:
      - "www.bookinfo.com"
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
      - "www.bookinfo.com"
  ---
  # Source: bookinfo-citrix-ingress/templates/bookinfo_https_gateway.yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: Gateway
  metadata:
    name: bookinfo-gateway
  spec:
    selector:
      app: citrix-ingressgateway
    servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
        privateKey: /etc/istio/ingressgateway-certs/tls.key
      hosts:
      - "www.bookinfo.com"
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
      - "www.bookinfo.com"

  ---
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: productpage
  spec:
    hosts:
    - "www.bookinfo.com"
    gateways:
    - cpx-bookinfo-gateway
    - bookinfo-gateway
    http:
    - match:
      - uri:
          exact: /productpage
      - uri:
          prefix: /
      route:
      - destination:
          host: productpage
  ---

  apiVersion: networking.istio.io/v1alpha3
  kind: Gateway
  metadata:
    name: cpx-httpbin-gateway
  spec:
    selector:
      app: cpx_ingressgateway
    servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        serverCertificate: /etc/istio/httpbin-ingressgateway-certs/tls.crt
        privateKey: /etc/istio/httpbin-ingressgateway-certs/tls.key
      hosts:
      - "www.httpbin.com"
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
      - "www.httpbin.com"
  ---
  apiVersion: networking.istio.io/v1alpha3
  kind: Gateway
  metadata:
    name: httpbin-gateway
  spec:
    selector:
      app: citrix-ingressgateway
    servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        serverCertificate: /etc/istio/httpbin-ingressgateway-certs/tls.crt
        privateKey: /etc/istio/httpbin-ingressgateway-certs/tls.key
      hosts:
      - "www.httpbin.com"
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
      - "www.httpbin.com"
  ---

  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: httpbin
  spec:
    hosts:
    - "www.httpbin.com"
    gateways:
    - httpbin-gateway
    - cpx-httpbin-gateway
    http:
    - route:
      - destination:
          host: httpbin
          port:
            number: 8000
  ---
```