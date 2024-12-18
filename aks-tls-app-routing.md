# Configure TLS for using Application Routing (Managed NGINX)

> Estimated Duration: 60 minutes
>> **NOTE:** You need to fulfill these [requirements](environment-setup.md) and [AKS Basic Cluster](aks-basic-cluster.md) to complete this exercise.

An Ingress is an API object that defines rules, which allow external access to services in an Azure Kubernetes Service (AKS) cluster. When you create an Ingress object that uses the application routing add-on nginx Ingress classes, the add-on creates, configures, and manages one or more Ingress controllers in your AKS cluster.

This lab shows you how to set up an advanced Ingress configuration to encrypt the traffic with SSL/TLS certificates stored in an Azure Key Vault, and use Azure DNS to manage DNS zones.

## Reference

* [Managed NGINX ingress with the application routing add-on](https://learn.microsoft.com/en-us/azure/aks/app-routing)
* [Set up a custom domain name and SSL certificate with the application routing add-on](https://learn.microsoft.com/en-us/azure/aks/app-routing-dns-ssl)

## Setup

In a terminal, export variables required for this lab (if not already exported):

```bash
INITIALS=abc
CLUSTER_NAME=aks-$INITIALS
RG=aks-$INITIALS-rg
LOCATION=eastus2
```

If not already connected, connect to the cluster from your local client machine.

```bash
az aks get-credentials --name $CLUSTER_NAME -g $RG
```

## Enable Application Routing add-on

Run the following command to enable the `approuting` add-on if not already enabled

```bash
az aks approuting enable --resource-group $RG --name $CLUSTER_NAME
```

## Create Azure Key Vault

Create a new Azure Key Vault with Azure role-based access control (Azure RBAC) enabled:

```bash
AKV_NAME=$INITIALS-kv
az keyvault create --name $AKV_NAME --resource-group $RG --location $LOCATION --enable-rbac-authorization
```

Retrieve the Azure Key Vault resource id for later use:

```bash
KEYVAULT_ID=$(az keyvault show --name $AKV_NAME --resource-group $RG \
--query id --output tsv | tr -d '\r')
```

## Enable Azure Key Vault integration

Update the app routing add-on to enable the Azure Key Vault secret store CSI driver and apply the role assignment.

```bash
az aks approuting update --resource-group $RG --name $CLUSTER_NAME --enable-kv --attach-kv ${KEYVAULT_ID}
```

## Create a domain registration

Choose a domain name expected to be unique:

```bash
DOMAIN_NAME=$INITIALS-azuredemo.com
```

### Option 1: Create Domain registration (optional)

This step is optional, but without this validation will be only via IP. If you choose to register a domain, you can confirm it is not registered by adding the `--dryrun` flag to the command below:

```bash
az appservice domain create -g $RG --hostname $DOMAIN_NAME \
--contact-info @manifests/contact-info.json --accept-terms 
```

### Option 2: Create a public Azure DNS zone

Create an Azure DNS zone using the az network dns zone create command.

```bash
az network dns zone create --resource-group $RG --name $DOMAIN_NAME
```

## Attach Azure DNS zone to the application routing add-on

Retrieve the resource ID for the DNS zone using this command:

```bash
ZONE_ID=$(az network dns zone show -g $RG --name $DOMAIN_NAME --query "id" --output tsv)
```

Update the add-on to enable the integration with Azure DNS using this command:

```bash
az aks approuting zone add -g $RG --name $CLUSTER_NAME --ids=${ZONE_ID} --attach-zones
```

## Generate a TLS certificate

Export certificate name variable:

```bash
CERT_NAME=aks-ingress-cert
```

### Option 1: Using trusted CA certificate

This option requires using a registered domain and then issuing a Trusted CA certificate

#### Generate trusted CA certificate

This certificate is issued by Let's Encrypt CA and requires installing `cerbot` see [environment setup](environment-setup.md)

Run the following command:

```bash
sudo certbot certonly --agree-tos --register-unsafely-without-email --manual \
--preferred-challenges dns -d $DOMAIN_NAME -d *.$DOMAIN_NAME
```

On a separate terminal run the command below. You will need to export variables and copy challenge value:

```bash
CHALLENGE=<challenge_value>
az network dns record-set txt add-record --resource-group $RG --zone-name $DOMAIN_NAME --record-set-name _acme-challenge --value $CHALLENGE
```

Before confirming challenge, confirm TXT DNS record has been propagated using this command:

```bash
nslookup -q=txt _acme-challenge.$DOMAIN_NAME
```

Once the record is successfully validated, use the command below to export certificate into pfx format (provide sudo password if prompted, but skip export password when prompted):

```bash
sudo openssl pkcs12 -export -out $CERT_NAME.pfx -inkey /etc/letsencrypt/live/$DOMAIN_NAME/privkey.pem \
-in /etc/letsencrypt/live/$DOMAIN_NAME/fullchain.pem
```

Change ownership of exported certificate file:

```bash
sudo chown $(whoami) $CERT_NAME.pfx
```

### Option 2: Using self-signed certificate

> In this option you will only be able to validate using `curl` command to resolve domain.
Generate a TLS certificate using the following command:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -out aks-ingress-tls.crt \
    -keyout aks-ingress-tls.key \
    -subj "/CN=${DOMAIN_NAME}/O=aks-ingress-tls"
```

Export the certificate to a PFX file using the following command (skip password prompt):

```bash
openssl pkcs12 -export -in aks-ingress-tls.crt -inkey aks-ingress-tls.key -out $CERT_NAME.pfx
```

## Import the certificate into Azure Key Vault

Get the user principal name and the key vault ID:

```bash
PRINCIPAL_NAME=$(az ad signed-in-user show --query userPrincipalName --output tsv | tr -d '\r')
```

Grant Key Vault Contributor role to be able to import certificate:

```bash
az role assignment create --assignee $PRINCIPAL_NAME --role "Key Vault Certificates Officer" --scope $KEYVAULT_ID
```

Import the certificate using this command:

```bash
az keyvault certificate import --vault-name $AKV_NAME --name $CERT_NAME --file $CERT_NAME.pfx
```

## Deploy test application to AKS

Create a namespace:

```bash
NAMESPACE=tls-managed-nginx
kubectl create namespace $NAMESPACE
```

Deploy using this command:

```bash
kubectl apply -f manifests/aks-helloworld.yaml -n $NAMESPACE
```

Confirm deployment is successful and pod is in `RUNNING` state:

```bash
kubectl get pods -n $NAMESPACE
```

## Create Ingress that uses host name and certificate from Azure Key Vault

Get the certificate URI to use in the Ingress from Azure Key Vault using this command:

```bash
KV_CERT_URI=$(az keyvault certificate show --vault-name $AKV_NAME --name $CERT_NAME --query "id" --output tsv | tr -d '\r')
```

Before running next command verify required variables are exported:

```bash
echo CERT_URI: $KV_CERT_URI, DOMAIN_NAME: $DOMAIN_NAME, NAMESPACE: $NAMESPACE
```

Create the App Routing Ingress using this command:

```bash
cat <<EOF | kubectl apply -n $NAMESPACE -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.azure.com/tls-cert-keyvault-uri: $KV_CERT_URI
  name: aks-helloworld
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
  - host: $DOMAIN_NAME
    http:
      paths:
      - backend:
          service:
            name: aks-helloworld
            port:
              number: 80
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - $DOMAIN_NAME
    secretName: keyvault-aks-helloworld
EOF
```

## Validate test application

Confirm secret was created, pods are running (you should see a new one for `keyvault` injection) and ingress has IP `ADDRESS` assigned:

```bash
kubectl get pods,secret,ingress -n $NAMESPACE
```

Get the external IP for the `nginx-ingress` service:

```bash
EXTERNAL_IP=$(kubectl get ingress -n $NAMESPACE -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
```

Verify your ingress is properly configured with TLS using the following command:

```bash
curl -v -k --resolve $DOMAIN_NAME:443:$EXTERNAL_IP https://$DOMAIN_NAME
```

You should see the server certificate (either self-signed or CA issued) in the output

### Trusted CA with domain registration validation

If a domain registration and Trusted Certificate was used, then you can additionally validate using the qualified domain name. Create a DNS record to point to the ingress IP:

```bash
az network dns record-set a add-record --ipv4-address $EXTERNAL_IP --record-set-name "@" -g $RG --zone-name $DOMAIN_NAME
```

Test reaching the fully qualified domain name:

```bash
curl -v https://$DOMAIN_NAME
```

## Cleanup

If you are done with all labs and have no plan to continue further, then clean up by deleting the resource group:

```bash
az group delete --name $RG --yes --no-wait
```

Otherwise remove the following resources, keeping the AKS cluster:

```bash
kubectl delete namespace $NAMESPACE
```
