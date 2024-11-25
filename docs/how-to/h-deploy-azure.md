# How to deploy on Azure

[Azure](https://azure.com/) is a cloud computing platform developed by Microsoft. It has management, access and development of applications and services to individuals, companies, and governments through its global infrastructure. Access the Azure web console at [portal.azure.com](https://portal.azure.com/).

## Summary
* [Set up Juju and Azure tooling](#set-up-juju-and-azure-tooling)
  * [Install Juju and Azure CLI](#install-juju-and-azure-cli)
  * [Authenticate](#authenticate)
  * [Bootstrap Juju controller on Azure](#bootstrap-juju-controller)
* [Deploy charms](#deploy-charms)
* [Expose database (optional)](#expose-database-optional)
* [Clean up](#clean-up)

---

## Set up Juju and Azure tooling
[note type="caution"]
**Warning**: The described `Azure interactive` method (with web browser authentication `service-principal-secret-via-browser`) described here is only supported starting Juju 3.6-rc1+!
[/note]
### Install Juju and Azure CLI
Install Juju via snap:
```shell
sudo snap install juju --channel 3.6/edge
```

Follow the installation guides for:
* [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt) - the Azure CLI for Linux

To check they are all correctly installed, you can run the commands demonstrated below with sample outputs:

```console
> juju version
3.6-rc1-genericlinux-amd64

> az --version
azure-cli                         2.65.0
core                              2.65.0
telemetry                          1.1.0

Dependencies:
msal                              1.31.0
azure-mgmt-resource               23.1.1
...

Your CLI is up-to-date.
```

### Authenticate

Please follow [the official Juju Azure documentation](https://juju.is/docs/juju/microsoft-azure) and check [the extra explanation about possible options](/t/15219). Choose the authentication method which fits you best. 

We are describing here the currently recommended `interactive` method with web browser authentication `service-principal-secret-via-browser`. This method does not require logging in with the Azure CLI locally, but it **requires an Azure subscription**.

The first mandatory step is to [create an Azure subscription](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/create-subscription) - you will need the Azure subscription ID for Juju. 

Once you have it, add Azure credentials to Juju:
```none
juju add-credential azure
```
This will start a script that will help you set up the credentials, where you will be asked to fill in a set of parameters:  
* `credential-name`: Fill this with a sensible name that will help you identify the credential set, say `<CREDENTIAL_NAME>`
* `region`: Select any default region that is more convenient for you to deploy your controller and applications. Note that credentials are not region-specific.
* `auth type`: select `interactive`, which is the recommended way to authenticate to Azure using Juju
* `subscription_id`: Use the value `<subscription_id>` from the Azure subscription created in the previous step.
* `application_name`: Generate a random string to avoid collision with other users or applications
* `role-definition-name`: Generate a random string to avoid collision with other users or applications, and store it as `<AZURE_ROLE>`

After prompting this information, you will be asked to authenticate the requests via web browser, as shown in the following example outputs:

```shell
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code <YOUR_CODE> to authenticate.
```

In the browser, open the [authentication page](https://microsoft.com/devicelogin) and enter the code `<YOUR_CODE>` provided in the output. 

You will be asked to authenticate twice, for allowing the creation of two different resources in Azure.

If successful, you will see a confirmation that the credentials have been correctly added locally:

```shell
Credential <CREDENTIAL_NAME> added locally for cloud "azure".
```

[details=Full sample output of `juju add-credential azure`]
```shell
> juju add-credential azure

This operation can be applied to both a copy on this client and to the one on a controller.
No current controller was detected and there are no registered controllers on this client: either bootstrap one or register one.
Enter credential name: azure-test-credentials1

Regions
  centralus
  eastus
  ...

Select region [any region, credential is not region specific]: eastus

Auth Types
  interactive
  service-principal-secret
  managed-identity

Select auth type [interactive]: interactive

Enter subscription-id: [USE-YOUR-REAL-AZURE-SUBSCRIPTION-ID]

Enter application-name (optional): azure-test-name1

Enter role-definition-name (optional): azure-test-role1

Note: your user account needs to have a role assignment to the
Azure Key Vault application (....).
You can do this from the Azure portal or using the az cli:
  az ad sp create --id ...

Initiating interactive authentication.

To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code HIDDEN to authenticate.
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code HIDDEN to authenticate.
Credential "azure-test-credentials1" added locally for cloud "azure".
```
[/details]

### Bootstrap Juju controller

Once successfully completed, bootstrap the new Juju controller on Azure:
```shell
> juju bootstrap azure azure

Creating Juju controller "azure" on azure/centralus
Looking for packaged Juju agent version 3.6-rc1 for amd64
No packaged binary found, preparing local Juju agent binary
Launching controller instance(s) on azure/centralus...
 - juju-aeb5ea-0 (arch=amd64 mem=3.5G cores=1)
Installing Juju agent on bootstrap instance
Waiting for address
Attempting to connect to 192.168.16.4:22
Attempting to connect to 172.170.35.99:22
Connected to 172.170.35.99
Running machine configuration script...
Bootstrap agent now started
Contacting Juju controller at 192.168.16.4 to verify accessibility...

Bootstrap complete, controller "azure" is now available
Controller machines are in the "controller" model

Now you can run
	juju add-model <model-name>
to create a new model to deploy workloads.
```

You can check the [Azure instances availability](https://portal.azure.com/#browse/Microsoft.Compute%2FVirtualMachines):

![image|689x313](upload://bB5lCMIHtL1KToftKQVv7z86aoi.png)


## Deploy charms

Create a new Juju model if you don't have one already
```shell
juju add-model welcome
```
> (Optional) Increase the debug level if you are troubleshooting charms:
> ```shell
> juju model-config logging-config='<root>=INFO;unit=DEBUG'
> ```

The following command deploys MySQL and [Data Integrator](https://charmhub.io/data-integrator), a charm that can be used to requests a test database:

```shell
juju deploy mysql
juju deploy data-integrator --config database-name=test123
juju integrate mysql data-integrator
```
Check the status:
```shell
> juju status
Model    Controller       Cloud/Region     Version    SLA          Timestamp
welcome  azure-centralus  azure/centralus  3.6-rc1.1  unsupported  16:05:59+02:00

App              Version          Status  Scale  Charm            Channel        Rev  Exposed  Message
data-integrator                   active      1  data-integrator  latest/stable   41  no
mysql            8.0.36-0ubun...  active      1  mysql            8.0/stable     240  no

Unit                Workload  Agent  Machine  Public address  Ports           Message
data-integrator/1*  active    idle   3        20.83.8.228
mysql/0*            active    idle   4        20.83.9.99      3306,33060/tcp  Primary

Machine  State    Address      Inst id        Base          AZ  Message
3        started  20.83.8.228  juju-07d319-3  ubuntu@22.04
4        started  20.83.9.99   juju-07d319-4  ubuntu@22.04  
```

Once deployed, request the credentials for your newly bootstrapped MySQL database:
```shell
juju run data-integrator/leader get-credentials
```

Example output:
```shell
...
mysql:
  data: '{"database": "test123", "external-node-connectivity": "true", "requested-secrets":
    "[\"username\", \"password\", \"tls\", \"tls-ca\", \"uris\"]"}'
  database: test123
  endpoints: 192.168.0.5:3306
  password: 1yGJ2KqtN4Qlf6KgeJDLOVES
  username: relation-14
  version: 8.0.36-0ubuntu0.22.04.1
```

At this point, you can access your DB inside Azure VM using the internal IP address. All further Juju applications will use the database through the internal network:
```shell
mysql -h 192.168.0.5 -u relation-14 -P1yGJ2KqtN4Qlf6KgeJDLOVES test123
```

From here you can begin to use your newly deployed MySQL. Learn more about operations like scaling, enabling TLS, managing users and passwords, and more in the [Charmed MySQL tutorial](/t/9922).

## Expose database (optional)

If it is necessary to access the database from outside of Azure, open the Azure firewall using the simple [juju expose](https://juju.is/docs/juju/juju-expose) functionality: 
```shell
juju expose mysql
```
> Be wary that [opening ports to the public is risky](https://www.beyondtrust.com/blog/entry/what-is-an-open-port-what-are-the-security-implications).

Once exposed, you can connect your database using the same credentials as above. This time use the Azure VM public IP assigned to the MySQL instance. You can see this with `juju status`:
```shell
> juju status mysql
Model    Controller       Cloud/Region     Version    SLA          Timestamp
welcome  azure-centralus  azure/centralus  3.6-rc1.1  unsupported  16:05:59+02:00

App              Version          Status  Scale  Charm            Channel        Rev  Exposed  Message
data-integrator                   active      1  data-integrator  latest/stable   41  no
mysql            8.0.36-0ubun...  active      1  mysql            8.0/stable     240  yes

Unit                Workload  Agent  Machine  Public address  Ports           Message
data-integrator/1*  active    idle   3        20.83.8.228
mysql/0*            active    idle   4        20.83.9.99      3306,33060/tcp  Primary

Machine  State    Address      Inst id        Base          AZ  Message
3        started  20.83.8.228  juju-07d319-3  ubuntu@22.04
4        started  20.83.9.99   juju-07d319-4  ubuntu@22.04  
```

Note the IP and port (`20.83.9.99:3306`) and connect via `mysql`:
```
mysql -h 20.83.9.99 -u relation-14 -P1yGJ2KqtN4Qlf6KgeJDLOVES test123
```
To close public access, run:
```shell
juju unexpose mysql
```

## Clean up

[note type="caution"]
Always clean Azure resources that are no longer necessary -  they could be costly!
[/note]

See all controllers in your machine with the following command:
```
> juju controllers
...
Controller  Model    User   Access     Cloud/Region     Models  Nodes    HA  Version
azure*      welcome  admin  superuser  azure/centralus       2      1  none  3.6-rc1.1  
```

To destroy the `azure` Juju controller and remove the Azure instance, run the command below. **All your data will be permanently removed.**
```shell
juju destroy-controller azure --destroy-all-models --destroy-storage --force
```

Next, check and manually delete all unnecessary Azure VM instances and resources. To show the list of all your Azure VMs, run the following command (make sure no running resources are left): 
```shell
az vm list
az resource list
```

List your Juju credentials:
```shell
> juju credentials

...
Client Credentials:
Cloud        Credentials
azure        azure-test-name1
...
```
Remove Azure CLI credentials from Juju:
```shell
juju remove-credential azure azure-test-name1
```

After deleting the credentials, the `interactive` process may still leave the role resource and its assignment hanging around. 
We recommend you to check if these are still present with:

```shell
az role definition list --name azure-test-role1
```
> Use it without specifying the `--name` argument to get the full list. 

You can also check whether you still have a role assignment bound to `azure-test-role1` registered using:

```shell
az role assignment list --role azure-test-role1
```

If this is the case, you can remove the role assignment first and then the role itself with the following commands:

```shell
az role assignment delete --role azure-test-role1
az role definition delete --name azure-test-role1
```

Finally, log out of the Azure CLI user credentials to prevent any credential leakage:
```shell
az logout 
```