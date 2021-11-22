# Deploy Functions On Aks 

- Simple basic setup before looking at advance scenarios. 
- You would be able to create a http trigger function and deploy it on AKS
- Would allow you to control/inspect network.
- Same steps would allow to deploy on any other Kubernetes environment
- Azure arc not mandatory. However it would be recommended as it provides a better control plane. Set up Azure Arc for App Service, Functions, and Logic Apps - Azure App Service | Microsoft Docs
- Function can be exposed internally or have a public IP.
- Please comment for any doubts or blockers.

## What we are aiming to do.
Expose a HTTP trigger azure function internally. We would be able to access it from a bastion VM on the same network but not externally.
Alternatively, see how a simple YAML change will allow access to function via public IP.



## Steps -
1. Create a test function with http endpoint.  You can do it in visual studio code or Visual studio IDE. No change needed in code to support deployment on AKS. see below local.settings.json and host.json file that I used.

`{
    "IsEncrypted": false,
    "Values": {
        "AzureWebJobsStorage": "UseDevelopmentStorage=true",
        "FUNCTIONS_WORKER_RUNTIME": "dotnet"
    }
}`

`{
    "version": "2.0",
    "logging": {
        "applicationInsights": {
            "samplingSettings": {
                "isEnabled": true,
                "excludedTypes": "Request"
            }
        }
    }
}`

2. Create docker image of function. Add a dockerfile to the project and create docker file. See below the dockerfile I used. Its the default one visual studio created for me when I added docker support to project. No alterations were done to it to support functions on K8.

FROM mcr.microsoft.com/azure-functions/dotnet:3.0 AS base
WORKDIR /home/site/wwwroot
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:3.1 AS build
WORKDIR /src
COPY ["FunctionAppHttpTrigger1.csproj", "."]
RUN dotnet restore "./FunctionAppHttpTrigger1.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "FunctionAppHttpTrigger1.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "FunctionAppHttpTrigger1.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /home/site/wwwroot
COPY --from=publish /app/publish .
ENV AzureWebJobsScriptRoot=/home/site/wwwroot \
    AzureFunctionsJobHost__Logging__Console__IsEnabled=true

3. Create azure container registry. Push the docker image here. You can use any registry to hold docker images but since we are going to deploy on azure AKS, we are using ACR.

4. Create an AKS cluster. We are going to deploy our function on this AKS. When creating AKS, we are using CNI network. That will give our function endpoint a private IP from VNET itself. To create a CNI based AKS, follow guidelines here. https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni

5. Install keda on this AKS cluster. First get credentials of the AKS cluster using steps here - https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough#connect-to-the-cluster 
then install KEDA on this AKS cluster. KEDA is optional for this exercise but mandatory as this is resposnsible for scaling of functions in AKS using metrics available. You would need function CLI to install KEDA. https://github.com/Azure/azure-functions-core-tools
Once you have functions CLI installed, use below command after connecting to AKS to install.
`func kubernetes install --namespace keda
`

6. Next step is to create a YAML for deploying function to AKS. We dont need to create one from scratch. We can use the --dry-run option to create one. Use the function cli and run command below to create YAML file. 
`func kubernetes deploy --name review-functions --image-name "xxx-image patch from ACR-xxx" --dry-run > ./function-deploy.yaml
`
7. In the autogenarated YAML, we have service defined as LoadBalancer. Inorder to use this function endpoint via private IP only, we need to change Load balancer to an interna one. We can do that by just adding one line to YAML file. If we dont add this file, function will be available via public IP also. Add this in service metadata. See YAML file attached in github repo.
`annotations:
  service.beta.kubernetes.io/azure-load-balancer-internal: "true"
`

8. Now apply the YAML file. This will deploy function and create needed deployment and  service. 
`kubectl apply -f ./manifests/review-function-deploy.yaml
`

9. Once you deploy the file, you would be able to run the below command to see deployment logs and also the API endpoint that can be used for tesitng. 
`kubectl logs -l app=review-functions-http --follow --tail 1000`

10. to get the IP of the http endpoint of function, run the below command. Note the external-IP. It will be under column External-IP and will be a private IP from Vnet used.
`kubectl get services`

11. Now run the following on your browser. You will not be able to call function endpoint since its deployed using private IP. 
http://External-IP/api/SampleHttpApplication1?name=testing123

12. Now on the same VNET where we have deployed AKS, create another subnet and install a private VM. Login to this VM using bastion service and call above function endpoint again using curl. You will see it will work since VM on same VNET as AKS can reach the private IP.

