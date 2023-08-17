**EKS-Multi-cluster-Istio**

**Prerequisites**

* Helm v3
* kubectl
* aws cli
* eksdemo
* Amazon EKS v1.27


Setup
1. Create 2 EKS Clusters in the Same Region

eksdemo create cluster Cluster1 -N 3 -i t3.large --region us-west-2 --version 1.27
eksdemo create cluster Cluster2 -N 3 -i t3.large --region us-west-2 --version 1.27 --vpc-cidr 192.168.0.0/16


Note: Replace instance type as per your convenience.

2. Install cert-manager Addon

eksdemo install cert-manager --cluster Cluster1
eksdemo install cert-manager --cluster Cluster2

3. Configure Root CA on Each Cluster

Cluster1:

kubectl apply -f cert-manager/self-signed-ca.yaml 
kubectl apply -f cert-manager/istio-cert.yaml

Cluster2:

kubectl apply -f cert-manager/self-signed-ca.yaml 
kubectl apply -f cert-manager/istio-cert.yaml


4. Install Istio on Both Clusters

Cluster1:

istioctl install -f istio-cluster1.yaml
kubectl apply -f istio-auth.yaml
kubectl apply -f istio-east-west-gateway.yaml

Cluster2:

istioctl install -f istio-cluster2.yaml
kubectl apply -f istio-auth.yaml
kubectl apply -f istio-east-west-gateway.yaml


5. Establish Connectivity Between API Endpoints

Generate on Cluster1:

istioctl x create-remote-secret --name=cluster1 --server=<API Server Endpoint of Cluster1> > for-cluster2-secret.yaml

Apply this on cluster2
kubectl apply -f for-cluster2-secret.yaml

Note: Get the API Server endpoint from EKS console under overview for Cluster1.


Cluster2:

Generate this on Cluster 2

istioctl x create-remote-secret --name=cluster2 --server=<API Server Endpoint for cluster2> > for-cluster1-secret.yaml

Apply the YAML on Cluster1

kubectl apply -f for-cluster1-secret.yaml



6. Deploy Hello World App

kubectl create ns sleep
kubectl create ns demo
kubectl label ns sleep istio-injection=enabled
kubectl label ns demo istio-injection=enabled


On Cluster1:

kubectl apply -f helloworld-v1.yaml -n demo
kubectl apply -f sleep.yaml -n sleep

On Cluster2:

kubectl apply -f helloworld-v2.yaml -n demo
kubectl apply -f sleep.yaml -n sleep

7. Validation

Switch to Cluster1:

kubectl exec -n sleep -c sleep \
    "$(kubectl get pod -n sleep -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- sh -c "while true; do curl -sS helloworld.demo:5000/hello; done"

output:
Hello version: v1, instance: helloworld-v1-b6c45f55-mdsnx
Hello version: v1, instance: helloworld-v1-b6c45f55-mdsnx
Hello version: v1, instance: helloworld-v1-b6c45f55-mdsnx
Hello version: v1, instance: helloworld-v1-b6c45f55-mdsnx
Hello version: v1, instance: helloworld-v1-b6c45f55-mdsnx


8. Create a Gateway and a Virtual Service for hello world on Cluster2

kubectl apply -f gateway-virtualservice-cluster2.yaml -n demo


export INGRESS_NAME=istio-ingressgateway
export INGRESS_NS=istio-system

GATEWAY_URL=$(kubectl -n "$INGRESS_NS" get service "$INGRESS_NAME" -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo $GATEWAY_URL


Check the service by calling virtual service from the Cluster1


Switch to Cluster1

kubectl exec -n sleep -c sleep \
    "$(kubectl get pod -n sleep -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- sh -c "while true; do curl -s http://$GATEWAY_URL/hello; done"


Output:
Hello version: v2, instance: helloworld-v2-79d5467d55-ptwkp
Hello version: v2, instance: helloworld-v2-79d5467d55-ptwkp
Hello version: v2, instance: helloworld-v2-79d5467d55-ptwkp


This concludes end of the demo and ensures Istio Multi-cluster is working on EKS in us-west-2 region.




