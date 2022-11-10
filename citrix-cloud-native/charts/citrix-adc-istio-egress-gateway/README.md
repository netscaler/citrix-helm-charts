# Deploy Citrix ADC as an egress Gateway in Istio environment using Helm charts

Citrix Application Delivery Controller (ADC) can be deployed as an Istio egress Gateway to control the egress traffic to Istio service mesh.

# Table of Contents
1. [TL; DR;](#tldr)
2. [Introduction](#introduction)
3. [Deploy Citrix ADC VPX or MPX as an Egress Gateway](#deploy-citrix-adc-vpx-or-mpx-as-an-egress-gateway)
4. [Deploy Citrix ADC CPX as an Egress Gateway](#deploy-citrix-adc-cpx-as-an-egress-gateway)
5. [Visualizing statistics of Citrix ADC Egress Gateway with Metrics Exporter](#visualizing-statistics-of-citrix-adc-Egress-gateway-with-metrics-exporter)
6. [Citrix ADC CPX License Provisioning](#citrix-adc-cpx-license-provisioning)
7. [Service Graph Configuration](#configuration-for-servicegraph)
8. [Generate Certificate for Egress Gateway](#generate-certificate-for-egress-gateway)
9. [Citrix ADC as Egress Gateway: a sample deployment](#citrix-adc-as-egress-gateway-a-sample-deployment)
10. [Uninstalling the Helm chart](#uninstalling-the-helm-chart)
11. [Configuration Parameters](#configuration-parameters)

## <a name="tldr">TL; DR;</a>
### To deploy Citrix ADC VPX or MPX as an Egress Gateway:

       kubectl create secret generic nsloginegress --from-literal=username=<citrix-adc-user> --from-literal=password=<citrix-adc-password> -n citrix-system

       helm repo add citrix https://citrix.github.io/citrix-helm-charts/

       helm install citrix-adc-istio-egress-gateway citrix/citrix-cloud-native --namespace citrix-system --set iaEgress.enabled=true,iaEgress.egressGateway.EULA=YES,iaEgress.egressGateway.netscalerUrl=https://<nsip>[:port],iaEgress.egressGateway.vserverIP=<IPv4 Address> --set iaEgress.secretName=nsloginegress

### To deploy Citrix ADC CPX as an Egress Gateway:

       helm repo add citrix https://citrix.github.io/citrix-helm-charts/

       helm install citrix-adc-istio-egress-gateway citrix/citrix-cloud-native --namespace citrix-system --set iaEgress.enabled=true --set iaEgress.egressGateway.EULA=true --set iaEgress.citrixCPX=true

## <a name="introduction">Introduction</a>

    This chart deploys Citrix CPX as an Egress Gateway. An egress gateway defines the exit point from the mesh. It provides features like load balancing at the edge of the mesh, monitoring, and routing rules to exiting the mesh. 

### Compatibility Matrix between Citrix xDS-adaptor and Istio version

Below table provides info about recommended Citrix xDS-Adaptor version to be used for various Istio versions.

| Citrix xDS-Adaptor version | Istio version |
|----------------------------|---------------|
| quay.io/citrix/citrix-xds-adaptor:0.10.3 | Istio v1.14+ |
| quay.io/citrix/citrix-xds-adaptor:0.10.1 | Istio v1.12 to Istio v1.13 |
| quay.io/citrix/citrix-xds-adaptor:0.9.9 | Istio v1.10 to Istio v1.11 |
| quay.io/citrix/citrix-xds-adaptor:0.9.8 | Istio v1.8 to Istio v1.9 |
| quay.io/citrix/citrix-xds-adaptor:0.9.5 | Istio v1.6 |

### Prerequisites

The following prerequisites are required for deploying Citrix ADC as an Egress Gateway in Istio service mesh:

- Ensure that **Istio version 1.8 onwards** is installed
- Ensure that Helm with version 3.x is installed. Follow this [step](https://github.com/citrix/citrix-helm-charts/blob/master/Helm_Installation_version_3.md) to install the same.
- Ensure that your cluster Kubernetes version should be 1.16 onwards and the `admissionregistration.k8s.io/v1`, `admissionregistration.k8s.io/v1beta1` API is enabled

You can verify the API by using the following command:

        kubectl api-versions | grep admissionregistration.k8s.io/v1

The following output indicates that the API is enabled:

        admissionregistration.k8s.io/v1
        admissionregistration.k8s.io/v1beta1

- **For deploying Citrix ADC VPX or MPX as an Egress gateway:**

  Create a Kubernetes secret for the Citrix ADC user name and password using the following command:
  
        kubectl create secret generic nsloginegress --from-literal=username=<citrix-adc-user> --from-literal=password=<citrix-adc-password> -n citrix-system

- **Create system user account for xDS-adaptor in Citrix ADC:**

  The Citrix ADC appliance needs to have system user account (non-default) with certain privileges so that `xDS-adaptor` can configure the Citrix ADC VPX or MPX appliance. Follow the instructions to create the system user account on Citrix ADC.

    Create a Kubernetes secret for the user name and password using the following command:

    ```
       kubectl create secret generic nslogin --from-literal=username='cxa' --from-literal=password='mypassword'
    ```

  The `xDS-adaptor` configures the Citrix ADC using a system user account of the Citrix ADC. The system user account should have certain privileges so that the xDS-adaptor has permissions configure the following on the Citrix ADC:

  -  Add, Delete, or View Content Switching (CS) virtual server
  -  Configure CS policies and actions
  -  Configure Load Balancing (LB) virtual server
  -  Configure Service groups
  -  Cofigure SSl certkeys
  -  Configure routes
  -  Configure user monitors
  -  Add system file (for uploading SSL certkeys from Kubernetes)
  -  Configure Virtual IP address (VIP)
  -  Check the status of the Citrix ADC appliance
  -  Add, Delete or view authentication virtual server, policy, authaction
  -  Add, Delete or view Policy
  -  Add, Delete or view Responder policy, action, param
  -  Add, Delete or view Rewrite policy, action, param
  -  Add, Delete or view analytics profile
  -  Add, Delete or view DNS name server
  -  Add, Delete or view  network netprofile
  -  Add, Delete or view Traffic Management Commands(sessionaction, session policy, sessionparameter)


> **Note:**
>
> The system user account would have privileges based on the command policy that you define.

 To create the system user account, do the following:

 1.  Log on to the Citrix ADC appliance. Perform the following:
     1.  Use an SSH client, such as PuTTy, to open an SSH connection to the Citrix ADC appliance.

     2.  Log on to the appliance by using the administrator credentials.

 2.  Create the system user account using the following command:

     ```
        add system user <username> <password>
     ```

     For example:

     ```
        add system user cxa mypassword
     ```

 3.  Create a policy to provide required permissions to the system user account. Use the following command:

     ```
        add cmdpolicy cxa-policy ALLOW "((^\S+\s+cs\s+\S+)|(^\S+\s+lb\s+\S+)|(^\S+\s+service\s+\S+)|(^\S+\s+servicegroup\s+\S+)|(^stat\s+system)|(^show\s+ha)|(^\S+\s+ssl\s+certKey)|(^\S+\s+ssl)|(^\S+\s+route)|(^\S+\s+monitor)|(^show\s+ns\s+ip)|(^\S+\s+system\s+file)|)|(^\S+\s+aaa\s+\S+)|(^\S+\s+aaa\s+\S+\s+.*)|(^\S+\s+authentication\s+\S+)|(^\S+\s+authentication\s+\S+\s+.*)|(^\S+\s+policy\s+\S+)|(^\S+\s+policy\s+\S+\s+.*)|(^\S+\s+rewrite\s+\S+)|(^\S+\s+rewrite\s+\S+\s+.*)|(^\S+\s+analytics\s+\S+)|(^\S+\s+analytics\s+\S+\s+.*)|(^\S+\s+dns\s+\S+)|(^\S+\s+dns\s+\S+\s+.*)|(^\S+\s+netProfile)|(^\S+\s+netProfile\s+.*)|(^\S+\s+tm\s+\S+)|(^\S+\s+tm\s+\S+\s+.*)"
     ```

 4.  Bind the policy to the system user account using the following command:

     ```
        bind system user cxa cxa-policy 0
     ```

- **Registration of Citrix ADC CPX in ADM**

Create a secret for ADM username and password

        kubectl create secret generic admloginegress --from-literal=username=<adm-username> --from-literal=password=<adm-password> -n citrix-system
        
- **Important Note:** For deploying Citrix ADC VPX or MPX as egress gateway, you should establish the connectivity between Citrix ADC VPX or MPX and cluster nodes. This connectivity can be established by configuring routes on Citrix ADC as mentioned [here](https://github.com/citrix/citrix-k8s-ingress-controller/blob/master/docs/network/staticrouting.md) or by deploying [Citrix Node Controller](https://github.com/citrix/citrix-k8s-node-controller).

## <a name="deploy-citrix-adc-vpx-or-mpx-as-an-egress-gateway">Deploy Citrix ADC VPX or MPX as an egress Gateway</a>

 To deploy Citrix ADC VPX or MPX as an Egress Gateway in the Istio service mesh, do the following step. In this example, release name is specified as `citrix-adc-istio-egress-gateway` and namespace as `citrix-system`.

        kubectl create secret generic nsloginegress --from-literal=username=<citrix-adc-user> --from-literal=password=<citrix-adc-password> -n citrix-system
        
        helm repo add citrix https://citrix.github.io/citrix-helm-charts/

        helm install citrix-adc-istio-egress-gateway citrix/citrix-cloud-native --namespace citrix-system --set iaEgress.enabled=true,iaEgress.egressGateway.EULA=YES,iaEgress.secretName=nsloginegress,iaEgress.egressGateway.netscalerUrl=https://<nsip>[:port],iaEgress.egressGateway.vserverIP=<IPv4 Address> 

## <a name="deploy-citrix-adc-cpx-as-an-egress-gateway">Deploy Citrix ADC CPX as an Egress Gateway</a>

 To deploy Citrix ADC CPX as an egress Gateway, do the following step. In this example, release name is specified as `citrix-adc-istio-egress-gateway` and namespace is used as `citrix-system`.

        helm repo add citrix https://citrix.github.io/citrix-helm-charts/

        helm install citrix-adc-istio-egress-gateway citrix/citrix-cloud-native --namespace citrix-system --set iaEgress.enabled=true --set iaEgress.egressGateway.EULA=true --set iaEgress.citrixCPX=true

## <a name="visualizing-statistics-of-citrix-adc-Egress-gateway-with-metrics-exporter">Visualizing statistics of Citrix ADC Egress Gateway with Metrics Exporter</a>

 By default, [Citrix ADC Metrics Exporter](https://github.com/citrix/citrix-adc-metrics-exporter) is also deployed along with Citrix ADC Egress Gateway    . Citrix ADC Metrics Exporter fetches statistical data from Citrix ADC and exports it to Prometheus running in Istio service mesh. When you add Prometheus as a data source in Grafana, you can visualize this statistical data in the Grafana dashboard.
 
 Metrics Exporter requires the IP address of Citrix ADC CPX as Egress Gateway. It is retrieved from the value specified for `EgressGateway.netscalerUrl`.

 When Citrix ADC CPX is deployed as Egress Gateway, Metrics Exporter runs along with Citrix CPX Egress Gateway in the same pod and specifying IP address is optional.

To deploy Citrix ADC as Egress Gateway without Metrics Exporter, set the value of `metricExporter.required` as false.

         helm repo add citrix https://citrix.github.io/citrix-helm-charts/

         helm install citrix-adc-istio-egress-gateway citrix/citrix-cloud-native --namespace citrix-system --set iaEgress.enabled=true --set iaEgress.egressGateway.EULA=YES  --set iaEgress.citrixCPX=true --set iaEgress.metricExporter.required=false         

 To deploy Citrix ADC VPX or MPX as Egress Gateway without Metrics Exporter, set the value of `iaIngress.metricExporter.required` as false.


        kubectl create secret generic nsloginegress --from-literal=username=<citrix-adc-user> --from-literal=password=<citrix-adc-password> -n citrix-system
    
        helm repo add citrix https://citrix.github.io/citrix-helm-charts/

        helm install citrix-adc-istio-egress-gateway citrix/citrix-cloud-native --namespace citrix-system --set iaEgress.enabled=true,iaEgress.egressGateway.EULA=YES,iaEgress.secretName=nsloginegress,iaEgress.egressGateway.netscalerUrl=https://<nsip>[:port],iaEgress.egressGateway.vserverIP=<IPv4 Address>,iaEgress.metricExporter.required=false                                                                                                                                                                                                                                   
     
   **Note:** To remotely access telemetry addons such as Prometheus and Grafana, see [Remotely Accessing Telemetry Addons](https://istio.io/docs/tasks/telemetry/gateways/).

## <a name="generate-certificate-for-Egress-gateway">Generate Certificate for Egress Gateway </a>

Citrix Egress gateway needs TLS certificate-key pair for establishing secure communication channel with backend applications. Earlier these certificates were issued by Istio Citadel and bundled in Kubernetes secret. Certificate was loaded in the application pod by doing volume mount of secret. Now `xDS-Adaptor` can generate its own certificate and get it signed by the Istio Citadel (Istiod). This eliminates the need of secret and associated [risks](https://kubernetes.io/docs/concepts/configuration/secret/#risks). 

xDS-Adaptor needs to be provided with details Certificate Authority (CA) for successful signing of Certificate Signing Request (CSR). By default, CA is `istiod.istio-system.svc` which accepts CSRs on port 15012. 
To skip this process, don't provide any value (empty string) to `iaEgress.certProvider.caAddr`.

```
        helm repo add citrix https://citrix.github.io/citrix-helm-charts/

        helm install citrix-adc-istio-egress-gateway citrix/citrix-cloud-native --namespace citrix-system --set iaEgress.enabled=true --set iaEgress.egressGateway.EULA=YES --set iaEgress.citrixCPX=true --set iaEgress.certProvider.caAddr=""
```

### <a name="using-third-party-service-account-tokens">Configure Third Party Service Account Tokens</a>

In order to generate certificate for application workload, xDS-Adaptor needs to send valid service account token along with Certificate Signing Request (CSR) to the Istio control plane (Citadel CA). Istio control plane authenticates the xDS-Adaptor using this JWT. 
Kubernetes supports two forms of these tokens:

* Third party tokens, which have a scoped audience and expiration.
* First party tokens, which have no expiration and are mounted into all pods.
 
 If Kubernetes cluster is installed with third party tokens, then the same information needs to be provided for automatic sidecar injection by passing `--set iaEgress.certProvider.jwtPolicy="third-party-jwt"`. By default, it is `first-party-jwt`.

```
        helm repo add citrix https://citrix.github.io/citrix-helm-charts/

        helm install cpx-sidecar-injector citrix/citrix-cpx-istio-sidecar-injector --namespace citrix-system --set iaEgress.cpxProxy.EULA=YES --set iaEgress.certProvider.caAddr="istiod.istio-system.svc" --set iaEgress.certProvider.jwtPolicy="third-party-jwt"

```

To determine if your cluster supports third party tokens, look for the TokenRequest API using below command. If there is no output, then it is `first-party-jwt`. In case of `third-party-jwt`, output will be like below.

```
# kubectl get --raw /api/v1 | jq '.resources[] | select(.name | index("serviceaccounts/token"))'

{
    "name": "serviceaccounts/token",
    "singularName": "",
    "namespaced": true,
    "group": "authentication.k8s.io",
    "version": "v1",
    "kind": "TokenRequest",
    "verbs": [
        "create"
    ]
}

```


## <a name="citrix-adc-cpx-license-provisioning">**Citrix ADC CPX License Provisioning**</a>
By default, CPX runs with 20 Mbps bandwidth called as [CPX Express](https://www.citrix.com/en-in/products/citrix-adc/cpx-express.html) however for better performance and production deployment customer needs licensed CPX instances. [Citrix ADM](https://www.citrix.com/en-in/products/citrix-application-delivery-management/) is used to check out licenses for Citrix ADC CPX.

**Bandwidth based licensing**
For provisioning licensing on Citrix ADC CPX, it is mandatory to provide License Server information to CPX. This can be done by setting **ADMSettings.licenseServerIP** as License Server IP. In addition to this, **ADMSettings.bandWidthLicense** needs to be set true and desired bandwidth capacity in Mbps should be set **ADMSettings.bandWidth**.
For example, to set 2Gbps as bandwidth capacity, below command can be used.

    helm repo add citrix https://citrix.github.io/citrix-helm-charts/

    helm install citrix-adc-istio-egress-gateway citrix/citrix-cloud-native --namespace citrix-system --set iaEgress.enabled=true --set iaEgress.egressGateway.EULA=YES --set iaEgress.ADMSettings.licenseServerIP=<Licenseserver_IP>,iaEgress.ADMSettings.bandWidthLicense=True --set iaEgress.ADMSettings.bandWidth=2000 --set citrixCPX=true

## <a name="configuration-for-servicegraph">**Configure ServiceGraph**</a>
   Citrix ADM Service graph is an observability tool that allows user to analyse service to service communication. The service graph is generated by ADM post collection of transactional data from registered Citrix ADC instances. More details about it can be found [here](https://docs.citrix.com/en-us/citrix-application-delivery-management-service/application-analytics-and-management/service-graph.html).
   Citrix ADC needs to be provided with ADM details for registration and data export. This section lists the steps needed to deploy Citrix ADC and register it with ADM.

**Deploy Citrix ADC CPX as egress gateway**
   1. Create secret using Citrix ADM Agent credentials, which will be used by Citrix ADC CPX to communicate with Citrix ADM Agent:

	kubectl create secret generic admlogin --from-literal=username=<adm-agent-username> --from-literal=password=<adm-agent-password>

   2. Deploy Citrix ADC CPX as egress gateway using helm command with `ADM` details:

	helm install citrix-adc-istio-egress-gateway citrix/citrix-cloud-native --namespace citrix-system --set iaEgress.enabled=true --set iaEgress.egressGateway.EULA=true --set iaEgress.citrixCPX=true --set iaEgress.ADMSettings.ADMIP=<ADM-Agent-IP>

> **Note:**
> If container agent is being used here for Citrix ADM, specify `serviceIP` of container agent in the `iaEgress.ADMSettings.ADMIP` parameter.

**Deploy Citrix ADC VPX/MPX as egress gateway**

   Deploy Citrix ADC VPX/MPX as egress gateway using the following helm command and set analytics settings on Citrix ADC VPX/MPX for sending transaction metrics to Citrix ADM

	helm install citrix-adc-istio-egress-gateway citrix/citrix-cloud-native --namespace citrix-system --set iaEgress.enabled=true,iaEgress.egressGateway.EULA=YES,iaEgress.egressGateway.netscalerUrl=https://<nsip>[:port],iaEgress.egressGateway.vserverIP=<IPv4 Address> --set iaEgress.secretName=nsloginegress

   Add the following configurations in Citrix ADC VPX/MPX

        en ns mode ulfd

        en ns feature appflow

        add appflow collector logproxy_lstreamd -IPAddress <ADM-AGENT-IP/POD-IP> -port 5557 -Transport logstream

        set appflow param -templateRefresh 3600 -httpUrl ENABLED -httpCookie ENABLED -httpReferer ENABLED -httpMethod ENABLED -httpHost ENABLED -httpUserAgent ENABLED -httpContentType ENABLED -httpAuthorization ENABLED -httpVia ENABLED -httpXForwardedFor ENABLED -httpLocation ENABLED -httpSetCookie ENABLED -httpSetCookie2 ENABLED -httpDomain ENABLED -httpQueryWithUrl ENABLED  metrics ENABLED -events ENABLED -auditlogs ENABLED

        add appflow action logproxy_lstreamd -collectors logproxy_lstreamd

        add appflow policy logproxy_policy true logproxy_lstreamd

        bind appflow global logproxy_policy 10 END -type REQ_DEFAULT

        bind appflow global logproxy_policy 10 END -type OTHERTCP_REQ_DEFAULT


> **Note:**
> If container agent is being used here for Citrix ADM, specify `podIP` of container agent in above manual config.


## <a name="citrix-adc-as-egress-gateway-a-sample-deployment">Citrix ADC as Egress Gateway: a sample deployment</a>
A sample deployment of Citrix ADC as an Egress gateway to excess external services is provided [here](https://github.com/citrix/citrix-helm-charts/tree/master/examples/citrix-adc-egress-in-istio).

## <a name="uninstalling-the-helm-chart">Uninstalling the Helm chart</a>
 
 To uninstall or delete a chart with release name as `citrix-adc-istio-egress-gateway`, do the following step.

         helm uninstall citrix-adc-istio-egress-gateway -n citrix-system

 The command removes all the Kubernetes components associated with the chart and deletes the release.

## <a name="configuration-parameters">Configuration parameters</a>

The following table lists the configurable parameters in the Helm chart and their default values.

| Parameter                      | Description                   | Default                   | Optional/Mandatory                  |
|--------------------------------|-------------------------------|---------------------------|---------------------------|
| `iaEgress.citrixCPX`                    | Citrix ADC CPX                    | FALSE                  | Mandatory for Citrix ADC CPX |
| `iaEgress.xDSAdaptor.imageRegistry`                | Image registry of the Citrix xDS adaptor container               | `quay.io`               |     Mandatory                  |
| `iaEgress.xDSAdaptor.imageRepository`              | Image repository of the Citrix xDS adaptor container               | `citrix/citrix-xds-adaptor`               |    Mandatory                  |
| `iaEgress.xDSAdaptor.imageTag`                     | Image tag of the Citrix xDS adaptor container               | `0.10.3`               |   Mandatory                  |
| `iaEgress.xDSAdaptor.imagePullPolicy`   | Image pull policy for xDS adaptor | IfNotPresent       | Optional|
| `iaEgress.xDSAdaptor.secureConnect`     | If this value is set to true, xDS-adaptor establishes secure gRPC channel with Istio Pilot   | TRUE                       | Optional|
| `iaEgress.xDSAdaptor.logLevel`   | Log level to be set for xDS-adaptor log messages. Possible values: TRACE (most verbose), DEBUG, INFO, WARN, ERROR (least verbose) | DEBUG       | Optional|
| `iaEgress.xDSAdaptor.jsonLog`   | Set this argument to true if log messages are required in JSON format | false       | Optional|
| `iaEgress.xDSAdaptor.defaultSSLListenerOn443` | Create SSL vserver by default for LDS resource for 0.0.0.0 and port 443. If set to false, TCP vserver will be created in absence of TLSContext in tcp_proxy filter | true | Optional |
| `iaEgress.coe.coeURL`          | Name of [Citrix Observability Exporter](https://github.com/citrix/citrix-observability-exporter) Service in the form of "<servicename>.<namespace>"  | null            | Optional|
| `iaEgress.coe.coeTracing`          | Use COE to send appflow transactions to Zipkin endpoint. If it is set to true, ADM servicegraph (if configured) can be impacted.  | false           | Optional|
| `iaEgress.ADMSettings.ADMIP`          | Citrix Application Delivery Management (ADM) IP address  | NIL            | Mandatory for Citrix ADC CPX |
| `iaEgress.ADMSettings.licenseServerIP`          | Citrix License Server IP address  | NIL            | Optional |
| `iaEgress.ADMSettings.licenseServerPort` | Citrix ADM port if a non-default port is used                                                                                        | 27000                                                                 | Optional|
| `iaEgress.ADMSettings.bandWidth`          | Desired bandwidth capacity to be set for Citrix ADC CPX in Mbps  | 1000            | Optional |
| `iaEgress.ADMSettings.bandWidthLicense`          | To specify bandwidth based licensing  | false            | Optional |
| `iaEgress.ADMSettings.licenseEdition`| License edition that can be Standard, Platinum and Enterprise . By default, Platinum is selected | PLATINUM | optional |
| `iaEgress.ADMSettings.analyticsServerPort` |      Port used for Analytics in ADM. Required to plot ServiceGraph.                                                                                   | 5557                                                                 | Optional|
| `iaEgress.egressGateway.netscalerUrl`       | URL or IP address of the Citrix ADC which Istio-adaptor configures (Mandatory if citrixCPX=false)| null   |Mandatory for Citrix ADC MPX or VPX|
| `iaEgress.egressGateway.vserverIP`       | Virtual server IP address on Citrix ADC (Mandatory if citrixCPX=false) | null | Mandatory for Citrix ADC MPX or VPX|
| `iaEgress.egressGateway.adcServerName`          | Citrix ADC ServerName used in the Citrix ADC certificate  | NIL            | Optional |
| `iaEgress.egressGateway.imageRegistry`                   | Image registry of Citrix ADC CPX designated to run as egress Gateway              | `quay.io`               |     Mandatory for Citrix ADC CPX                  |
| `iaEgress.egressGateway.imageRepository`                 | Image repository of Citrix ADC CPX designated to run as egress Gateway              | `citrix/citrix-k8s-cpx-ingress`               |    Mandatory for Citrix ADC CPX                  |
| `iaEgress.egressGateway.imageTag`                  | Image tag of Citrix ADC CPX designated to run as egress Gateway              | `13.1-30.52`               |   Mandatory for Citrix ADC CPX                  |
| `iaEgress.egressGateway.imagePullPolicy`   | Image pull policy                                                                                                                  | IfNotPresent                                                          | Optional|
| `iaEgress.egressGateway.EULA`             | End User License Agreement(EULA) terms and conditions. If yes, then user agrees to EULA terms and conditions.                                     | false                                                                    | Mandatory for Citrix ADC CPX
| `iaEgress.egressGateway.mgmtHttpPort`      | Management port of the Citrix ADC CPX                                                                                              | 9080                                                                  | Optional|
| `iaEgress.egressGateway.mgmtHttpsPort`    | Secure management port of Citrix ADC CPX                                                                                           | 9443                                                                  | Optional|
| `iaEgress.egressGateway.label` | Custom label for the egress Gateway service                                                                                       | citrix-ingressgateway                                                                 |Optional|
| `iaEgress.egressGateway.cpxLicenseAggregator` | IP/FQDN of the CPX License Aggregator if it is being used to license the CPX. | null                                                                |Optional|
| `iaEgress.egressGateway.enableLabelsFeature` | If this variable is true, Istio's [subset](https://istio.io/latest/docs/reference/config/networking/destination-rule/#Subset) of the service and some metadata of the service such as servicename, namespace etc will be stored in the Citrix ADC that might be used for analytics purpose.                                                                                   | FALSE                                                                 |Optional|
| `iaEgress.istioPilot.name`                 | Name of the Istio Pilot (Istiod) service                                                                                                        | istiod                                                           |Optional|
| `iaEgress.istioPilot.namespace`     | Namespace where Istio Pilot is running                                                                                        | istio-system                                                          |Optional|
| `iaEgress.istioPilot.secureGrpcPort`       | Secure GRPC port where Istiod (Istio Pilot) is listening (default setting)                                                                  | 15012                                                                 |Optional|
| `iaEgress.istioPilot.insecureGrpcPort`      | Insecure GRPC port where Istio Pilot is listening                                                                                  | 15010                                                                 |Optional|
| `iaEgress.istioPilot.SAN`                 | Subject alternative name for Istio Pilot which is the secure production identity framework for everyone (SPIFFE) ID of Istio Pilot                                                        | null |Optional|
| `iaEgress.metricExporter.required`          | Metrics exporter for Citrix ADC                                                                                                    | TRUE                                                                  |Optional|
| `iaEgress.metricExporter.imageRegistry`                  | Image registry of the Citrix ADC Metrics Exporter               | `quay.io`               |     Optional                  |
| `iaEgress.metricExporter.imageRepository`                | Image repository of the Citrix ADC Metrics Exporter               | `citrix/citrix-adc-metrics-exporter`               |    Optional                  |
| `iaEgress.metricExporter.imageTag`                 | Image tag of the Citrix ADC Metrics Exporter               | `1.4.9`               |   Optional                  |
| `iaEgress.metricExporter.port`              | Port over which Citrix ADC Metrics Exporter collects metrics of Citrix ADC.                                                      | 8888                                                                  |Optional|
| `iaEgress.metricExporter.secure`            | Enables collecting metrics over TLS                                                                                                | YES                                                                    |Optional|
| `iaEgress.metricExporter.logLevel`          | Level of logging in Citrix ADC Metrics Exporter. Possible values are: DEBUG, INFO, WARNING, ERROR, CRITICAL                                       | ERROR                                                                 |Optional|
| `iaEgress.metricExporter.imagePullPolicy`   | Image pull policy for Citrix ADC Metrics Exporter                                                                                       | IfNotPresent 
| `iaEgress.certProvider.caAddr`   | Certificate Authority (CA) address issuing certificate to application                           | istiod.istio-system.svc                          | Optional |
| `iaEgress.certProvider.caPort`   | Certificate Authority (CA) port issuing certificate to application                              | 15012 | Optional |
| `iaEgress.certProvider.trustDomain`   | SPIFFE Trust Domain                         | cluster.local | Optional |
| `iaEgress.certProvider.certTTLinHours`   | Validity of certificate generated by xds-adaptor and signed by Istiod (Istio Citadel) in hours. Default is 30 days validity              | 720 | Optional |
| `iaEgress.certProvider.clusterId`   | clusterId is the ID of the cluster where Istiod CA instance resides (default Kubernetes). It can be different value on some cloud platforms or in multicluster environments. For example, in Anthos servicemesh, it might be of the format of `cn<project-name>-<region>-<cluster_name>`. In multiCluster environments, it is the value of global.multiCluster.clusterName provided during servicemesh control plane installation         | Kubernetes | | Optional |
| `iaEgress.certProvider.jwtPolicy`   | Service Account token type. Kubernetes platform supports First party tokens and Third party tokens. Usually public cloud based Kubernetes has third-party-jwt | null | Optional |
| `iaEgress.secretName`   | Name of the Kubernetes secret holding Citrix ADC credentials | nsloginegress | Mandatory for Citrix ADC VPX/MPX |
