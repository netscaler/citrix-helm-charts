# Deploy Citrix cloud native solution on RedHat OpenShift 

You can deploy Citrix cloud native solution in a service mesh on RedHat OpenShift cluster. To deploy Citrix cloud native solution on RedHat OpenShift cluster, you must perform the following tasks:

1. Install the OpenShift CLI
1. Install Helm
1. Install Istio service Mesh
1. Configure Grafana and Prometheus
1. Secure applications
1. Install Citrix ADC Observability Exporter
1. Install Citrix ADM agent
1. Install ingress Gateway
1. Install Sidecar injector
1. Deploy applications

Perform the following tasks to deploy Citrix cloud native solution in Istio service mesh on RedHat OpenShift cluster:

## Install the OpenShift CLI

You can install the OpenShift CLI (oc) in order to interact with OpenShift Container Platform from a command-line interface.

Access the installation resource archive at [RedHat Download](https://access.redhat.com/downloads/content/290/ver=4.7/rhel---8/4.7.18/x86_64/product-software) and extract the archive. After extracting the archive, move the `oc` and `kubectl` binaries at a location on the PATH such as `/usr/local/bin`.

check the oc version using the following command:

    sudo oc version

## Install Helm

Helm is a package manager for Kubernetes that contains information for installing and upgrading Kubernetes resources into a Kubernetes cluster. Helm charts encapsulate YAML definitions, provide a mechanism for configuration.

For information about installing Helm package manager version 3, see [Helm Installation](https://github.com/citrix/citrix-helm-charts/blob/master/Helm_Installation_version_3.md).

## Install Istio Service Mesh

To install the Istio service mesh, perform the following steps:

 1. Download the Istio 1.9.2 or later from [Download Istio](https://istio.io/latest/docs/setup/getting-started/#download).
 2. Install Istio on OpenShift cluster. For installation instructions, see [OpenShift](https://istio.io/latest/docs/setup/platform-setup/openshift/).
 
        curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.9.2 TARGET_ARCH=x86_64 sh -
        cd istio-1.9.2
        sudo cp ./bin/istioctl /usr/local/bin/istioctl
        chmod +x /usr/local/bin/istioctl
        oc adm policy add-scc-to-group anyuid system:serviceaccounts:istio-system
        securitycontextconstraints.security.openshift.io/anyuid added to groups: ["system:serviceaccounts:istio-system"]
        istioctl install --set profile=openshift                                      
        oc -n istio-system expose svc/istio-ingressgateway --port=http2
        route.route.openshift.io/istio-ingressgateway exposed:

## Configure Grafana and Prometheus

To configure Prometheus and Grafana, run the following commands:

    oc apply -f https://raw.githubusercontent.com/istio/istio/release-1.9/samples/addons/prometheus.yaml

    oc apply -f https://raw.githubusercontent.com/istio/istio/release-1.9/samples/addons/grafana.yaml

    oc get pods -n istio-system
    NAME READY STATUS RESTARTS AGE
    grafana-784c89f4cf-x95p7 1/1 Running 0 79m
    istio-ingressgateway-7cc49dcd99-df8cp 1/1 Running 0 19h
    istiod-db9f9f86-qvrnw 1/1 Running 0 19h
    prometheus-7bfddb8dbf-f4v6c 2/2 Running 0 80m

## Security constraints for application

Here need to give scc permission to the serviceaccount

**Note**: Replace <target-namespace> with the appropriate namespace.

For Citrix projects, target names are :`citrix-system` and `bookinfo`. 

When adding your application, add the permissions as follows:  

    oc adm policy add-scc-to-group anyuid system:serviceaccounts:<target-namespace>
    oc adm policy add-scc-to-group privileged  system:serviceaccounts:<target-namespace>

When removing your application, remove the permissions as follows:

    oc adm policy remove-scc-from-group anyuid system:serviceaccounts:<target-namespace>
    oc adm policy remove-scc-from-group privileged system:serviceaccounts:<target-namespace>

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

## Configure Citrix ADM Agent onboarding and Citrix ADC Observability Exporter

1. To configure Citrix ADC Observability Exporter, run the following commands:

        helm repo add citrix https://citrix.github.io/citrix-helm-charts/
 
        helm install coe citrix/citrix-observability-exporter --namespace citrix-system --set timeseries.enabled=true

2. To install ADM agent onboarding, run the following commands:

        
3. To disable mTLS for Citrix ADC Observability Exporter and ADMIP, run the following commands:
      
        kubectl create secret -n citrix-system generic admadaptor --from-literal=accessid=<accessid> --from-literal=accesssecret=<secret_id>

        helm install citrix-adm adm-agent-onboarding --namespace citrix-system
        oc label ns citrix-system citrix-cpx=enabled
      
## Deploy Ingress Gateway with Citrix ADM

To deploy Citrix ADC CPX as ingress gateway, run the following commands:

    helm install citrix-adc-istio-ingress-gateway citrix-adc-istio-ingress-gateway --namespace citrix-system --set ingressGateway.EULA=YES --set citrixCPX=true  --set xDSAdaptor.image=quay.io/ajeetas/xds-adaptor:0.9.8.master  --set ADMSettings.ADMIP=10.131.0.144  --set coe.coeURL=coe.citrix-system --set ingressGateway.secretVolumes[0].name=httpbin-ingressgateway-certs,ingressGateway.secretVolumes[0].secretName=httpbin-ingressgateway-certs,ingressGateway.secretVolumes[0].mountPath=/etc/istio/httpbin-ingressgateway-certs

    helm install citrix-adc-istio-ingress-gateway citrix-adc-istio-ingress-gateway --namespace citrix-system --set ingressGateway.EULA=YES --set citrixCPX=true  --set xDSAdaptor.image=quay.io/ajeetas/xds-adaptor:0.9.8.master  --set ADMSettings.ADMIP=<adm-ip>  --set coe.coeURL=coe.citrix-system --set ingressGateway.secretVolumes[0].name=httpbin-ingressgateway-certs,ingressGateway.secretVolumes[0].secretName=httpbin-ingressgateway-certs,ingressGateway.secretVolumes[0].mountPath=/etc/istio/httpbin-ingressgateway-certs

**Expose Prometheus and Grafana**:

To expose Prometheus and Grafana on the port, run the following command:

    helm install citrix-adc-istio-ingress-gateway citrix-adc-istio-ingress-gateway --namespace citrix-system --set ingressGateway.EULA=YES --set citrixCPX=true --set xDSAdaptor.image=quay.io/ajeetas/xds-adaptor:0.9.8.latest --set ADMSettings.ADMIP=10.128.0.99 --set coe.coeURL=coe.citrix-system --set ingressGateway.secretVolumes[0].name=httpbin-ingressgateway-certs,ingressGateway.secretVolumes[0].secretName=httpbin-ingressgateway-certs,ingressGateway.secretVolumes[0].mountPath=/etc/istio/httpbin-ingressgateway-certs --set ingressGateway.tcpPort[0].name=tcp1,ingressGateway.tcpPort[0].nodePort=30900,ingressGateway.tcpPort[0].port=9090,ingressGateway.tcpPort[0].targetPort=9090 --set ingressGateway.tcpPort[1].name=tcp2,ingressGateway.tcpPort[1].nodePort=30300,ingressGateway.tcpPort[1].port=3000,ingressGateway.tcpPort[1].targetPort=3000

  **Note**: For `ADMSettings.ADMIP=<adm-ip>`, specify Pod IP address of adm-agent. 

## Deploy Citrix ADC CPX Sidecar injector

To deploy Citrix ADC CPX Sidecar injector, run the following commands:

    helm install cpx-sidecar-injector citrix-cpx-istio-sidecar-injector --namespace citrix-system --set cpxProxy.EULA=YES --set xDSAdaptor.image=quay.io/ajeetas/xds-adaptor:0.9.8.master  --set coe.coeURL=coe.citrix-system  --set ADMSettings.ADMIP=10.131.0.144

    helm install cpx-sidecar-injector citrix-cpx-istio-sidecar-injector --namespace citrix-system --set cpxProxy.EULA=YES --set xDSAdaptor.image=quay.io/ajeetas/xds-adaptor:0.9.8.master  --set coe.coeURL=coe.citrix-system  --set ADMSettings.ADMIP=<adm-ip>

    helm install cpx-sidecar-injector citrix-cpx-istio-sidecar-injector --namespace citrix-system --set cpxProxy.EULA=YES --set xDSAdaptor.image=quay.io/ajeetas/xds-adaptor:0.9.8.latest --set coe.coeURL=coe.citrix-system --set ADMSettings.ADMIP=10.128.0.99

  **Note**: For 'ADMSettings.ADMIP=<adm-ip>', specify Pod IP address of adm-agent. 

## Deploy Applications

  The following example shows the deployment of a sample application`httpbin`. 

    oc create ns httpbin
    oc label ns httpbin cpx-injection=enabled
    openssl genrsa -out httpbin_key.pem 2048
    openssl req -new -key httpbin_key.pem -out httpbin_csr.pem -subj "/CN=www.httpbin.com"
    openssl x509 -req -in httpbin_csr.pem -sha256 -days 365 -extensions v3_ca -signkey httpbin_key.pem -CAcreateserial -out httpbin_cert.pem
    kubectl create -n citrix-system secret tls httpbin-ingressgateway-certs --key httpbin_key.pem --cert httpbin_cert.pem

    kubectl create -f examples/servicemesh_with_coe_and_adm/manifest/httpbin.yaml -n httpbin

    kubectl create -f  examples/servicemesh_with_coe_and_adm/manifest/httpbin_secure_gateway.yaml -n httpbin

  The following example shows the deployment of a sample application `bookinfo`.

    cert creation command for bookinfo:
    oc create ns bookinfo 
    oc label ns bookinfo cpx-injection=enabled
    openssl genrsa -out bookinfo_key.pem 2048
    openssl req -new -key bookinfo_key.pem -out bookinfo_csr.pem -subj "/CN=www.bookinfo.com"
    openssl x509 -req -in bookinfo_csr.pem -sha256 -days 365 -extensions v3_ca -signkey bookinfo_key.pem -CAcreateserial -out bookinfo_cert.pem

    kubectl create -n citrix-system secret tls citrix-ingressgateway-certs --key bookinfo_key.pem --cert bookinfo_cert.pem

    kubectl apply -n bookinfo -f https://raw.githubusercontent.com/citrix/citrix-helm-charts/master/examples/servicemesh_with_coe_and_adm/manifest/bookinfo.yaml  
    kubectl apply -n bookinfo -f https://raw.githubusercontent.com/citrix/citrix-helm-charts/master/examples/citrix-adc-in-istio/bookinfo/deployment-yaml/bookinfo_https_gateway.yaml
    kubectl apply -n bookinfo -f https://raw.githubusercontent.com/citrix/citrix-helm-charts/master/examples/citrix-adc-in-istio/bookinfo/deployment-yaml/bookinfo_http_gateway.yaml

    kubectl apply -n bookinfo -f https://raw.githubusercontent.com/citrix/citrix-helm-charts/master/examples/citrix-adc-in-istio/bookinfo/deployment-yaml/productpage_vs.yaml

