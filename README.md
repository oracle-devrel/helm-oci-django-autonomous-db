# helm-oci-django-atp
[![License: UPL](https://img.shields.io/badge/license-UPL-green)](https://img.shields.io/badge/license-UPL-green) [![Quality gate](https://sonarcloud.io/api/project_badges/quality_gate?project=oracle-devrel_terraform-oci-arch-ci-cd)](https://sonarcloud.io/dashboard?id=oracle-devrel_terraform-oci-arch-ci-cd)


## Introduction

This Helm Chart will demonstrate how to automate the deployment of a web server built with the Django Web Framework and connect it to an Oracle Autonomous Database instance (ATP)  in a Container Engine for Kubernetes (OKE) cluster. Installing this Helm chart will allow you to create a new Autonomous Database instance or connect to an existing instance in Oracle Cloud Infrastructure (OCI). A custom resource definition (CRD)  will be produced as designation for the  ATP and can be managed like any Kubernetes resource with the command-line tool, kubectl.

An API server built with the Django Web Framework will be deployed to provide access to a user-profile database on the Autonomous Database to perform create, read, update and delete (CRUD) operations. The API server will be accessible via a load balancer provisioned with an external IP address.

This Helm chart relies on the OCI Service Operator for Kubernetes (OSOK), and it is a pre-requisite to have OSOK deployed within the cluster to use this Helm chart.


## Pre-requisites

- A Kuberntes cluster deployed in OCI 
- [OCI Service Operator for Kuberntes (OSOK) deployed in the cluster](https://github.com/oracle/oci-service-operator/blob/main/docs/installation.md)
- [kubectl installed and using the context for the Kubernetes cluster where the ATP resource will be deployed](https://kubernetes.io/docs/tasks/tools/)
- [Helm installed](https://helm.sh/docs/intro/install/)


##  Getting Started

**1. Clone or download the contents of this repo** 
     
     git clone https://github.com/chiphwang1/helm-oci-django-atp.git
**2. Change to the directory that holds the Helm Chart** 

      cd ./helm-oci-django-atp.git  
        
**3. Populate the values.yaml file with the required information**   

**4. Create the namespace where the ATP resource will be deployed**

     kubectl create ns <namespace name>

**5. Install the Helm chart.**

A database and wallet passwords needs to be created for the Autonomous Database. You can define the passwords in the values.yaml file, but it is best practice not to have passwords written to code. You can specify the passwords during the installation of the Helm chart with the following command.


     helm -n <namespace name> install \
     --set dbPassword=<database password> \  
     --set walletPassword=<wallet password> \
       <name for this install> .
  
  ***Example:***
     helm -n autodb  --set dbPassword=Admin!2345  --set walletPassword=Admin!2345 autodb .   
     
The password must be between 8 and 32 characters long, and must contain at least 1 numeric character, 1 lowercase character, 1 uppercase character, and 1        special (nonalphanumeric) character.


**6. List the Helm installation**

     helm  -n <namespace name> ls


**7. Connect to the API Server**  
     The API server can be connected with the external IP address of the load balancer with the /api extension.
     
   ***Example:***
   ```
   http://<ip address of load balancer>/api
   
   ```  
   To retrieve the IP adddress of the load balancer use the following command
   
```sh
$ kubectl -n <name of namespace> get svc django-service

NAME             TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)        AGE
django-service   LoadBalancer   10.96.253.175   129.153.142.122   80:32438/TCP   9h
```
     
**8. To uninstall the Helm chart**

     helm uninstall -n <namespace name> <name of the install> .
 
Uninstalling the helm chart will only remove the Autonomous Database resource from the cluster and not from OCI. You will need to use the console or the OCI CLI to remove the ATP from OCI. This function is to prevent accidental deletion of the database.

     
  **Notes/Issues:**
 
 Provisioning the Autonomous Database (ATP) can take up to 5 minutes. 
 
 To confirm that the ATP is active, run the following command and check the status of the ATP system.

```sh
$ kubectl -n <name of namespace> get autonomousdatabases -o wide
NAME            DISPLAYNAME     DBWORKLOAD   STATUS   OCID                                                                                            AGE
autodbtest301   autodbtest301   OLTP         Active   ocid1.autonomousdatabase.oc1.iad.anuwcljsnlc5nbyazyyzlqxytdmghb5eyafntqnxq6cupu3zmxf6jihz6vna   28m
```

 ## How to connect to an Autonomous Database
 
 Oracle client credentials (wallet files) are required to connect to an Autonomous Database. Typically wallet files are downloaded from the OCI console or CLI, but the wallet files are also exposed as a secret within the Kubernetes cluster. The name of the secret is defined in values.yaml file under wallet.walletName and can be extracted with the kubectl command-line. 
 
 In this Helm chart, the secret is extracted with a python script. An example of the script is provided in the python folder in this repository.

The following are the files kept in the secret with the key as the file name and the values as the file contents.

| Parameter          | Description                                                              | Type   |
| ------------------ | ------------------------------------------------------------------------ | ------ |
| `ewallet.p12`      | Oracle Wallet.                                                           | string |
| `cwallet.sso`      | Oracle wallet with autologin.                                            | string |
| `tnsnames.ora`     | Configuration file containing service name and other connection details. | string |
| `sqlnet.ora`       |                                                                          | string |
| `ojdbc.properties` |                                                                          | string |
| `keystore.jks`     | Java Keystore.                                                           | string |
| `truststore.jks`   | Java trustore.                                                           | string |
| `user_name`        | Pre-provisioned DB ADMIN Username.                                       | string |



## Requirements to connect Django Web Framework to an Autonomous Database. 

**These steps are automated in this Helm Chart. This is to provide guidance to attach your Django applications to an Autonomous Database when 
creating your container images**.

The Django Web Framework requires the following prerequistes to to connect to an Autonomous Database.

**1. The Oracle instant client needs to be installed on the web Django web server** 
     
     https://www.oracle.com/database/technologies/instant-client/downloads.html
     
     To install the client on a Ubuntu server use the following commands.
    
     sudo apt install alien libaio1
     wget https://download.oracle.com/otn_software/linux/instantclient/215000/oracle-instantclient-basic-21.5.0.0.0-1.x86_64.rpm
     sudo alien -i oracle-instantclient-basic-21.5.0.0.0-1.x86_64.rpm
     
    
     The wallet files need to be copied to  /usr/lib/oracle/21/client64/lib/network/admin/ directory.

**2. The cx_Oracle python libary needs to be installed on the Django web server** 
     
     
     pip install cx_Oracle
     

**3. The settings.py on the Django server needs to be modified to use an Oracle Database**

```

# Database
# https://docs.djangoproject.com/en/2.2/ref/settings/#databases

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.oracle',
        'NAME': os.environ.get('DBNAME'),
        'USER': os.environ.get('DBUSER'),
        'PASSWORD': os.environ.get('DBPASSWORD'),
    }
}

```



## Autonomous Database Specification Parameters

The Complete Specification of the `AutonomousDatabase` Custom Resource (CR) is as detailed below:

| Parameter                          | Description                                                         | Type   | Mandatory |
| ---------------------------------- | ------------------------------------------------------------------- | ------ | --------- |
| `spec.id` | The Autonomous Database [OCID](https://docs.cloud.oracle.com/Content/General/Concepts/identifiers.htm). | string | no  |
| `spec.displayName` | The user-friendly name for the Autonomous Database. The name does not have to be unique. | string | yes       |
| `spec.dbName` | The database name. The name must begin with an alphabetic character and can contain a maximum of 14 alphanumeric characters. Special characters are not permitted. The database name must be unique in the tenancy. | string | yes       |
| `spec.compartmentId` | The [OCID](https://docs.cloud.oracle.com/Content/General/Concepts/identifiers.htm) of the compartment of the Autonomous Database. | string | yes       |
| `spec.cpuCoreCount` | The number of OCPU cores to be made available to the database. | int    | yes       |
| `spec.dataStorageSizeInTBs`| The size, in terabytes, of the data volume that will be created and attached to the database. This storage can later be scaled up if needed. | int    | yes       |
| `spec.dbVersion` | A valid Oracle Database version for Autonomous Database. | string | no        |
| `spec.isDedicated` | True if the database is on dedicated [Exadata infrastructure](https://docs.cloud.oracle.com/Content/Database/Concepts/adbddoverview.htm).  | boolean | no       |
| `spec.dbWorkload`  | The Autonomous Database workload type. The following values are valid:  <ul><li>**OLTP** - indicates an Autonomous Transaction Processing database</li><li>**DW** - indicates an Autonomous Data Warehouse database</li></ul>  | string | yes       |
| `spec.isAutoScalingEnabled`| Indicates if auto scaling is enabled for the Autonomous Database OCPU core count. The default value is `FALSE`. | boolean| no        |
| `spec.isFreeTier` | Indicates if this is an Always Free resource. The default value is false. Note that Always Free Autonomous Databases have 1 CPU and 20GB of memory. For Always Free databases, memory and CPU cannot be scaled. | boolean | no |
| `spec.licenseModel` | The Oracle license model that applies to the Oracle Autonomous Database. Bring your own license (BYOL) allows you to apply your current on-premises Oracle software licenses to equivalent, highly automated Oracle PaaS and IaaS services in the cloud. License Included allows you to subscribe to new Oracle Database software licenses and the Database service. Note that when provisioning an Autonomous Database on [dedicated Exadata infrastructure](https://docs.oracle.com/iaas/Content/Database/Concepts/adbddoverview.htm), this attribute must be null because the attribute is already set at the Autonomous Exadata Infrastructure level. When using [shared Exadata infrastructure](https://docs.oracle.com/iaas/Content/Database/Concepts/adboverview.htm#AEI), if a value is not specified, the system will supply the value of `BRING_YOUR_OWN_LICENSE`. <br>Allowed values are:<ul><li>LICENSE_INCLUDED</li><li>BRING_YOUR_OWN_LICENSE</li></ul>. | string | no       |
| `spec.freeformTags` | Free-form tags for this resource. Each tag is a simple key-value pair with no predefined name, type, or namespace. For more information, see [Resource Tags](https://docs.oracle.com/iaas/Content/General/Concepts/resourcetags.htm). `Example: {"Department": "Finance"}` | string | no |
| `spec.definedTags` | Defined tags for this resource. Each key is predefined and scoped to a namespace. For more information, see [Resource Tags](https://docs.oracle.com/iaas/Content/General/Concepts/resourcetags.htm). | string | no |
| `spec.adminPassword.secret.secretName` | The Kubernetes Secret Name that contains admin password for Autonomous Database. The password must be between 12 and 30 characters long, and must contain at least 1 uppercase, 1 lowercase, and 1 numeric character. It cannot contain the double quote symbol (") or the username "admin", regardless of casing. | string | yes       |
| `spec.wallet.walletName` | The Kubernetes Secret Name of the wallet which contains the downloaded wallet information. | string | yes       |
| `spec.walletPassword.secret.secretName`| The Kubernetes Secret Name that contains the password to be used for downloading the Wallet. | string |  no  |

## Useful commands 


**1. To check the status of the Autonomous Database System run the following command**
     
     kubectl -n <namespace of autonomousdatabase> get autonomousdatabases -o wide

**2. To describe the  Autonomous Database System  run the following command** 
     
     kubectl -n <namespace of autonomousdatabase> describe autonomousdatabases 

**3. To retreive the OCID of Autonomous Database System run the following command** 

      kubectl -n <namespace of autonomousdatabase> get autonomousdatabase <name of mysqldbsystem> -o jsonpath="{.items[0].status.status.ocid}
      

**4. To retrive the wallet password of the Autonomous Database System run the following command**
     
    kubectl -n   <autonomousdatabase>   get secret <name of wallet secret>  -o  jsonpath="{.data.walletPassword}" | base64 --decode





## Additional Resources

- [OCI Service Operator for Kuberntes (OSOK) deployed in the cluster](https://github.com/oracle/oci-service-operator)
- [OCI Autonomous Database Serice OSOK](https://github.com/oracle/oci-service-operator/blob/main/docs/adb.md)
- [Oracle Autonomous Database](https://www.oracle.com/database/what-is-autonomous-database/)
- [Developing Python Applications for Oracle ATP](https://www.oracle.com/database/technologies/appdev/python/quickstartpython.html)


## License
Copyright (c) 2022 Oracle and/or its affiliates.

Licensed under the Universal Permissive License (UPL), Version 1.0.

See [LICENSE](LICENSE) for more details.
