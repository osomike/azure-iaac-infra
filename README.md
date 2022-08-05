# Data Platform template for Azure using Terraform
This is a repository intended to deploy a data platform infrastructure using terraform in azure

For its use, you will have to:

1. Install [terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)

2. Install [azure-cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)<br/>
3. Loggin into azure with the command:
```console
az login
```

---
### Terraform commands

To initialize the terraform files run the following command inside the folder containing the ***.tf** files:
```console
terraform init
```

Once the terraform file have been initialized you can run the following commands:<br/><br/>
To format the terraform files run:
```console
terraform fmt
```

To validate the terraform files run:
```console
terraform validate
```

To check the deployment plan run:
```console
terraform plan
```


To deploy the configuration run:
```console
terraform apply
```


To delete/destroy the configuration deployed run:
```console
terraform destroy
```

---

### Additional terraform commands create resources in existing resource group

[Optional] If you want to create the resources in an existing resource group, you have to import this existing resource group before running the next commands. Make sure you comment the Databricks terraform provider details before you import the resource group (otherwise an error will occur). And uncomment this part when you run `terraform apply`. 
This can be achieved by using the following import statement:
```console
terraform import <Terraform Resource Name>.<Resource Label> <Azure Resource ID>
```

* Terraform Resource Name: The name of the resource group within the .tf file (often 'azurerm_resource_group')
* Resource Label: The label given to the resource group in the .tf file (often 'rg')
* Azure Resource ID: Can be found by running the following command:
  * ```az group show --name <Terraform Resource Name> | grep id```

Example command:
```console
terraform import azurerm_resource_group.rg /subscriptions/ef0661c5-0e9a-4467-ba85-e57a8816570d/resourceGroups/dip-tst-46s4e6-rg
```

