
# Azure Demo Infrastructure with Azure CLI


## Ubuntu Virtual Machine

- create a resource group
```
az group create --name benDemoResourceGroup --location westeurope
az group list
```

- Create a virtual network and subnet
```
az network vnet create \
    --resource-group benDemoResourceGroup \
    --name benDemoVnet \
    --address-prefix 192.168.0.0/16 \
    --subnet-name benDemoSubnet \
    --subnet-prefix 192.168.1.0/24
```

- Create public IP
```
az network public-ip create \
    --resource-group benDemoResourceGroup \
    --name benDemoVM01PublicIP \
    --dns-name bendemopublicdns
```

- Create VM
```
az vm create \
  --resource-group benDemoResourceGroup \
  --name benDemoVM01 \
  --image UbuntuLTS \
  --generate-ssh-keys \
  --public-ip-address benDemoVM01PublicIP \
  --vnet-name benDemoVnet \
  --subnet benDemoSubnet
```
default user here is `bunyamin`. to specify username `--admin-username azureuserblabla`


- if you want to connect to the vm: Provide a path to your SSH private key file. Private key path: `~/.ssh/bunyamin` Run the example command below to connect to your VM.

``` 
ssh -i <private key path> bunyamin@bendemopublicdns.westeurope.cloudapp.azure.com
```


- by default outbound https port is open. If you want to test the connection to github:

```
telnet www.github.com 443
```


## Postgres DB in a Virtual Network

- now we create a postgresdb. but first we have to list configuration options:
```
az postgres server list-skus -l westeurope
```

- it may give an error about subscription id. do this: 
`az account list` to show your subscription id
`az account set --subscription <subscr id>` to set that as your default subscription id

- Now, create postgresdb
```
az postgres server create \ 
  --resource-group benDemoResourceGroup \
  --name benDemoPostgres01 \
  --location westeurope \
  --admin-user bunyamin \
  --admin-password Mytestpostgres123 \
  --sku-name B_Gen5_1
  --storage-size 10240
```
...but it didnt like this format. try one liner:
```
az postgres server create   --resource-group benDemoResourceGroup  --name benDemoPostgres01  --location westeurope  --admin-user bunyamin --admin-password Mytestpostgres123 --sku-name B_Gen5_1 --storage-size 10240
```
for general purpose tier use: `--sku-name GP_Gen5_2`

- to put your postgres in the same virtual network with the VM:
```
az postgres server vnet-rule create \
-n benDemoRule \
-g benDemoResourceGroup \
-s benDemoPostgres01 \
--vnet-name benDemoVnet \
--subnet benDemoSubnet
```
this didnt worked. because in order to put your postgres in a Virtual network, a virtual network service endpoint for postgresDB has to be created and added to vnet. for this, your DB has to be in general purpose pricing tier. NOT BASIC. i will delete my db and create it again in that tier. and then update your vnet:
```
az network vnet subnet update -n benDemoSubnet --vnet-name benDemoVnet -g benDemoResourceGroup --service-endpoints Microsoft.SQL
```
Test network connectivity between our VM and our postgresDB server:
`telnet bendemopostgres01.postgres.database.azure.com 5432`
Connected!

- Now we have our endpoint. we can disable public network acces to our DB. to check if it is public accesible or already disabled to public:
```
az postgres server show -n benDemoPostgres01 -g benDemoResourceGroup --query "publicNetworkAccess"
```
- to disable:
```
az postgres server update -n benDemoPostgres01 -g benDemoResourceGroup --set publicNetworkAccess="Disabled"
```
test again if we still can connect to our DB from VM:
`telnet bendemopostgres01.postgres.database.azure.com 5432`
Connected!



## Shared Mountable Storage

- Now we start to create a shared mountable storage :


-first a storage account and type etc..
```
export storageAccountName="benstorageacct$RANDOM"
az storage account create \
    --resource-group benDemoResourceGroup \
    --name $storageAccountName \
    --location westeurope \
    --kind StorageV2  \
    --sku Standard_LRS \
    --enable-large-file-share
```
storage account name = benstorageacct32693

