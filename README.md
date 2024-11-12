# 🌟Service mesh observability setup using Istio, Kiali, and Jaeger🌟:

![arch](https://github.com/user-attachments/assets/11989562-79fb-48e6-9820-dca82b3c8c99)


✅ServiceMesh-Istio-Kiali-Jaeger is a complete service mesh observability solution, leveraging Istio for traffic management, Kiali for real-time service visualization, and Jaeger for distributed tracing. 


✅This repository provides configuration, deployment guides, and best practices for setting up a robust monitoring and tracing environment within a Kubernetes-based service mesh. Ideal for gaining deep insights into microservices traffic flow, performance bottlenecks, and service health in cloud-native applications.

# Istio Service Mesh

This guide will walk you through setting up a Service Mesh using Istio. The app consists of five microservices:

1. **Vote Microservice**: Collects votes and sends them to Redis for temporary processing.
2. **Redis**: Holds votes temporarily before transferring them to the Worker service.
3. **Worker Microservice**: Processes each vote and updates the PostgreSQL database.
4. **PostgreSQL**: Securely stores votes for result aggregation.
5. **Result Microservice**: Fetches and displays live voting results from PostgreSQL.

## Why Use a Service Mesh?

Before Service Mesh, each microservice communicated directly with others using REST APIs or gRPC over HTTP. However, this approach required each service to handle its own security, retries, and monitoring, which made the system complex and hard to manage. Developers had to write a lot of custom code to handle these tasks.

**Enter Istio**: A service mesh that automatically manages communication between services. Istio handles:

- **Security**: Ensures services communicate securely.
- **Retries**: Automatically retries failed requests.
- **Traffic Control**: Monitors and manages traffic between services.

With Istio, developers can focus on building features rather than managing communication.

## Installation Guide

### 1. Install Istio

Install Istio by running the following commands:

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.18.0 sh -
cp istio-1.18.0/bin/istioctl /usr/local/bin/
istioctl --version
```



![1-istio](https://github.com/user-attachments/assets/36818fd3-9e59-4823-a781-7697093ff276)



### 2. Deploy Istio Ingress Gateway

Create and deploy the Istio Ingress Gateway:


```bash
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-ingressgateway
spec:
  profile: empty # Do not install CRDs or the control plane
  components:
    ingressGateways:
    - name: ingressgateway
      namespace: istio-system
      enabled: true
      k8s:
        serviceAnnotations:
          service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
      label:
        istio: ingressgateway
  values:
    gateways:
      istio-ingressgateway:
        injectionTemplate: gateway
```



```bash
nano /tmp/istio-ingressgateway.yml
istioctl install --set profile=minimal -f istio-ingressgateway.yml
```



![2-istio-ingress-gateway](https://github.com/user-attachments/assets/5f58abb1-3fbc-4b5d-9561-f383ef25d8d4)





### 3. Install Prometheus for Monitoring

Deploy Prometheus in the Istio namespace:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.16/samples/addons/prometheus.yaml
```


![3-prometheus](https://github.com/user-attachments/assets/f7bfe595-329a-436f-95b0-392facae956d)





### 4. Enable Istio on the Default Namespace

Enable Istio for the default namespace where your microservices are deployed:

```bash
kubectl label namespace default istio-injection=enabled
```


![4-istio-enebled-default-ns](https://github.com/user-attachments/assets/46633d5d-d06e-4672-95a6-7fcb55908fcf)






### 5. Install Kiali and Jaeger for Visualization and Tracing

Deploy Kiali (for dashboard) and Jaeger (for tracing) in the Istio namespace:

```bash
# Kiali
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.16/samples/addons/kiali.yaml
```


![5-kiali-install](https://github.com/user-attachments/assets/8a30648b-cec6-4971-8215-4579d3ab90e7)




```bash
# Jaeger
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.16/samples/addons/jaeger.yaml
```


![6-jaeger-install](https://github.com/user-attachments/assets/3161db2e-3545-4fce-8c19-a33f9c4700ce)






### 6. Deploy the Voting Application:


![image](https://github.com/user-attachments/assets/51bb1e4a-d9e4-4941-97bb-b0eb62d37d48)






Deploy your voting application in the default namespace, ensuring that you create the necessary secrets for private Docker images.


```bash
# redis
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: redis
  name: redis
spec:
  clusterIP: None
  ports:
    - name: redis-service
      port: 6379
      targetPort: 6379
  selector:
    app: redis
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:alpine
          ports:
            - containerPort: 6379
              name: redis

# db
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: db
  name: db
spec:
  clusterIP: None
  ports:
    - name: db
      port: 5432
      targetPort: 5432
  selector:
    app: db
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  labels:
    app: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: db
          image: postgres:9.4
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              value: postgres
          ports:
            - containerPort: 5432
              name: db
          volumeMounts:
            - name: db-data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: db-data
          emptyDir: {}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

# result
---
apiVersion: v1
kind: Service
metadata:
  name: result
  labels:
    app: result
spec:
  #type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
      name: result-service
  selector:
    app: result
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result
  labels:
    app: result
spec:
  replicas: 1
  selector:
    matchLabels:
      app: result
  template:
    metadata:
      labels:
        app: result
    spec:
      containers:
        - name: result
          image: rajpractise/votingapp:results
          ports:
            - containerPort: 80
              name: result
      imagePullSecrets:
        - name: docker-pwd
# vote
---
apiVersion: v1
kind: Service
metadata:
  name: vote
  labels:
    apps: vote
spec:
  #type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
      name: vote-service
  selector:
    app: vote
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote
  labels:
    app: vote
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vote
  template:
    metadata:
      labels:
        app: vote
    spec:
      containers:
        - name: vote
          image: rajpractise/votingapp:vote
          ports:
            - containerPort: 80
              name: vote
      imagePullSecrets:
        - name: docker-pwd

# worker
---
apiVersion: v1
kind: Service
metadata:
  labels:
    apps: worker
  name: worker
spec:
  clusterIP: None
  selector:
    app: worker
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: worker
  name: worker
spec:
  replicas: 1
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
        - image: rajpractise/votingapp:worker
          name: worker
      imagePullSecrets:
        - name: docker-pwd
```



![7-voteapp-deployment](https://github.com/user-attachments/assets/ab21aec4-c15d-42d8-a408-fc5456459139)





1. Create TLS keys:

    ```bash
    nano tls.key
    nano tls.crt
    ```


 ![8-tls-secrets](https://github.com/user-attachments/assets/d1f30a8f-e071-4dd2-aa24-4bbd11199590)
   




2. Create secrets in both namespaces:

    ```bash
    kubectl create secret tls my-tls-secret --key="tls.key" --cert="tls.crt" -n istio-system
    kubectl create secret tls my-tls-secret --key="tls.key" --cert="tls.crt" -n default
    ```


 ![9-tls-secrets-created](https://github.com/user-attachments/assets/fa931954-55f9-4535-ace2-27ee80fa97a3)
   





### 7. Update Route 53 Records and Deploy Gateway

Before deploying, update your DNS records in Route 53. Then deploy the gateway:



🔹     ![10-dnsrecord](https://github.com/user-attachments/assets/f83984f1-f37e-436b-abee-52d14aae2a68)





🔹    ![10-dnsrecord1](https://github.com/user-attachments/assets/f97cdbe5-05fb-4071-bb1c-b59c69533a83)





🔹    ![10-dnsrecord2](https://github.com/user-attachments/assets/434fb65a-b385-426f-bdfb-3c7bfcda4338)






🔹      ![10-dnsrecord3](https://github.com/user-attachments/assets/720cab1f-e10b-46c0-8a8d-36f00064bb6d)






🔹      ![10-dnsrecord4](https://github.com/user-attachments/assets/61f25a5a-06d0-43e2-aabe-02d38fa8a1b6)





###########################################################################################################



# Deploy the gateway:

nano votingapp-gw-vs-ssl.yaml

```bash
apiVersion: networking.istio.io/v1beta1
kind: Gateway #Similar to Kubernetes Ingress
metadata:
  name: app-gateway
spec:
  selector:
    istio: ingressgateway # label of ingressgateway deployed above
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "vote.cloudopswithswapnil.in"
    - "result.cloudopswithswapnil.in"
    - "www.cloudopswithswapnil.in"
    tls:
     httpsRedirect: true
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "vote.cloudopswithswapnil.in"
    - "result.cloudopswithswapnil.in"
    - "www.cloudopswithswapnil.in"
    tls:
      credentialName: my-tls-secret # this should match with Certificate secretName
      mode: SIMPLE
      privateKey: sds
      serverCertificate: sds

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: vote
spec:
  hosts:
  - "vote.cloudopswithswapnil.in"
  - "www.cloudopswithswapnil.in"
  gateways:
  - app-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: vote
        port:
          number: 80

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: result
spec:
  hosts:
  - "result.cloudopswithswapnil.in"
  gateways:
  - app-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: result
        port:
          number: 80
```


```bash
kubectl apply -f votingapp-gw-vs-ssl.yaml -n default
```





![11-deploy-gw-vs-ssl](https://github.com/user-attachments/assets/a447efdf-2acb-4b5f-b146-19753b86bfe1)








# Deploy the tools gateway and virtaul services for kiali and jaeger:


```bash
nano kiali-jaeger.yml

apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: tools-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "tools.cloudopswithswapnil.in"
    tls:
     httpsRedirect: true
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "tools.cloudopswithswapnil.in"
    tls:
      credentialName: my-tls-secret # this should match with Certificate secretName
      mode: SIMPLE
      privateKey: sds
      serverCertificate: sds
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: kiali
  namespace: istio-system
spec:
  hosts:
  - "tools.cloudopswithswapnil.in"
  gateways:
  - tools-gateway
  http:
  - match:
    - uri:
        prefix: /kiali
    route:
    - destination:
        host: kiali
        port:
          number: 20001

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: tracing
  namespace: istio-system
spec:
  hosts:
  - "tools.cloudopswithswapnil.in"
  gateways:
  - tools-gateway
  http:
  - match:
    - uri:
        prefix: /jaeger
    route:
    - destination:
        host: tracing
        port:
          number: 80
```




![12-kiali-jaeger-deploy](https://github.com/user-attachments/assets/fc8e8cea-28e6-4c04-b62f-7401f920cda0)









# Verify the APP deployment:



🔹       ![13-app-deploy](https://github.com/user-attachments/assets/7447919e-6239-44f3-b980-8b16cf9181d5)






🔹       ![13-app-deploy1](https://github.com/user-attachments/assets/ebe3fafc-babf-4830-9693-6cb5a2588e82)










### 8. Monitor Your Services

Use Kiali and Jaeger in the Istio namespace to monitor and trace your services.


# Istio:




🔹      ![14-monitor-kiali](https://github.com/user-attachments/assets/50a2a2b8-cd64-4240-bad5-3e332f7902f9)










🔹      ![14-monitor-kiali1](https://github.com/user-attachments/assets/46224149-219c-4b58-b011-05c7c6914b27)










🔹       ![14-monitor-kiali2](https://github.com/user-attachments/assets/f83a8e3b-63de-4a53-bdd0-08719c810b72)









# Jaeger:






🔹           ![15-monitor-jaeger](https://github.com/user-attachments/assets/922cb79f-abfc-4b2f-aa1e-02db69191700)











🔹           ![15-monitor-jaeger1](https://github.com/user-attachments/assets/7b037991-83fd-49f5-9e63-2ee1931bd85e)


















# Conclusion:
ServiceMesh-Istio-Kiali-Jaeger delivers a powerful observability solution for Kubernetes service meshes, combining Istio for traffic management, Kiali for visualization, and Jaeger for tracing. This setup enhances microservices monitoring and performance, with ongoing improvements encouraged.







