---
title: Public and Private Istio Ingress Gateways on AWS
date: 2019-08-30 12:00:00
categories: [kubernetes, aws]
tags: [kubernetes,aws,istio,gateways]
---

![Public and Private Istio Ingress Gateways on AWS](/assets/img/2019/09/dusk.jpeg)

By default, istio creates a service with a publicly accessible classic load balancer (ELB). However, there are times where we only want access from our internal network or a network we are connected to via a secure VPN.

To achieve this we need a copy of our current ingressgateway **service** and **deployment** configuration.

The command below will output our current configuration to a file:

```shell
kubectl get svc istio-ingressgateway -n istio-system -o yaml > istio-pvt-ingressgateway.yaml
```

Open the file in your favourite text editor and get ready to make some changes to the file. You can remove all the auto-generated fields. What you will need to add to create an NLB is the annotation `service.beta.kubernetes.io/aws-load-balancer-type: "nlb"`. If however if you prefer the classic load balancer (ELB), then just remove this annotation. The next annotation we need to add is `service.beta.kubernetes.io/aws-load-balancer-internal: "true"`. This annotation will ensure that this load balancer is only available to other services within your VPC. This is great for microservices that do not need to be exposed to public traffic, or for access on your company intranet connected via VPN to your VPC.

What you will need to change in the file is the following:

1. change the `name` to anything you want: I changed it from `istio-ingressgateway` to `istio-pvt-ingressgateway`

2. change the `app` label to anything you want: I changed it from `istio-ingressgateway` to `istio-pvt-ingressgateway`

3. change the `istio` label to anything you want: I changed it from `ingressgateway` to `pvt-ingressgateway`

4. in the `selector` configuration section update the `app` and `istio` label to match the values you defined in the metadata `labels` section

5. remove all the `nodePort` values from the `ports` configuration so that the newly created service allocated the ports automatically

6. now is your chance to add on any additional ports that you want to use

```yml
apiVersion: v1
kind: Service
metadata:
  annotations:    
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
  labels:
    app: istio-pvt-ingressgateway
    chart: gateways
    heritage: Tiller
    istio: pvt-ingressgateway
    release: istio
  name: istio-pvt-ingressgateway
  namespace: istio-system
spec:
  ports:
  - name: status-port
    port: 15020
    protocol: TCP
    targetPort: 15020
  - name: http2
    port: 80
    protocol: TCP
    targetPort: 80
  - name: https
    port: 443
    protocol: TCP
    targetPort: 443
  - name: smpp
    port: 2775
    protocol: TCP
    targetPort: 2775
  - name: tcp
    port: 31400
    protocol: TCP
    targetPort: 31400
  - name: https-kiali
    port: 15029
    protocol: TCP
    targetPort: 15029
  - name: https-prometheus
    port: 15030
    protocol: TCP
    targetPort: 15030
  - name: https-grafana
    port: 15031
    protocol: TCP
    targetPort: 15031
  - name: https-tracing
    port: 15032
    protocol: TCP
    targetPort: 15032
  - name: tls
    port: 15443
    protocol: TCP
    targetPort: 15443
  selector:
    app: istio-pvt-ingressgateway
    istio: pvt-ingressgateway
    release: istio
  type: LoadBalancer
```

Now save and apply that file:
```shell
kubectl apply -f istio-pvt-ingressgateway.yaml
```

Wait for your load balancer to become operational. After a few minutes, you can check the status of the load balancer and creation:

```shell
kubectl get svc -n istio-system | grep istio-pvt-ingressgateway
```

You should see the DNS name of your load balancer and you can log into your AWS and verify that the load balancer created is indeed an NLB where Type is `network` and that the Scheme is `internal`.

Now feel free to create any domain/subdomain in Route53 and point it to your load balancer DNS as an A record alias.

The next step would be to create a copy of the `istio-ingressgateway` **deployment** and change it to facilitate the new service we have created:

```shell
kubectl get deployment/istio-ingressgateway -n istio-system -o yaml > istio-pvt-ingressgateway-deployment.yaml
```

Open up the newly created `file` in a text editor and remove the standard fields that are automatically created (clean up the file a bit, you don’t have to).

1. We then need to change the `name` to something a bit more custom: I changed mine to match the service which is `name: istio-pvt-ingressgateway`

2. We need to change the `app` labels as well to `app: istio-pvt-ingressgateway`

3. We need to change the `istio` labels to `istio: pvt-ingressgateway`

4. Feel free to add on any additional `ports` you need to be defined.

```yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: istio-pvt-ingressgateway
    chart: gateways
    heritage: Tiller
    istio: pvt-ingressgateway
    release: istio
  name: istio-pvt-ingressgateway
  namespace: istio-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: istio-pvt-ingressgateway
      istio: pvt-ingressgateway
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "false"
      labels:
        app: istio-pvt-ingressgateway
        chart: gateways
        heritage: Tiller
        istio: pvt-ingressgateway
        release: istio
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - preference:
              matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - amd64
            weight: 2
          - preference:
              matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - ppc64le
            weight: 2
          - preference:
              matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - s390x
            weight: 2
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - amd64
                - ppc64le
                - s390x
              - key: kubelet.kubernetes.io/role
                operator: In
                values:
                - management
      containers:
      - args:
        - proxy
        - router
        - --domain
        - $(POD_NAMESPACE).svc.cluster.local
        - --log_output_level=default:info
        - --drainDuration
        - 45s
        - --parentShutdownDuration
        - 1m0s
        - --connectTimeout
        - 10s
        - --serviceCluster
        - istio-ingressgateway
        - --zipkinAddress
        - zipkin:9411
        - --proxyAdminPort
        - "15000"
        - --statusPort
        - "15020"
        - --controlPlaneAuthPolicy
        - NONE
        - --discoveryAddress
        - istio-pilot:15010
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        - name: POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: INSTANCE_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
        - name: HOST_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.hostIP
        - name: ISTIO_META_POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: ISTIO_META_CONFIG_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: ISTIO_META_ROUTER_MODE
          value: sni-dnat
        image: gcr.io/istio-release/proxyv2:master-latest-daily
        imagePullPolicy: IfNotPresent
        name: istio-proxy
        ports:
        - containerPort: 15020
          protocol: TCP
        - containerPort: 80
          protocol: TCP
        - containerPort: 443
          protocol: TCP
        - containerPort: 31400
          protocol: TCP
        - containerPort: 15029
          protocol: TCP
        - containerPort: 15030
          protocol: TCP
        - containerPort: 15031
          protocol: TCP
        - containerPort: 15032
          protocol: TCP
        - containerPort: 15443
          protocol: TCP
        - containerPort: 15090
          name: http-envoy-prom
          protocol: TCP
        readinessProbe:
          failureThreshold: 30
          httpGet:
            path: /healthz/ready
            port: 15020
            scheme: HTTP
          initialDelaySeconds: 1
          periodSeconds: 2
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: "2"
            memory: 1Gi
          requests:
            cpu: 100m
            memory: 128Mi
        volumeMounts:
        - mountPath: /etc/certs
          name: istio-certs
          readOnly: true
        - mountPath: /etc/istio/ingressgateway-certs
          name: ingressgateway-certs
          readOnly: true
        - mountPath: /etc/istio/ingressgateway-ca-certs
          name: ingressgateway-ca-certs
          readOnly: true
      serviceAccount: istio-ingressgateway-service-account
      serviceAccountName: istio-ingressgateway-service-account
      tolerations:
      - effect: NoSchedule
        key: dedicated
        operator: Equal
        value: management
      volumes:
      - name: istio-certs
        secret:
          defaultMode: 420
          optional: true
          secretName: istio.istio-ingressgateway-service-account
      - name: ingressgateway-certs
        secret:
          defaultMode: 420
          optional: true
          secretName: istio-ingressgateway-certs
      - name: ingressgateway-ca-certs
        secret:
          defaultMode: 420
          optional: true
          secretName: istio-ingressgateway-ca-certs
```

Now we want to apply that deployment:

```shell
kubectl apply -f istio-pvt-ingressgateway-deployment.yaml
```

You can now check to ensure that the pod is running:

```shell
kubectl get pods -n istio-system | grep istio-pvt-ingressgateway
```

Next, create an istio gateway configuration and ensure that the `selector` is set to what we created earlier on in the private gateway service. In my case it was `istio: pvt-ingressgateway`. This selector will use that label to be used with our service configuration.

```yml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: myapp-gateway
spec:
  selector:
    istio: pvt-ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "myapp.mydomain.com"
```

Apply the file:

```
kubectl apply -f gateway.yaml
```

Now we can just follow the normal convention of deploying the rest of the istio configuration and kubernetes configuration for our application.

## Bonus — rest of the configuration for an istio based application

Once we have a gateway configuration setup we now need a virtual service. The virtual service acts as a firewall to your application. In the config, you can define some really amazing routing rules and whitelisting/blacklisting.

Here is an example config for a pretty standard HTTP application:

```yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - "myapp.mydomain.com"
  gateways:
  - myapp-gateway
  http:
  - match:
    - authority:
        exact: "myapp.mydomain.com"
    route:
    - destination:
        port:
          number: 80
        host: myapp
```

Apply the file:
```shell
kubectl apply -f virtualservice.yaml
```

Next, we need a standard Kubernetes service:
```yml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: myapp
  name: myapp
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: myapp
  type: NodePort
```

Apply the file:

```shell
kubectl apply -f service.yaml
```

And then we need our standard Kubernetes deployment:

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  replicas: 2
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

Now before we apply the deployment, because we are using istio, we need to remember to inject the envoy sidecar, unless of course, you have added the auto-injection to your namespace. If not, remember to run the following:

```shell
kubectl apply -f <(istioctl kube-inject -f deployment.yaml)
```

Wait for a few seconds (maybe minute or two), then try to access your service using your URL (from within your VPC, preferably on the VPN). Mine was `http://myapp.mydomain.com`

If all went well then you should now be able to access your service from within your network (VPC) and no prying eyes allowed from outside your network.

Please leave comments if this was helpful, or if you have questions, or if you have a better way to implement this.

Thank you for reading.

Medium Article: [https://medium.com/swlh/public-and-private-istio-ingress-gateways-on-aws-f968783d62fe](https://medium.com/swlh/public-and-private-istio-ingress-gateways-on-aws-f968783d62fe)