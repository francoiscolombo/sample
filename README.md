# sample

![CI](https://github.com/francoiscolombo/sample/workflows/CI/badge.svg)

## the application

this is a sample python flask application, that we can run with docker-compose.

this application is using SQLAlchemy which allows to use any DB backend with an abstraction layer (Object Relational Mapper).

so for development process we can use SQLlite, and later use mySQL (with the docker-compose stack)

we are also using flask-migrate which bring us the `flask db migrate` command, allowing to upgrade or downgrade easily the database by the usage of automated migration scripts.

we are also using flask-wtf for managing the forms, and flask-login for the user-login state, so we can navigate through different pages and the application "remembers" that we are logged in. it also provides the "remember me" functionality that allows to stay logged in even after closing the browser.

finally, a very simple tests file is present and can be run like this:

    (venv) fco@venus:~/workspaces/sample/fcblog$ python tests.py 
    [2020-03-07 21:52:33,029] INFO in __init__: FC Blog startup
    test_avatar (__main__.UserModelCase) ... ok
    test_follow (__main__.UserModelCase) ... ok
    test_follow_posts (__main__.UserModelCase) ... ok
    test_password_hashing (__main__.UserModelCase) ... ok

    ----------------------------------------------------------------------
    Ran 4 tests in 0.403s

    OK

this will be used by the GitHub CI workflow to execute tests (well it's just to have a 'test' step actually)

## docker compose stack

simply run the following command to run the docker-compose stack:

    $ docker-compose up -d

this will build the application as a container, and starts two other containers: a mySQL DB container (for the backend), and a phpMySQLAdmin which allows to check the database.

you can check that the application is working with the following url:
- phpmyadmin: http://localhost:8080/
- FC Blog (this application): http://localhost:5000/

## AKS deployment

!!!PLEASE NOTE THAT THIS PART IS CURRENTLY DEACTIVATED!!!

you need an Azure subscription, of course. we are going to use the Azure Cloud Shell, and the Azure CLI, in order to create all what we need.

### Create the Resource Group

the first step is to create a resource group. it's the root of all the objects that we are going to create, and also it allows to create resources in a particular location.

open Azure Cloud Shell and enter this command:

    $ az account list-locations -o table
    $ az group create --name CfAksRg --location switzerlandnorth --tags "environment=dev"

the first command allows to get the list of locations associated with your azure account. since we are in Switzerland, I choosed a Switzerland location.

we should get an answer like this one:

```json
{
    "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/CfAksRg",
    "location": "switzerlandnorth",
    "managedBy": null,
    "name": "CfAksRg",
    "properties": {
        "provisioningState": "Succeeded"
    },
    "tags": {
        "environment": "dev"
    },
    "type": "Microsoft.Resources/resourceGroups"
}
```

### Create the Azure Container Registry

this step is straight forward, simply use this command:

    $ az acr create --resource-group CfAksRg --name CfAksContainerRegistry --sku Basic --tags "environment=dev"

once it's finished, you get this answer:

```json
{
  "adminUserEnabled": false,
  "creationDate": "2020-03-08T09:57:02.777325+00:00",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/CfAksRg/providers/Microsoft.ContainerRegistry/registries/CfAksContainerRegistry",
  "location": "switzerlandnorth",
  "loginServer": "cfakscontainerregistry.azurecr.io",
  "name": "CfAksContainerRegistry",
  "networkRuleSet": null,
  "policies": {
    "quarantinePolicy": {
      "status": "disabled"
    },
    "retentionPolicy": {
      "days": 7,
      "lastUpdatedTime": "2020-03-08T09:57:13.449314+00:00",
      "status": "disabled"
    },
    "trustPolicy": {
      "status": "disabled",
      "type": "Notary"
    }
  },
  "provisioningState": "Succeeded",
  "resourceGroup": "CfAksRg",
  "sku": {
    "name": "Basic",
    "tier": "Basic"
  },
  "status": null,
  "storageAccount": null,
  "tags": {
    "environment": "dev"
  },
  "type": "Microsoft.ContainerRegistry/registries"
}
```

do you see the field `loginServer`? this is the FQDN of our private container registry. keep it preciously we will need it later.

now we have to get an access token, in order to be able to connect on the private container registry:

    $ az acr login --name CfAksContainerRegistry --expose-token
    Argument '--expose-token' is in preview. It may be changed/removed in a future release.
    You can perform manual login using the provided access token below, for example: 'docker login loginServer -u 00000000-0000-0000-0000-000000000000 -p accessToken'
    {
    "accessToken": "...",
    "loginServer": "cfakscontainerregistry.azurecr.io"
    }

if you can connect the command send back the access token you need in order to connect to your private container registry. keep it, and register it under your GitHub repository as secret. you also need to create a secret with your subscription ID. this could be used in the CI GitHub workflow that deploy automatically your application.

but here we are going to use a slighty easier way. in your azure portal, navigate to the ACR than select "enable Admin user". this will allow GitHub to connect with an username and password, that you have to register as secrets.

from Azure Cloud Shell we can display the images hosted in the registry with this command

    $ az acr repository list --name CfAksContainerRegistry --output table

and you can get more information on an image (tags) with this command:

    $ az acr repository show-tags --name CfAksContainerRegistry --repository <your image name> --output table

### Create an Azure managed MySQL database

our application is going to use a MySQL database. we have two choices here, first one is to deploy a MYSQL helm char under AKS. second one is to use an Azure MySQL database.

we are going to use the second choice, because it's the optimal choice when running under Azure. that's cheaper, faster and much easier.

so how can we do that?

once again, let's go to Azure cloud shell, and enter the following command:

    $ az mysql server create --resource-group CfAksRg --name CfAksDbServer --location switzerlandnorth --admin-user fcblog --admin-password <server_admin_password> --sku-name B_Gen5_1 --version 5.7 --tags "environment=dev"

the SKU B_Gen5_1 is the smallest possible, and that's perfect for a test dev database.

you should see something like that if the database is sucessfully created:
```json
{
  "administratorLogin": "fcblog",
  "earliestRestoreDate": "2020-03-08T11:47:34.157000+00:00",
  "fullyQualifiedDomainName": "cfaksdbserver.mysql.database.azure.com",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/CfAksRg/providers/Microsoft.DBforMySQL/servers/cfaksdbserver",
  "location": "switzerlandnorth",
  "masterServerId": "",
  "name": "cfaksdbserver",
  "replicaCapacity": 5,
  "replicationRole": "None",
  "resourceGroup": "CfAksRg",
  "sku": {
    "capacity": 1,
    "family": "Gen5",
    "name": "B_Gen5_1",
    "size": null,
    "tier": "Basic"
  },
  "sslEnforcement": "Enabled",
  "storageProfile": {
    "backupRetentionDays": 7,
    "geoRedundantBackup": "Disabled",
    "storageAutogrow": "Enabled",
    "storageMb": 5120
  },
  "tags": {
    "environment": "dev"
  },
  "type": "Microsoft.DBforMySQL/servers",
  "userVisibleState": "Ready",
  "version": "5.7"
}
```

see the "sslEnforcement" field? it's activated by default. since this is a dev test server, we don't need to have SSL. so deactivate it with the following command:

    $ az mysql server update --resource-group CfAksRg --name CfAksDbServer --ssl-enforcement Disabled

later we will need to allow our AKS cluster to talk with this server, for that we will have to open the firewall with the following command:

    $ az mysql server firewall-rule create --resource-group CfAksRg --server CfAksDbServer --name AllowAksIP --start-ip-address <start IP> --end-ip-address <end IP>

the same can be used with an external IP, if you need to access your mySQL server from your computer for example. then use the same start and end IP which will be your WAN IP.

for now we don't know AKS cluster IPs range so let's keep this for later.

Okay, now we need to be able to connect on this database. use the following command:

    $ az mysql server show --resource-group CfAksRg --name CfAksDbServer

which will give you everything you need to connect.

    "administratorLogin": "fcblog",
    "fullyQualifiedDomainName": "cfaksdbserver.mysql.database.azure.com",
    "name": "cfaksdbserver",
 
you can then use the `mysql` command to connect (if the port is open):

    $ mysql -h cfaksdbserver.mysql.database.azure.com -u fcblog@cfaksdbserver -p

### Create AKS dev cluster

Okay, so we have our database, and our image in a private docker registry.
now it's time to create the AKS dev cluster.

you have several way to create an AKS cluster, by terraform, by ansible or with the az cli.

since we just need a very simple dev cluster, and because it's going to be a oneshot, we are going to use the az cli. for production ready cluster we should have used terraform, but not need to add complexity for this sample.

use this command to create the cluster with 1 node (for this demo we don't need more)

    $ az aks create --resource-group CfAksRg --name CfAksCluster --node-count 1 --enable-addons monitoring --generate-ssh-keys --tags "environment=dev"

Azure Monitor for containers is also enabled since we are using the `--enable-addons monitoring` parameter.

you have to wait several minutes before it's finished, and at the end you should get something like that:

```json
{
  "aadProfile": null,
  "addonProfiles": {
    "omsagent": {
      "config": {
        "logAnalyticsWorkspaceResourceID": "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/defaultresourcegroup-eus/providers/microsoft.operationalinsights/workspaces/defaultworkspace-00000000-0000-0000-0000-000000000000-eus"
      },
      "enabled": true,
      "identity": null
    }
  },
  "agentPoolProfiles": [
    {
      "availabilityZones": null,
      "count": 1,
      "enableAutoScaling": null,
      "enableNodePublicIp": null,
      "maxCount": null,
      "maxPods": 110,
      "minCount": null,
      "name": "nodepool1",
      "nodeLabels": null,
      "nodeTaints": null,
      "orchestratorVersion": "1.14.8",
      "osDiskSizeGb": 100,
      "osType": "Linux",
      "provisioningState": "Succeeded",
      "scaleSetEvictionPolicy": null,
      "scaleSetPriority": null,
      "tags": null,
      "type": "VirtualMachineScaleSets",
      "vmSize": "Standard_DS2_v2",
      "vnetSubnetId": null
    }
  ],
  "apiServerAccessProfile": null,
  "dnsPrefix": "CfAksClust-CfAksRg-f13324",
  "enablePodSecurityPolicy": null,
  "enableRbac": true,
  "fqdn": "cfaksclust-cfaksrg-f13324-6875d79d.hcp.switzerlandnorth.azmk8s.io",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/CfAksRg/providers/Microsoft.ContainerService/managedClusters/CfAksCluster",
  "identity": null,
  "identityProfile": null,
  "kubernetesVersion": "1.14.8",
  "linuxProfile": {
    "adminUsername": "azureuser",
    "ssh": {
      "publicKeys": [
        {
          "keyData": "..."
        }
      ]
    }
  },
  "location": "switzerlandnorth",
  "maxAgentPools": 10,
  "name": "CfAksCluster",
  "networkProfile": {
    "dnsServiceIp": "10.0.0.10",
    "dockerBridgeCidr": "172.17.0.1/16",
    "loadBalancerProfile": {
      "allocatedOutboundPorts": null,
      "effectiveOutboundIps": [
        {
          "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/MC_CfAksRg_CfAksCluster_switzerlandnorth/providers/Microsoft.Network/publicIPAddresses/00000000-0000-0000-0000-000000000000",
          "resourceGroup": "MC_CfAksRg_CfAksCluster_switzerlandnorth"
        }
      ],
      "idleTimeoutInMinutes": null,
      "managedOutboundIps": {
        "count": 1
      },
      "outboundIpPrefixes": null,
      "outboundIps": null
    },
    "loadBalancerSku": "Standard",
    "networkPlugin": "kubenet",
    "networkPolicy": null,
    "outboundType": "loadBalancer",
    "podCidr": "10.244.0.0/16",
    "serviceCidr": "10.0.0.0/16"
  },
  "nodeResourceGroup": "MC_CfAksRg_CfAksCluster_switzerlandnorth",
  "privateFqdn": null,
  "provisioningState": "Succeeded",
  "resourceGroup": "CfAksRg",
  "servicePrincipalProfile": {
    "clientId": "00000000-0000-0000-0000-000000000000",
    "secret": null
  },
  "tags": null,
  "type": "Microsoft.ContainerService/ManagedClusters",
  "windowsProfile": null
}
```

since we are using Azure cloud shell, no need to install `kubectl`, it's already available.

now we need to download the credentials in order to be able to use our newly created cluster with this command:

    $ az aks get-credentials --resource-group CfAksRg --name CfAksCluster
    Merged "CfAksCluster" as current context in /home/francois_colombo/.kube/config

and then we can check the cluster status:

    $ kubectl cluster-info
    Kubernetes master is running at https://cfaksclust-cfaksrg-f13324-6875d79d.hcp.switzerlandnorth.azmk8s.io:443
    healthmodel-replicaset-service is running at https://cfaksclust-cfaksrg-f13324-6875d79d.hcp.switzerlandnorth.azmk8s.io:443/api/v1/namespaces/kube-system/services/healthmodel-replicaset-service/proxy
    CoreDNS is running at https://cfaksclust-cfaksrg-f13324-6875d79d.hcp.switzerlandnorth.azmk8s.io:443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
    kubernetes-dashboard is running at https://cfaksclust-cfaksrg-f13324-6875d79d.hcp.switzerlandnorth.azmk8s.io:443/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy
    Metrics-server is running at https://cfaksclust-cfaksrg-f13324-6875d79d.hcp.switzerlandnorth.azmk8s.io:443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

    $ kubectl get nodes
    NAME                                STATUS   ROLES   AGE   VERSION
    aks-nodepool1-40205044-vmss000000   Ready    agent   10m   v1.14.8

if you want to see the K8S dashboard from your laptop, this is what you have to do:

first, install `az cli` if it's not yet done

    $ curl -L https://aka.ms/InstallAzureCli | bash

then:

    $ az login
    $ az aks install-cli --install-location $HOME/bin/kubectl
    $ az aks get-credentials --resource-group CfAksRg --name CfAksCluster
    $ kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
    $ az aks browse --resource-group CfAksRg --name CfAksCluster

et voila! now you can access the K8S dashboard.

usually, when we deploy an application under K8S, we are using a specific namespace, so we can setup resources limit usages, or even use OPA to isolate the application authorization.
we are not going to do that, since it's a very simple dev cluster with a minimal set of resources.

now let's associate our ACR with this AKS, so we will be able to use the image that we are building and deploy it under AKS.

first, go back to your portal, CfAksContainerRegistry - Access keys
copy the username and the password, and the fqdn.

go back to you Azure Cloud Shell and launch the following commands:

    $ kubectl create secret docker-registry acr-auth \
      --docker-server <fqdn> \
      --docker-username <username> \
      --docker-password <password> \
      --docker-email <your email address>

then check in your K8S dashboard that the secret is created.

we also need to create a secret for the mySQL connection string.

    $ kubectl create secret generic dev-db-secret --from-literal=connect=mysql+pymysql://fcblog%40cfaksdbserver:<password>@cfaksdbserver.mysql.database.azure.com/fcblog
