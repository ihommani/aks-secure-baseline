# Workflow Prerequisites

The AKS Cluster has been enrolled in [GitOps management](./06-gitops), wrapping up the infrastructure focus of the [AKS secure Baseline reference implementation](./). Follow the steps below to create the TLS certificate that the Ingress Controller will serve for Application Gateway to connect to your web app.

## Steps

## Generate the wildcard certificate for the Ingress Controller using Azure KeyVault

> :book: Contoso Bicycle needs to procure a CA certificate for the web site. As this is going to be a user-facing site, they purchase an EV cert from their CA.  This will serve in front of the Azure Application Gateway.  They will also procure another one, a standard cert, to be used with the AKS Ingress Controller. This one is not EV, as it will not be user facing.

1. Obtain the Azure Key Vault details and give the current user permissions to create certificates.

   > :book: Finally the app team decides to use a wildcard certificate of `*.aks-ingress.contoso.com` for the ingress controller. They use Azure Key Vault to create and manage the lifecycle of this certificate.

   ```bash
   KEYVAULT_NAME=$(az deployment group show --resource-group rg-bu0001a0008 -n cluster-stamp --query properties.outputs.keyVaultName.value -o tsv)
   az keyvault set-policy --certificate-permissions create list get -n $KEYVAULT_NAME --upn $(az account show --query user.name -o tsv)
   ```

1. Generate the Cluster Ingress Controller's Wildcard Certificate for `*.aks-ingress.contoso.com`.

    :warning: If you already have access to an appropriate certificate, or can procure one from your organization, consider skipping this step and instead [import it into Key Vault](https://docs.microsoft.com/azure/key-vault/certificates/tutorial-import-certificate#import-a-certificate-to-key-vault).

    :warning: Do not use the certificate created by this script for actual deployments. The use of self-signed certificates are provided for ease of illustration purposes only. For your cluster, use your organization's requirements for procurement and lifetime management of TLS certificates, _even for development purposes_.

   ```bash
   cat <<EOF | az keyvault certificate create --vault-name $KEYVAULT_NAME -n traefik-ingress-internal-aks-ingress-contoso-com-tls -p @-
   {
     "issuerParameters": {
       "certificateTransparency": null,
       "name": "Self"
     },
     "keyProperties": {
       "curve": null,
       "exportable": true,
       "keySize": 2048,
       "keyType": "RSA",
       "reuseKey": true
     },
     "lifetimeActions": [
       {
         "action": {
           "actionType": "AutoRenew"
         },
         "trigger": {
           "daysBeforeExpiry": 90
         }
       }
     ],
     "secretProperties": {
       "contentType": "application/x-pkcs12"
     },
     "x509CertificateProperties": {
       "keyUsage": [
         "cRLSign",
         "dataEncipherment",
         "digitalSignature",
         "keyEncipherment",
         "keyAgreement",
         "keyCertSign"
       ],
       "subject": "O=Contoso Aks Ingress, CN=*.aks-ingress.contoso.com",
       "validityInMonths": 12
     }
   }
   EOF
   ```

## Configure Azure Application Gateway and Azure Key Vault

Since we are extending TLS from the Azure Application Gateway through to the Ingress Controller, application gateway needs to know about the root certificate, which it can find from Key Vault.

1. Query the Azure Application Gateway name.

   ```bash
   export APP_GATEWAY_NAME=$(az deployment group show -g rg-bu0001a0008 -n cluster-stamp --query properties.outputs.agwName.value -o tsv)
   ```

1. Configure the trusted root certificate.

   ```bash
   az network application-gateway root-cert create -g rg-bu0001a0008 --gateway-name $APP_GATEWAY_NAME --name  root-cert-wildcard-aks-ingress-contoso --keyvault-secret $(az keyvault certificate show --vault-name  $KEYVAULT_NAME -n traefik-ingress-internal-aks-ingress-contoso-com-tls --query sid -o tsv)
   ```

1. Configure the HTTP Settings to use the root certificate.

   ```bash
   az network application-gateway http-settings update -g rg-bu0001a0008 --gateway-name $APP_GATEWAY_NAME -n aks-ingress-contoso-backendpool-httpsettings --root-certs root-cert-wildcard-aks-ingress-contoso --protocol Https
   ```

### Next step

:arrow_forward: [Configure AKS Ingress Controller with Azure Key Vault integration](./08-secret-managment-and-ingress-controller.md)