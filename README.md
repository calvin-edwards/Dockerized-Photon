# Introduction
Photon is a game networking engine and multiplayer platform developed and licensed by Exit Games. According to the [company's web site](https://www.photonengine.com/), Photon is used by various developers and studios, including Disney, Ubisoft and Oculus.

Exit Games currently provides two versions of Photon: a cloud-based service and a self-hosted server. This repo shows one way to run the self hosted server as a Windows Docker container.

This repository enables running Photon self-hosted server in a Docker container. It configures the `LoadBalancing` (`Master` and `GameServer`) Photon application, forwards all [default Photon self-hosted server ports](https://doc.photonengine.com/en-us/pun/v2/connection-and-authentication/tcp-and-udp-port-numbers) and turns performance counters on.

**NOTE**: Exit Games Photon is not free software. At the time of writting, Exit Games offers an evaluation version of Photon Self Hosted Server limited to 20 simultaneous users.

# Setup
1. Clone this repository in a convenient location, perhaps C:\DockerizedPhoton. We will call this location `<repo root>`,
2. Download [Exit Games Photon Server SDK 4.0.29.11263](https://dashboard.photonengine.com/download/photon-server-sdk_v4-0-29-11263.exe) or later. Other versions of Photon Server SDKs are available on the [Exit Games download page](https://www.photonengine.com/en-US/sdks#serverserver) (Exit Games login required),
3. Extract the SDK in a directory called Photon. The directory structure should be as follows:
    ```
        <repo root>
            | .dockerignore
            | .gitignore
            | ConfigurePhoton.ps1
            | Dockerfile
            | EntryPoint.ps1
            | Template
            |     | template.json
            |
            | Photon
            |     | build
            |     | deploy
            |     | doc
            |     | lib
            |     | src-server
            |
            | LICENSE
            | README.md
    ```
4. Make sure Docker for Windows is setup for Windows Containers and is running.

## Building and running locally
1. Open a Powershell window and cd into `<repo root>`
2. Build and tag the image `photon:1.0` by running:
    ```
    docker build -t photon:1.0 .
    ```
    Docker will pull Windows Server Core 1809 from the Microsoft Image Registry `mcr.microsoft.com/windows/servercore:1903` if needed. This may take a while.
3. Create a custom NAT Docker network to run Photon locally. This only needs to be done once:
    ```
    docker network create --driver=nat --subnet=172.24.1.0/24 --gateway=172.24.1.1 photon-nat
    ```
4. Run the container:
    ```
    docker run --rm --interactive --tty \
        --network photon-nat --ip 172.24.1.20 \
        -e PHOTON_ENDPOINT=172.24.1.20 \
        -p 843:843/tcp -p 4530:4530/tcp -p 4531:4531/tcp -p 4533:4533/tcp -p 5055:5055/udp \
        -p 5056:5056/udp -p 5058:5058/udp -p 6060:6060/tcp -p 6061:6061/tcp  -p 6063:6063/tcp \
        -p 9090:9090/tcp -p 9091:9091/tcp -p 19090:19090/tcp -p 19091:19091/tcp -p 19093:19093/tcp \
        photon:1.0
    ```

    By default the image starts the standard `LoadBalancing` application. The Photon server is ready when the container displays the PID of `PhotonSocketServer` as follows:
    ```
    DNS Name: 172.24.1.20
    Public IP: 172.24.1.20
    Configuring Master (Photon.LoadBalancing.dll.config)
    Configuring GameServer (Photon.LoadBalancing.dll.config)
    Starting PhotonSocketServer
    PhotonSocketServer has PID '1976'
    Waiting for PhotonSocketServer to exit
    ```
    Photon Server is now available at `172.24.1.20` (the value passed as `PHOTON_ENDPOINT`) and games can now connect.

# Deploy to Azure Container Instance
You will need an active Azure subscription and an [Azure Image Registry](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-portal) instance. A "Basic" SKU is sufficient. The following steps assume Administrative User access has been enabled. You will need to substitute `<registry login server>`, `<registry user name>` and `<registry password>` with actual values for your registry. These can be found under the "Access keys" blade in the Azure portal.

1. Tag the image with the registry login server. You can either build and tag the image in one step:
    ```
    docker build -t <registry login server>/gameserver/photon:1.0 .
    ```
    or tag an existing image:
    ```
    docker tag photon:1.0 <registry login server>/gameserver/photon:1.0
    ```
1. Login to the registry and push the image. Provide `<registry user name>` and `<registry password>` if asked:
    ```
    docker login <registry login server>
    docker push <registry login server>/gameserver/photon:1.0
    ```
2. Run the Photon image in an instance of [Azure Container Instance](https://docs.microsoft.com/en-us/azure/container-instances/). This can be done via [Azure Powershell](https://docs.microsoft.com/en-us/powershell/azure/overview?view=azps-2.1.0) :
    ```
    New-AzureRmResourceGroupDeployment `
            -ResourceGroupName <resource group name> `
            -TemplateFile Template\template.json `
            -imageTag <registry login server>/gameserver/photon:1.0 `
            -containerRegistryServer <registry login server> `
            -containerRegistryUsername <registry user name> `
            -containerRegistryPassword <registry password>
    ```
    or [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) :
    ```
    az group deployment create \
            --resource-group <resource group name> \
            --template-file Template\template.json \
            --parameters \
                imageTag=<registry login server>/gameserver/photon:1.0 \
                containerRegistryServer=<registry login server> \
                containerRegistryUsername=<registry user name> \
                containerRegistryPassword=<registry password> 
    ```
    If the Container Instance Group exists, it is updated. Caution: all containers in the group are stopped first.

The previous commands provide `<registry user name>` and `<registry password>` on the command line. It is best to store these secrets in [Azure Key Vault](https://docs.microsoft.com/en-us/azure/key-vault/) and retrieve them programatically during deployment. For instance the following command will retrieve the secret `myregsitry-admin-pass` from the Key Vault named `mykeyvault`. For more information, refer to [](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-using-azure-container-registry).

```
AKV_NAME=mykeyvault
$(az keyvault secret show --vault-name $AKV_NAME -n myregsitry-admin-pass --query value -o tsv)
```


# TODO
1. Collect logs (possibly with Log Analytics or Aplication Insights)
2. Understand versioning in ACI - seems like https://blogs.msdn.microsoft.com/stevelasker/2018/03/01/docker-tagging-best-practices-for-tagging-and-versioning-docker-images/ does niot automatically work
3. Add a troubleshooting section
4. Consider making enable perf counter an option to the container
5. Consider a custom counter publisher for log analytics
