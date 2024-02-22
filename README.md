# Install Istio & Demostrate Istio IngressGateway
In this lab, we will test belows - 

1. Use asdf tool to manage our Istio versions. 
2. Install Istio using profile.
3. Use Istio Ingress Gateway to listen on external traffic & route to our destination services.

Before install Istio, we need to make sure Istio is compatible with our Kubernetes cluster version.  

https://istio.io/latest/docs/releases/supported-releases/     

```yaml
cd $HOME
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.12.0

###Update .bashrc (or) .bash_profile as below to add
. $HOME/.asdf/asdf.sh
. $HOME/.asdf/completions/asdf.bash
###

source .bash_profile

asdf plugin-add istioctl https://github.com/virtualstaticvoid/asdf-istioctl.git

# Click here to check istio releases and download the istioctl version you want.
https://github.com/istio/istio/tags

asdf install istioctl 1.18.2
asdf install istioctl 1.17.5
asdf install istioctl 1.16.7

asdf list
#############
istioctl
  1.16.7
  1.17.5
  1.18.2
#############

# to use 1.18.2
asdf global istioctl 1.18.2

# verify
istioctl version
```

- List the Istio configuration profiles.

```yaml
vagrant@kindcluster-box:~$ istioctl profile list
Istio configuration profiles:
    ambient
    default
    demo
    empty
    external
    minimal
    openshift
    preview
    remote
```

- Install Istio profile “Demo”.

```yaml
vagrant@kindcluster-box:~$ istioctl install --set profile=demo -y
✔ Istio core installed
  Processing resources for Istiod. Waiting for Deployment/istio-system/istiod                                 ✔ Istio cor✔ Istio core installed
  Processing resources for Istiod. Waiting for Deployment/istio-system/istiod

✔ Istiod installed
- Processing resources for Egress gateways, Ingress gateways. Waiting for Deployment/istio-system/istio-egress✔ Egress gateways installed
✔ Ingress gateways installed
- Pruning removed resources                                                                                     Removed HorizontalPodAutoscaler:istio-system:istio-ingressgateway.
  Removed HorizontalPodAutoscaler:istio-system:istiod.
✔ Installation complete                                                                                       Making this installation the default for injection and validation.

Thank you for installing Istio 1.17.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/hMHGiwZHPU7UQRWe9
```

- Once installed, check the new Istio related namespace, pods, services, deployments, and replicas.
- Check the Istio supported CRDs.
- Core components are different based on the profile.

![image](https://github.com/myathway-lab/Istio-IngressGateway/assets/157335804/adf25be3-a591-4430-af5d-7a9fe6d82d8d)


![image](https://github.com/myathway-lab/Istio-IngressGateway/assets/157335804/8f2bde50-efae-4e44-bd0b-46d39476f4e6)


<aside>
💡 Now Let’s deploy the testing app & use our ingress gateway.

</aside>

1. Create the new namespace “istio-in-action”.
2. Inside the **istio-in-action** namespace, deploy the sample-apps using the codes from “[hellocloud-native-box/istio-cop/1-start-istio/sample-apps at main · hellocloudio/hellocloud-native-box (github.com)](https://github.com/hellocloudio/hellocloud-native-box/tree/main/istio-cop/1-start-istio/sample-apps)”.

![image](https://github.com/myathway-lab/Istio-IngressGateway/assets/157335804/b6b02916-7818-4528-9261-3e92e8bf1ec0)
![image](https://github.com/myathway-lab/Istio-IngressGateway/assets/157335804/f458fadf-987a-4f2c-a4b2-5ef4d308c783)


1. Inside the **istio-in-action** namespace, create gw & vs to route traffic from external to internal services. 

- web-api-gw.yaml  (GW is only for listening the traffic.)
    
(Make sure the selector is correctly mentioned to Istio Ingress GW label.)
        
    ```yaml
    apiVersion: networking.istio.io/v1beta1
    kind: Gateway   
    metadata:
      name: web-api-gateway
    spec:
      selector:  #Have to mention which "IngressGateway" will be used by virtual GW.
        **istio: ingressgateway** #kubectl get svc -n istio-system --show-labels
      servers:
      - port:
          number: 80   #VGW will listen on traffic with port 80
          name: http
          protocol: HTTP
        hosts:
        - "hellocloud.io"  #VGW will listen on traffic with hellocloud.io hostname
    ```
![image](https://github.com/myathway-lab/Istio-IngressGateway/assets/157335804/347bbe13-fc8d-4921-b7b9-fbfc9404fffa)    

- web-api-gw-vs.yaml (Decide how to route the traffic coming from GW to destination services.)
    
    ```yaml
    apiVersion: networking.istio.io/v1beta1
    kind: VirtualService
    metadata:
      name: web-api-gw-vs
    spec:
      hosts:
      - "hellocloud.io"
      gateways:
      - web-api-gateway  #Choose virtual GW that Virtual service will use. 
      http:
      - route:
        - destination:
            host: web-api.istio-in-action.svc.cluster.local #use the web-api app's svc name
            port:
              number: 8080 
    ```
    

![image](https://github.com/myathway-lab/Istio-IngressGateway/assets/157335804/38eee715-cd4f-4486-99a3-05a83009eea6)


Verify the GW & VS are created. 

![image](https://github.com/myathway-lab/Istio-IngressGateway/assets/157335804/3121eb35-8a27-4870-ad4e-58d7670dce8c)


Verify the web-api has the service created.

![image](https://github.com/myathway-lab/Istio-IngressGateway/assets/157335804/c304a221-630f-40a5-bc3c-afadc90b7645)


1. Once we created the GW & VS, we will verify end user is able to call [hellocloud.io](http://hellocloud.io) by using IngressGateway External IP. 

172.18.255.150 = Ingress GW External-IP

```
curl -H "Host: hellocloud.io" http://172.18.255.150:80
```

1. It is success. It means users are able to access Ingress gateway which is activated by GW. Then GW is listening on incoming traffic which is hellocloud.io:80. Then VS is activated and routed to destination service “web-api”.

```yaml
curl -H "Host: hellocloud.io" http://172.18.255.150:80
{
  "name": "web-api",
  "uri": "/",
  "type": "HTTP",
  "ip_addresses": [
    "10.239.1.7"
  ],
  "start_time": "2024-02-05T14:48:00.091404",
  "end_time": "2024-02-05T14:48:00.126858",
  "duration": "35.453702ms",
  "body": "Hello From Web API",
  "upstream_calls": [
    {
      "name": "recommendation",
      "uri": "http://recommendation:8080",
      "type": "HTTP",
      "ip_addresses": [
        "10.239.1.5"
      ],
      "start_time": "2024-02-05T14:48:00.107664",
      "end_time": "2024-02-05T14:48:00.123121",
      "duration": "15.456611ms",
      "body": "Hello From Recommendations!",
      "upstream_calls": [
        {
          "name": "purchase-history-v1",
          "uri": "http://purchase-history:8080",
          "type": "HTTP",
          "ip_addresses": [
            "10.239.1.6"
          ],
          "start_time": "2024-02-05T14:48:00.116526",
          "end_time": "2024-02-05T14:48:00.118797",
          "duration": "2.271165ms",
          "body": "Hello From Purchase History (v1)!",
          "code": 200
        }
      ],
      "code": 200
    }
  ],
  "code": 200
}
```

5) Uninstall the istio.

```yaml
istioctl uninstall --purge
```
