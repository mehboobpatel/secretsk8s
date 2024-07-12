# Sealed Secrets Setup Guide

## Overview

Sealed Secrets is a tool provided by Bitnami that allows you to encrypt Kubernetes secrets into SealedSecrets. This ensures that secrets can be safely stored in a version control system like Git. The Sealed Secrets Controller in the Kubernetes cluster decrypts these SealedSecrets and creates the actual Kubernetes secrets.

This guide will walk you through installing `kubeseal`, creating and encrypting secrets, and then applying them to your Kubernetes cluster.

## Prerequisites

- A running Kubernetes cluster.
- `kubectl` installed and configured to communicate with your Kubernetes cluster.
- `helm` installed (optional, for installing the Sealed Secrets Controller via Helm).
- Basic knowledge of Kubernetes and Helm.

## 1. Install Sealed Secrets Controller

### Using Helm

1. **Add the Bitnami Helm repository:**

   ```sh
   helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets

   * NOTE * 
   check the namespace to install not recommended to install on kube-system
   helm install sealed-secrets -n kube-system --set-string fullnameOverride=sealed-secrets-controller sealed-secrets/sealed-secrets
    ```

2. **Install the kubeseal cli**

   ```sh
  curl -sL https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.20.0/kubeseal-0.20.0-linux-amd64.tar.gz | tar xzvf - kubeseal
sudo mv kubeseal /usr/local/bin/
kubeseal --version
```
```




3. **Create a normal secret via yaml or cli** 

```

kubectl apply -f secret.yaml

kubectl create secret generic db-user-pass --from-file=username=./username.txt --from-file=password=./password.txt

kubectl create secret generic clisecret --from-literal=name=maheboob --from-literal=age=25 --dry-run=client -o yaml   

kubectl create secret docker-registry <secretname>  --docker-server=<xyz>.azurecr.io     --docker-username=<username>     --docker-password=<password>

```
 to delete ALL SECRETS 
 
 ```
 kubectl delete secrets --all -A
 kubectl delete sealedsecrets --all -A

```

To create SEALED SECRET

```
kubectl create secret generic newsecret --dry-run=client -n kube-system --from-literal=name=maddy -o yaml > newsec.yaml | kubeseal  --controller-name sealed-secrets-controller --controller-namespace kube-system --format yaml > mysecret.yaml

```

Since you will get this error if you try to follow documentaion format:
![alt text](image-6.png)

To solve this follow below steps
```
kubectl get secrets -A
```

Now get the certifcate(public key of sealed secret controller)
First see the secrets using
```
kubectl get secrets -A
```
![alt text](image-1.png)

now use this name under 

```
NOTE THE -n tag use your namespace where kubeseal controller is running
kubectl get secret -n kube-system <sealed-secrete-name> -o jsonpath="{.data.tls\.crt}" | base64 --decode > pub-cert.pem
```
![alt text](image-2.png)

4. **FINAL SEALING SECRET** 

Now 
```
kubeseal --cert pub-cert.pem --format yaml < secret.yaml > sealedsecret.yaml #THIS WILL GIVE YAML

OR 

cat secret.yaml | kubeseal --cert pub-cert.pem > catsealedsecret.yaml  #THIS WILL GIVE JSON



#YOU CAN APPLY ANY OF THE JSON OR YAML THE END RESULT IS SAME
kubectl apply -f sealedsecret.yaml

```
![alt text](image-3.png)

NOTE:

if you create secrets from a sealedsecret.yaml 
its listed under kubectl get secrets as well as 
kubectl get sealedsecrets
UNTILL and unless you delete the sealedsecrete the normal secret will b recreated
![alt text](image-4.png)

POST DELETING SEALED SECRET
![alt text](image-5.png)


5. **NOW YOU CAN PUSH TO SCM LIKE GITHUB AND GITLAB WITHOUT ANY WORRIES SECURITY ACHIEVED** 

When you pull the code into a new cluster or environment
you just need to apply the sealedsecret.yaml and K8s will decrypt the secrets in the cluster
* NOTE :
kubectl logs -n kube-system -l app=sealed-secrets-controller
kubectl get svc -n kube-system



## EXTERNAL SECRET OPERATOR  (encrypted in cloud , no encryption via k8s , yes encoding by k8s)


1. **Architecture & Installation** 

```
helm repo add external-secrets https://external-secrets.github.io/kubernetes-external-secrets/
helm install external-secrets external-secrets/kubernetes-external-secrets
```
![alt text](image-7.png)

External Secret Operator: A Kubernetes operator that fetches secrets from an external secrets management system.
Secret Store: An external system where secrets are stored, such as AWS Secrets Manager, HashiCorp Vault, or Azure Key Vault.
ExternalSecret Custom Resource: A Kubernetes custom resource that defines which secrets to fetch from the secret store and how to map them to Kubernetes secrets.
Kubernetes Secret: A standard Kubernetes secret that gets created and populated with the fetched secrets.


2. **Create ServiceAccount on Cloud** 
![alt text](image-8.png)
![alt text](image-10.png)
![alt text](image-11.png)
![alt text](/Screenshots/image-12.png)

* NOTE use only json as p12 is not supported

Download this json and copy paste the content as it is into secretaccess

now create 
```
kubectl apply -f secretaccess.yaml
```
now refer the below link for your provider you have to create the secretstore.yaml (configurationfile) accordingly
here we will see of GCP
https://external-secrets.io/latest/provider/google-secrets-manager/

now create 
```
kubectl apply -f secretstore.yaml

````

3. **Create Secret in Secret Manager on Cloud** 

now go to secret manager in google cloud portal and give the secret accessor permission from IAM to this service account.
and create a secret key in secretmanager(vault)

![alt text](image-9.png)

now create
```
contains:
the reference of secret key created on google cloud portal 
the SA account creds refrence to secretstore.
the Final Secret which will be created in k8s cluster from the key:value stored in GCP secret Manager
kubectl apply -f externalsecret.yaml 
```

to check 
```
kubectl get secrets -n external-secrets
kubectl get externalsecrets -n external-secrets
```