- Storage acoount key:
```
export storageAccountKey=$(az storage account keys list \
    --resource-group benDemoResourceGroup \
    --account-name $storageAccountName \
    --query "[0].value" | tr -d '"')
```
[Azure CLI query:](https://docs.microsoft.com/en-us/cli/azure/query-azure-cli) 
Even when using an output format other than JSON, CLI command results are first treated as JSON for queries. CLI results are either a JSON array or dictionary. bc of that, results has to be queried according to json key names (case sensitive). if your default output is table format rather than json, to see the json format, just put `-o json` at the end of your command. and then use `--query` to filter your output and get the exact result which you want. in this case, az list command gives us a list of keys:
```
[
  {
    "keyName": "key1",
    "permissions": "FULL",
    "value": "Vdw+tMJXbSKf5WVp0d1BI2dBElis1LV1M/jfCyjXz/ZPFQCYtS2sSnCFW6G1JVlHwDQ7gJO1bHM9Oj6uRSs4dA=="
  },
  {
    "keyName": "key2",
    "permissions": "FULL",
    "value": "c4ZQIZSg2d34ECUkAvaSPp99FVH+bklsFHdRJ765celM7KxK4HJ3oD54VBa5aAnHP8FkPPLhy6sRNWtA4DM7Ow=="
  }
]
```
and the tr command (=transform) with -d option deletes the quotes caracters in that string.


- create file share. that creates a 20GB shared storage:
```

az storage share-rm create \
    --resource-group benDemoResourceGroup \
    --storage-account $storageAccountName \
    --name bensharestorage \
    --quota 20 \
    --enabled-protocols SMB
```
share name has to contain only lowercase letters, numbers, and single hyphens (but they can't start with a hyphen).


- Now we can mount it from our VM, using Samba.
- prerequisites:
- cifs-utils package has to be installed on VM. 
```
sudo apt update
sudo apt install cifs-utils
```
- azure cli must be installed `curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash`
- TCP 445 port has to be open. to check: (but before this, login wit az login command)
```
# This command assumes you have logged in with az login
httpEndpoint=$(az storage account show \
    --resource-group benDemoResourceGroup \
    --name benstorageacct32693 \
    --query "primaryEndpoints.file" | tr -d '"')
smbPath=$(echo $httpEndpoint | cut -c7-$(expr length $httpEndpoint))
fileHost=$(echo $smbPath | tr -d "/")

nc -zvw3 $fileHost 445
```
connection succeeded!


- if everything is ok, create a directory to mount your shared storage
`sudo mkdir /sharedstorage`

- and the mount
```
# This command assumes you have logged in with az login
httpEndpoint=$(az storage account show \
    --resource-group benDemoResourceGroup \
    --name benstorageacct32693 \
    --query "primaryEndpoints.file" | tr -d '"')
smbPath=$(echo $httpEndpoint | cut -c7-$(expr length $httpEndpoint))bensharestorage

storageAccountKey=$(az storage account keys list \
    --resource-group benDemoResourceGroup \
    --account-name benstorageacct32693 \
    --query "[0].value" | tr -d '"')

sudo mount -t cifs $smbPath /sharedstorage -o vers=3.0,username=benstorageacct32693,password=$storageAccountKey,serverino
```
check with `df -kh` to see yout mounted storage


## Blob Storage Container and upload files to it from VM 

- Create a container (it corresponds to a S3 bucket in AWS world)

```
storageAccountKey=$(az storage account keys list \
    --resource-group benDemoResourceGroup \
    --account-name benstorageacct32693 \
    -o tsv \
    --query "[0].value" | tr -d '"')
```
`-o tsv` parameter formats the output to a tab seperated view but without the keys. its useful for using values in a script.   

```
az storage container create \
    --account-name benstorageacct32693 \
    --name ben-demo-container \
    --account-key $storageAccountKey \
    --auth-mode key
```
- `--auth-mode` parameter can also be `login` that means authenticate it with your azure active directory credentials. the you wont need a `--account-key` parameter. 


- to upload a file from VM: (assuming there is a `testfile` under `testfolder`. if not create one with `vi` and write something to it.)
```
az storage blob upload \
    --account-name benstorageacct32693 \
    --container-name ben-demo-container \
    --name helloworld \
    --file testfolder/testfile \
    --auth-mode key \
    --account-key $storageAccountKey
```
with `-- file` parameter we give the path of our file. `--name` parameter describes what we want the name of this file  to be in cointer.

- to list the content of the container:
```
az storage blob list \
    --account-name benstorageacct32693 \
    --container-name ben-demo-container \
    --auth-mode key \
    --account-key $storageAccountKey \
    --output table
```

- to download a blob (=file in container)
```
az storage blob download \
    --account-name benstorageacct32693 \
    --container-name ben-demo-container \
    --name helloworld \
    --file ~/destination/path/for/file \
    --auth-mode key \
    --account-key $storageAccountKey \
```

- there is also another option for transfering data in a scriptable way: `azcopy` command. it allows also copy data from AWS S3 and Google Cloud Storage. Here is the documentation: [Get started with AzCopy](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10) 

## Clean up resources

- to delete everything
```
az group delete \
    --name <resource-group> \
    --no-wait
```

