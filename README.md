# AKS_StorageAccount_Using_Managed_Identity
### In this document, I have mentioned that how a pod of an AKS Cluster can communicate with Storage account using "user_assigned" managed identity.

## Create AKS Cluster using below command:
```
az aks create --resource-group <Resource_Group_Name> --name <AKS_Cluster_Name> --enable-managed-identity
```

## Create Storage Account using below command:
```
az storage account create -n <Storage_Account_Name> -g <Resource_Group_Name> -l southeastasia  --sku Standard_LRS
```

## Create managed identity using below command:
```
az identity create --name <Managed_Identiy_Name> --resource-group <Resource_Group_Name>
```

## Connect AKS Cluste service with Storage Account:
 ```
> Go to AKS Cluster.
> Go to Settings
> Go to Service Connector
> Click On Connect to your services
> Service Type: Storage - Blob
> Storage Account: <Storage_Account_Name>
> Client Type: None
> Next
> User Assigned Managed Identity:  <Managed_Identiy_Name>
> Next
> Review + Create
> Create
```

## Update an Azure Kubernetes Service (AKS) cluster to use a User-Assigned Managed Identity
```
az aks update --resource-group <Resource_Group_Name> --name <AKS_Cluster_Name> --enable-managed-identity --assign-identity /subscriptions/66xxxxxx7-1xx2-4xx3-bxx5-axxxxxxxf6b/resourceGroups/<Resource_Group_Name>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<Managed_Identiy_Name>
```

## Assigns the "Storage Blob Data Contributor" role to a service for a specific Azure Storage Blob container
```
az role assignment create --assignee 60xxxxea-6xxe-4xx7-8xxb-bxxxxxxxxx28 --role "Storage Blob Data Contributor" --scope "/subscriptions/66xxxxx7-1xx2-4xx3-bxx5-a4axxxxxxxf6b/resourceGroups/<Resource_Group_Name>/providers/Microsoft.Storage/storageAccounts/<Storage_Account_Name>/blobServices/default/containers/<Container_Name>"
```

## Below are the python code and kubernetes manifest files whcih will be needed:

## requirements.txt:
```
azure-identity
azure-storage-blob
flask
```

## sample-source.txt:
```
Hello there, Azure Storage. I'm a friendly file ready to be stored in a blob.
```

### app.py
```
import os
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient, BlobClient

# Authenticate using default credentials
credential = DefaultAzureCredential()

# Define storage account URL
storage_url = "https://<Storage_Account_Name>.blob.core.windows.net/"

# Create a BlobServiceClient to manage the storage account
blob_service_client = BlobServiceClient(
    account_url=storage_url,
    credential=credential,
)

# Create a new container
container_name = "newcontainer"
try:
    container_client = blob_service_client.create_container(container_name)
    print(f"Container '{container_name}' created successfully at {container_client.url}")
except Exception as e:
    print(f"Container creation failed: {e}")

# Define blob name and file path
blob_name = "random.txt"
local_file_path = "/etc/sample-source/sample-source.txt"

# Create a BlobClient for the specific file
blob_client = blob_service_client.get_blob_client(container=container_name, blob=blob_name)

# Upload the file to the created container
try:
    with open(local_file_path, "rb") as data:
        blob_client.upload_blob(data)
        print(f"Uploaded {local_file_path} to {blob_client.url}")
except Exception as e:
    print(f"File upload failed: {e}")

```

### Before applying the k8s manifest files, you need to creeate configmaps in your k8s cluster:

```
kubectl create configmap sample-source --from-file=sample-source.txt=/location/to/file/sample-source.txt
 

 kubectl create configmap app-py --from-file=app.py=/location/to/file/app.py
 

 kubectl create configmap requirements-txt --from-file=requirements.txt=/location/to/file/requirements.txt
```

## identity.yaml:
```
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentity
metadata:
  name: azure-access
spec:
  type: 0
  resourceID: /subscriptions/664b6097-19f2-42a3-be95-a4a6b4069f6b/resourceGroups/<Resource_Group_Name>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<ManagedIdentityName>
  clientID: 0xxxxxxxx5-7xx0-4xx6-8xx1-2bxxxxxxxxd
```

## identity-binding.yaml:
```
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentityBinding
metadata:
  name: azure-access
spec:
  azureIdentity: azure-access
  selector: azure-access
```

## pod.yaml:
```
apiVersion: v1
kind: Pod
metadata:
  name: podidentitydemo
  labels:
    aadpodidbinding: azure-access
spec:
  containers:
    - name: podidentitydemo
      image: python:3.8
      command: ["bash", "-c", "pip install -r /etc/config/requirements.txt && python /etc/app/app.py"]
      ports:
        - containerPort: 5000
      env:
        - name: AZURE_CLIENT_ID
          value: "601XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXa28"
        - name: AZURE_TENANT_ID
          value: "0adXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX2ff"
        - name: AZURE_CLIENT_SECRET
          value: "dpXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXfq"
      volumeMounts:
        - name: requirements-volume
          mountPath: /etc/config  
        - name: app-volume
          mountPath: /etc/app
        - name: sample-source 
          mountPath: /etc/sample-source  
  volumes:
    - name: requirements-volume
      configMap:
        name: requirements-txt
    - name: app-volume
      configMap:
        name: app-py
    - name: sample-source
      configMap:
        name: sample-source
```    

## Run the below commands to deploy:
```
kubectl apply -f identity.yaml
kubectl apply -f identity-binding.yaml
kubectl apply -f pod.yaml 
```