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

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/f020007f-666a-401f-b7a3-4c1d3d9787c0/935b21eb-f577-4871-9889-f647a6125e83/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/f020007f-666a-401f-b7a3-4c1d3d9787c0/de791723-5078-4297-81fb-959c7c87f4d2/Untitled.png)

<aside>
💡 Now Let’s deploy the testing app & use our ingress gateway.

</aside>

1. Create the new namespace “istio-in-action”.
2. Inside the **istio-in-action** namespace, deploy the sample-apps using the codes from “[hellocloud-native-box/istio-cop/1-start-istio/sample-apps at main · hellocloudio/hellocloud-native-box (github.com)](https://github.com/hellocloudio/hellocloud-native-box/tree/main/istio-cop/1-start-istio/sample-apps)”.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/f020007f-666a-401f-b7a3-4c1d3d9787c0/b0320ed3-4c2d-42d2-820f-90af0b56f4e9/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/f020007f-666a-401f-b7a3-4c1d3d9787c0/e3e8dc55-2541-4dee-b6af-111505419b90/Untitled.png)

1. Inside the **istio-in-action** namespace, create gw & vs to route traffic from external to internal services. 

- web-api-gw.yaml  (GW is only for listening the traffic.)
    
    (Make sure the selector is correctly mentioned to Istio Ingress GW label.)
    
    ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/f020007f-666a-401f-b7a3-4c1d3d9787c0/78be1865-318b-4277-aa3a-98a8dbf3ad2a/Untitled.png)
    
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
    

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/f020007f-666a-401f-b7a3-4c1d3d9787c0/78be1865-318b-4277-aa3a-98a8dbf3ad2a/Untitled.png)

Verify the GW & VS are created. 

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/f020007f-666a-401f-b7a3-4c1d3d9787c0/8ef4551d-0b24-41c4-a238-27859097e55c/Untitled.png)

Verify the web-api has the service created.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/f020007f-666a-401f-b7a3-4c1d3d9787c0/8669c8b6-ed53-4d6a-8d79-9e09dd50d55b/Untitled.png)

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