For more details check [this](https://cloudskills.io/blog/terraform-azure-07) post.

---
### Additional configuration required

#### 1 Create a Service Principal secret
In order to authenticate from different services within Azure, a secret to the Service Principal will be required.
According this template the service principal should be named as **{prefix}-{environment}-{random_id}-terraform-app-01**. Where ***prefix*** and ***environment*** are variables defined in the **variables.tf** file
- Go to **Azure Active Directory**.
- Click on **Properties** on the left menu.
- On this interface you will be able to see the **Tenant ID** value. It is stored under the field **Tenant ID**. Copy and save it.
- Click on **App registrations** on the left menu and find the Application used as service principal. Ex: **{prefix}-{environment}-{random_id}-terraform-app-01**
- Click on the application name, that will open the application configuration and properties menu.
- On this interface you will be able to see the Application/Client ID value. It is stored under the field **Application (client) ID**. Copy and save it.
- Go to **Certificates & secrets** on the left menu.
- Click on **New client secret**. Add a description for this new secret and define its expiration time.
- Copy the value of the created secret. It will be visible only once, after its creation. Save that value.

#### 2 Add Secrets to Azure Key Vault
This section is intended to store secret values on azure key vault. If you followed this template you should have deployed an Azure Key Vault instance on your resource group called **{prefix}-{environment}-{random_id}-key-vault**. Where ***prefix*** and ***environment*** are variables defined in the **variables.tf** file.</br>

These steps show how to add the **Service Principal** secret to the key vault.</br>

- Go to the **Azure Key Vault** instance named **{prefix}-{environment}-{random_id}-key-vault**.
- Click on **Secrets** on the left menu.
- Click on **+ Generate/Import** button.
- Enter the name of your secret. Ex: **{prefix}-{environment}-{random_id}-terraform-app-01-secret**
- Enter the value of your secret. You got it on the previous section.
- Enter a content type. Ex: "Secret for **{prefix}-{environment}-{random_id}-terraform-app-01-secret**"
- Click on **Create**

Repeat the same steps for the **Application ID** and **Tenant ID** values (you also got them on the previous section).

#### 3 Create an Azure Key Vault-backed Secret Scope in Azure Databricks
This section details how to link a Secret Scope in Databricks to an Azure Key Vault instance.</br>

If you followed this template you should have deployed an Azure Key Vault instance on your resource group called **{prefix}-{environment}-{random_id}-key-vault**. Where ***prefix*** and ***environment*** are variables defined in the **variables.tf** file.</br>

- Go to the **Azure Key Vault** instance named **{prefix}-{environment}-{random_id}-key-vault**.
- Click on **Properties** on the left menu.
- On this interface you will be able to see the **Vault URl** and **Resource ID** values. Copy and save them.
- On Databricks. Go to **https://{databricks-instance}#secrets/createScope** replace {databricks-instance} with your actual Databricks instance URL.
- Add the **Scope name**. For example: **{prefix}-{environment}-{random_id}-key-vault**.
- For the **Manage Principal** field select **All Users**.
- The **DNS Name** is the **Vault URl** value you previously copied from the Azure Key Vault instance properties.
- The **Resource ID** is the **Resource ID** value you previously copied from the Azure Key Vault instance properties.
- Click on **Create**.

#### 4 Mount the ADSL Gen2 container in the databricks cluster
This section show how to mount the container created on the Storage account.

Inside Databricks, run the script/notebook on the folder ***/notebooks/adls_gen2_mount_into_databricks_script.py***</br>

For more details check [this](https://towardsdatascience.com/mounting-accessing-adls-gen2-in-azure-databricks-using-service-principal-and-secret-scopes-96e5c3d6008b) post.

---
### CI/CD for development and production environments

A proper implementation of a data platform should contain ***two different infrastructures***, one for ***production*** and another for ***development***.<br/>

The code of the application will be hosted on a version controlled system, and should contain the branches ***master*** and ***development***, which must be properly plugged to the production and development infrastructure respectively.<br/>

A schema of it is shown below:

![Drag Racing](imgs/img_prod_dev_infra_git.png)

If Azure Data Factory and Databricks are going to be used together we recommend to place the code on two different repositories:
1. Repository for the code in Databricks.
2. Repository for the code (json files) in Azure Data Factory.

Setting up the repositories for Databricks is trivial, just go to ***Repos*** section on your workspace and click 'Add Repo' (in this menu you can clone a remote git repo):
- Databricks code in ***master*** branch for ***prd*** infrastructure: **{prefix}-prd-{random_id}-databricks**
- Databricks code in ***development*** branch for ***dev*** infrastructure: **{prefix}-dev-{random_id}-databricks**

For Azure Data Factory, the implementation needs additional steps. The Data Factory development instance **{prefix}-dev-{random_id}-datafactory**
will be plugged to the collaboration branch development. For it go **Manage** -> **Git configuration** and set the collaboration branch parameter as the **development** branch and the **publish** branch as **adf_publish** (this branch is automatically created by Azure):

![ADF Git config for DEV](imgs/adf_git_config_for_dev.png)

The idea is that inside the development data factory instance you will create additional branches where new features 
will be added. Once those features have been tested they will be merged to the collaboration branch (development) via 
pull requests.</br>

After the collaboration branch has been updated and reviewed, you will click on the **Publish** button on Data Factory 
Development and that will create the ARM templates for the development Data Factory Pipelines in the branch 
**adf_publish**

Updating the **adf_publish** branch will trigger a CI/CD pipeline that will take the Data Factory development instance 
ARM template files, ***updates*** them for the production instance and finally deploy them on the production 
Data Factory instance.</br>

~~~
The term "update" mentioned above implies updating the linked services between Data Factory
and Databricks as well as between Data Factory and Keyvault, resource groups and data factory names.
Since the development and production infrastructures are separated, the credentials and adresses
linking the services within the development infrastructure must be replaced by the ones for the
production infrastructure.
~~~

You can follow [this tutorial](https://www.youtube.com/watch?v=7fsIwWheyDk&ab_channel=KnowledgeBank) for further explanation.

Here some screenshots on how to setup the linked services towards Databricks inside Data Factory:

Create the global parameters for domain url and for the cluster id and make sure they will be exported to the ARM templates:

![ADF Global parameters](imgs/adf_global_parameters.png)

Parametrize the linked service in a way that it will use **local parameters** for the domain url as well as for the cluster id:

![ADF Linked Service towards Databricks](imgs/adf_linked_service_databricks.png)

In the pipelines created in Data Factory, setup the steps using Databricks notebooks to work with the global parameters. Inside the 'linked service properties' section
select a field and click 'add dynamic content'. In the menu you can now select the parameter for the specific field you selected:

![ADF Pipeline parametrization](imgs/adf_pipelines_parametrization.png)

By configuring it in this way, the CI/CD pipeline just needs to update the global parameters to be able to have a template that works on PRD and DEV infrastucture.

---
### Known problems and solutions/workarounds

#### How to use Terraform on WSL

To be able to run the ```terraform plan``` and ```terraform init``` commands on WSL, some additional steps have to be taken.

##### 1. Turn off generation of /etc/resolv.conf
Using your Linux prompt, open /etc/wsl.conf an paste the following content

```console
[network]
generateResolvConf = false
```

##### 2. Restart WSL
In Powershell run:

```console
wsl --shutdown
```

##### 3. Create a custom /etc/resolv.conf
Delete the /etc/resolv.conf:

```console
rm -f /etc/resolv.conf
```

Create a new resolv.conf with the following content

```console
nameserver 8.8.8.8
```

##### 4. Restart WSL
In Powershell run:

```console
wsl --shutdown
```

Open WSL and the issue should be resolved.

For more details check [this](https://github.com/microsoft/WSL/issues/8022) post.

#### Databricks error when running the 'terraform plan' command

It might be the case that the ``` terraform plan ``` command would not work because of Databricks, while ``` terraform apply ``` apply will actually do
A temporary fix of the error for the ``` terraform plan ``` command is to use the steps in [this](https://github.com/hashicorp/terraform/issues/26211#issuecomment-705084351) post.
This way, you can still validate whether your 'plan' is working, and thereafter replace the code to make sure you can properly 'apply' your plan.