# sample

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

TO BE COMPLETED WITH DEPLOYMENT