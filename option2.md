## Moodle Manual Migration

  - This document explains how to migrate Moodle application from an on-premises environment to Azure.
- For each of the steps, you have two approaches provided
	- One that lets you to use Azure Portal
	- Other that lets you accomplish the same tasks on a command line using Azure CLI.

  
## Option 2: Moodle Migration without ARM Template Infrastructure

* Migration of Moodle without ARM template infrastructure is to create the infrastructure manually in Azure and migrate Moodle on it.
* Once the infrastructure is created, the Moodle software stack and associated dependencies are migrated.

  
## Prerequisites
- If the versions of the software stack deployed on-premises are lagging with respect to the versions supported in this guide, the expectation is that the on-premises versions will be updated/patched to the versions listed in this guide.
- Must have access to the on-premises infrastructure to take backup of Moodle deployment and configurations (including DB configurations).
- Azure subscription and Azure Blob storage should be created prior to migration.
- Make sure to have Azure CLI and Az Copy handy.
- Make sure Moodle website should be in maintenance mode.
- This migration guide supports the following software versions:
	- Ubuntu 16.04 LTS
	- Nginx 1.10.3
	- MySQL 5.6, 5.7 or 8.0 database server (This guide uses Azure Database for MYSQL)
	- PHP 7.2, 7.3, or 7.4
	- Moodle 3.8 & 3.9

  ## Migration Approach
-   Migration of Moodle application to Azure is broken down in the following three stages:
    - Pre-migration tasks.
    - Actual migration of the application.
    - Post-migration tasks.

## **Pre Migration:**
- Data Export from on-premises to Azure involves the following tasks.
	- Install Azure CLI.
	- Have an Azure subscription handy.
	- Create a Resource Group inside Azure.
	- Create a Storage Account inside Azure.
	- Backup all relevant data from on-premises infrastructure.
	- Ensure the on-premises database instance has MySQL-client installed.
	- Copy backup archive file (such as storage.tar.gz) to Blob storage on Azure.

<details> 
 <summary>(For detailed steps click on expand!)</summary>
  

-  **Data Export from on-premises to Azure Cloud:**

-  **Install Azure CLI**
	- Install Azure CLI on a host inside the on-premises infrastructure for all Azure related tasks.

		```
		curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
		```

	- Now login into your Azure account

		```
		az login
		```

	- az login: Azure CLI will quite likely launch an instance or a tab inside your default web-browser and prompt you to login to Azure using your Microsoft Account.

	- If the above browser launch does not happen, open a browser page at [https://aka.ms/devicelogin](https://aka.ms/devicelogin) and enter the authorization code displayed in your terminal.
	- To use command line use below command.

		```

		az login -u <username> -p <password>

		```

-  **Create Subscription:**
	- If you have a subscription handy skip this step.
	- And if you do not have a subscription, you can choose to [create one within the Azure Portal](https://ms.portal.azure.com/#blade/Microsoft_Azure_Billing/SubscriptionsBlade) or opt for a [Pay-As-You-Go](https://azure.microsoft.com/en-us/offers/ms-azr-0003p/)
	- To create the subscription using azure portal, navigate to Subscription from Home section.

		![image](/images/subscription1.png)

	 - Command to set the subscription.
            ```
            az account set --subscription "Subscription Name"

            # Example: az account set --subscription "FreeTrail"
            ```


-  **Create Resource Group:**
	- Once you have a subscription handy, you will need to create a Resource Group.
	- One option is to create resource group using Azure portal.
	- Navigate to home section and search for resource group, after clicking on add fill the mandatory fields and click on create.

		![image](/ss/Resourcegroup.PNG)

	- Alternatively, you can use the Azure CLI command to create a resource group.
	- Provide the same default Location provided in previous steps.
	- More details on [Location in Azure](https://azure.microsoft.com/en-in/global-infrastructure/data-residency/).

		```
		az group create -l location -n name -s Subscription_NAME_OR_ID
		# Update the screenshot and subscription name with sample test account
		# example: az group create -l eastus -n migration_option2 -s ComputePM LibrarySub - 067
		```

	- In above step resource group is created as "migration_option2". Use the same resource group in further steps.

-  **Create Storage Account:**
	- The next step would be to [create a Storage Account](https://ms.portal.azure.com/#create/Microsoft.StorageAccount) in the Resource Group you've just created.
	- Storage account can also be created using Azure portal or Azure CLI command.
	- To create using portal, navigate to portal and search for storage account and click on Add button.
	- After filling the mandatory details, click on create.

		![image](/ss/Storageaccount.PNG)

	- Alternatively, you can use Azure CLI command

		```
		az storage account create -n storageAccountName -g resourceGroupName --sku Standard_LRS --kind BlobStorage -l location
		example: az storage account create -n onpremstorage1 -g migration_option2 --sku Standard_LRS --kind BlobStorage -l eastus

		# In the above command --kind Indicates the type of storage account.
		```

	- Once the storage account "onpremstorage1" is created, this is used as the destination to take the on-premises backup.

-  **Backup of on-premises data:**
	 - Before taking backup of on-premises data, enable maintenance mode for moodle site.
        - Run the below command in on-premises virtual machine.
         ```
             sudo /usr/bin/php admin/cli/maintenance.php --enable
         ```
        - To check the status of the moodle site run the below command.
        ```
             sudo /usr/bin/php admin/cli/maintenance.php
         ```
	- Take backup of on-premises data such as moodle, moodledata, configurations and database backup file to a single directory. The following illustration should give you a good idea.

	  ![image](/images/folderstructure.png)

	 - First create an empty storage directory in any desired location to copy all the data.

  

		```
		sudo -s
		for example, the location is /home/azureadmin
		cd /home/azureadmin
		mkdir storage
		```
  

-  **Backup of moodle and moodledata**
	- The moodle directory consists of site HTML content and moodledata contains moodle site data.

	```
		#commands to copy moodle and moodledata
		cp -R /var/www/html/moodle /home/azureadmin/storage/
		cp -R /var/moodledata /home/azureadmin/storage/
	```

-  **Backup of PHP and webserver configuration**
	- Copy the PHP configuration files such as php-fpm.conf, php.ini, pool.d and conf.d directory to phpconfig directory under the configuration directory.
	- Copy the ngnix configuration such as nginx.conf, sites-enabled/dns.conf to the nginxconfig directory under the configuration directory.

	```
	cd /home/azureadmin/storage
	mkdir configuration
	# command to copy nginx and php configuration
	cp -R /etc/nginx /home/azureadmin/storage/configuration/nginx
	cp -R /etc/php /home/azureadmin/storage/configuration/php
	```
	- If the webserver used is Apache instead, copy all the relevant configuration for Apache to the configuration directory.

  
-  **Create a backup of database**
	- If you already have MySQL-client installed ,skip the step to install mysql-client.
	- If you do not have MySQL-client installed on the database instance, now would be a good time to do that.

	```

	sudo -s

	# command to check MySQL-client is installed or not

	mysql -V

	# if the MySQL-client is not installed, install the same by following command.

	sudo apt-get install MySQL-client

	#following command will allow to you to take the backup of database.

	mysqldump -h dbServerName -u dbUserId -pdbPassword dbName > /home/azureadmin/storage/database.sql

	# Replace dbServerName, dbUserId, dbPassword and bdName with on-premises database details

	```
	- Create an archive tar.gz file of backup directory.

	```

	cd /home/azureadmin/
	tar -zcvf storage.tar.gz storage

	```

  -  **Download and install Az Copy**
		- Execute the below commands to install Az Copy
			
			```

			sudo -s

			wget https://aka.ms/downloadazcopy-v10-linux

			tar -xvf downloadazcopy-v10-linux

			sudo rm /usr/bin/azcopy

			sudo cp ./azcopy_linux_amd64_*/azcopy /usr/bin/

			```

  
-  **Copy Archive file to Blob storage**

	- Copy the on-premises archive file to blob storage using Az Copy.

	- To use Az Copy, user should generate SAS Token first.

	- Go to the created Storage Account Resource and navigate to Shared access signature in the left panel.

	  ![image](ss/Storageaccountcreated.PNG)

	- Select the Container, object checkboxes and set the start, expiry date of the SAS token. Click on "Generate SAS and Connection String".

  
	    ![image](ss/SAStoken.PNG)

	- Copy and save the SAS token for further use.
	
	- Command to create a container in the storage account.

		```
		az storage container create --account-name <storageAccontName> --name <containerName> --auth-mode login
		Example: az storage container create --account-name onpremstorage1 --name migration --auth-mode login
		# --auth-mode login means authentication mode as login, after login the container will be created.
		```
        

    - Container can be created using Azure Portal, Navigate to the same storage account created and click on container and click on Add button.
	- After giving the mandatory container name, click on create button.
	    ![image](ss/Containercreation.PNG)



	- Command to copy archive file to blob storage.
        ```
        sudo azcopy copy '/home/azureadmin/storage.tar.gz' 'https://<storageAccountName>.blob.core.windows.net/<containerName>/<SAStoken>
        Example: azcopy copy '/home/azureadmin/storage.tar.gz' 'https://onpremstorage1.blob.core.windows.net/migration/?sv=2019-12-12&ss='
        ```
		
	- Now, you should have a copy of your archive inside the Azure blob storage account.
	 ![image](ss/ArchivefileinBlobstorage.PNG)

</details>
  
## **Actual Migration:**

- Actual Migration involves the following tasks.
	- Migration of Moodle.
	- Install prerequisites for Moodle.
	- Create Moodle Shared directory.
	- Download On-premises archive file.
	- Download and run the migrate_moodle.sh script.
	- Configuring permissions.
	- Importing Database.
	- Configuring PHP & Webserver.
	- Configuring VMSS.
	- Set a cron job.

<details> 
<summary>(For detailed steps click on expand!)</summary>

-  **Resources Creation**
	- To install the infrastructure for Moodle, navigate to the [azure portal](portal.azure.com) and select the created Resource Group.
	- Create the infrastructure by adding the resources.

  

-  **Creating Resources to host the Moodle application**

-  **Network Resources:**
- **Virtual Network**
	- An Azure Virtual Network is a representation of your own network in the cloud. It is a logical isolation of the Azure cloud dedicated to your subscription. When you create a VNet, your services and VMs within your VNet can communicate directly and securely with each other in the cloud. More information on [Virtual Network](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview).
	- you can create using Azure CLI command.

  

		```
		az network vnet create --name myVirtualNetwork --resource-group myResourceGroup --subnet-name default
		ex: az network vnet create --name migrationvnet --resource-group migration_option2 --subnet-name 
		```

	- Alternatively virtual network can be created using Azure Portal.
	- Navigate to the above created resource group, select Add to a resource. From the Azure Marketplace, select Networking > Virtual network.
	- In Create virtual network, for Basics section provide this information:

  
		![image](ss/vnetcreate.png)

	 - Subscription: Select the same subscription created or used in above steps.
	- Resource Group: Select same resource group as migration_option2
	- Name: Give the instance name ex:migrationvnet.
	- Region: Select default region as eastus
	- Select Next: IP Addresses, and for IPv4 address space, enter 10.1.0.0/16.
	- Select Add subnet, then select the default address name and select Subnet name 
	- Then create a subnet in the Virtual Network using Azure CLI command

  

		```
		az network vnet subnet create -g MyResourceGroup --vnet-name MyVnet -n MySubnet --address-prefixes 10.0.0.0/24 --network-security-group MyNsg --route-table MyRouteTable

		ex: az network vnet subnet create -g migration_option2 --vnet-name migrationvnet -n MySubnet --address-prefixes 10.0.0.0/24 --network-security-group MyNsg --route-table MyRouteTable
		```

  
	- Select Add, then select Review + create. Leave the rest parameters as default and select Create.
	- For more Details on [virtual network](https://docs.microsoft.com/en-us/azure/virtual-network/quick-create-portal)

  

 -  **Network Security Group:**
	- A network security group (NSG) is a networking filter (firewall) containing a list of security rules allowing or denying network traffic to resources connected to Azure VNets. For more information [Network security group](https://docs.microsoft.com/en-us/azure/virtual-network/security-overview).
	- One option is to create network security group using Azure Portal.
	- Navigate to same resource group, Click on Add resource and select network security group.
	- Give the instance details such as name and region to create network security group.
	
	![image](ss/networksecuritygroup.png)
	- You can create a network security group using Azure CLI

	```
		az network nsg create --resource-group myResourceGroup --name myNSG

		ex: az network nsg create --resource-group migration_option2 --name migration_nsg
	```

-  **Network Interface:**
	- A network interface enables an Azure Virtual Machine to communicate with internet, Azure, and on-premises resources.
	- One option is to create network interface using Azure Portal.
	- Navigate to same resource group, Click on Add resource and select network interface.
	- Give the instance details such as name and region and fill the other mandatory details.
	
	![image](ss/NI.png)
	- Create Network Interface with Azure CLI command

  

		```
		az network nic create --resource-group myResourceGroupLB --name myNicVM1 --vnet-name myVNet --subnet myBackEndSubnet --network-security-group myNSG

		ex: az network nic create --resource-group migration_option2 --name migration_ni --vnet-name migrationvnet --subnet MySubnet --network-security-group migration_nsg

		```

  
-  **Load Balancer:**
	- An Azure load balancer is a Layer-4 (TCP, UDP) load balancer that provides high availability by distributing incoming traffic among healthy VMs. A load balancer health probe monitors a given port on each VM and only distributes traffic to an operational VM. For more details on [Load balancer](https://docs.microsoft.com/en-us/azure/load-balancer/tutorial-load-balancer-standard-internal-portal)
	- One option is to create Load Balancer using Azure Portal.
	- Navigate to same resource group, Click on Add resource and select Load balancer.
	- Give the instance details such as name and region.
	- Give the type as public and sku as standard
	- For the public Ip address create a new IP address and give the name.
	- After filling the details. Click on review and create.
	
		![image](ss/Loadbalancer.png)
        - Alternatively, Load balancer can be created using Azure CLI commands.  
  

		```
		#Create a public IP
		az network public-ip create --resource-group myResourceGroupLB --name myPublicIP --sku Standard

		ex: az network public-ip create --resource-group migration_option2 --name migration_lb_ip --sku Standard

		#Create Load balancer
		az network lb create --resource-group myResourceGroupLB --name myLoadBalancer --sku Standard --public-ip-address myPublicIP --frontend-ip-name myFrontEnd --backend-pool-name myBackEndPool

		ex: az network lb create --resource-group migration_option2 --name migration_lb --sku Standard --public-ip-address migration_lb_ip --frontend-ip-name myFrontEnd --backend-pool-name myBackEndPool
		```
    	- Configure DNS name for load balancer.
			- Navigate to the load balancer IP created in Azure Portal.
			- In the left panel, look for configuration and click on it.
			- Add the DNS name and click on save.

	 	![image](ss/DNSConfiguration.PNG)

		 
	-  **Load Balancing Rules**
		- Go to the Load Balancer Resource in Azure portal.
			- In the left panel, find Health Probes in setting.
				- Create by clicking add button.
				- Enter Name, Port as 80 for http and 443 for https, Interval as 5 and Unhealthy threshold as 3.
				- Click on ok to save.
			- In the left panel, find Load Balancing Rules.
				- Add http and https rules by clicking add button.
				- Fill the Name, select the front end IP (load balancer IP), Port, Backend Port, select the backend pool and health probe.
				- Cick on ok button to save the rule.


		- Alternatively, Set the http (TCP/80) and https (TCP/443) rules from Azure CLI.
			```
			az network lb rule create --resource-group myResourceGroupLB --lb-name myLoadBalancer --name myHTTPRule --protocol tcp --frontend-port 80 --backend-port 80 --frontend-ip-name myFrontEnd --backend-pool-name myBackEndPool --probe-name myHealthProbe --disable-outbound-snat true
			```

  -  **Azure Application GateWay**
		- An Azure Application Gateway is a web traffic load balancer that enables you to manage traffic to your web applications. Traditional load balancers operate at the transport layer (OSI layer 4 - TCP and UDP) and route traffic based on source IP address and port, to a destination IP address and port. For more details on [Azure application gateway](https://docs.microsoft.com/en-us/azure/application-gateway/overview).
		- To deploy the Application gate way from [Azure Portal](https://docs.microsoft.com/en-us/azure/application-gateway/quick-create-portal).

	  	- To deploy the Application gate way from [Azure CLI](https://docs.microsoft.com/en-us/azure/application-gateway/quick-create-cli)

		  *Note:* Azure Application Gateway is optional, this migration document supports only Azure Load Balancer.

  

-  **Storage Resources**
	* An Azure storage account contains all of your Azure Storage data objects: blobs, files, queues, tables, and disks. The storage account provides a unique namespace for your Azure Storage data that is accessible from anywhere in the world over HTTP or HTTPS
	* Storage account will have specific type, replication, Performance, Size. For more details on [Storage account](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-overview).
	- Creating storage account with Azure Files Premium below should be the mandatory parameters.
	- Replication is Premium Locally-redundant storage (LRS)
	- Type is File Storage
	
		![image](ss/Storageaccountwithpremium.PNG)
	- Azure CLI command to create storage account

  

		```
		az storage account create -n storageAccountName -g resourceGroupName --sku Standard_LRS --kind StorageV2 -l eastus2euap -t Account
		```


  -  **Database Resources** -

	  - Creates an [Azure Database for MySQL server](https://docs.microsoft.com/en-in/azure/mysql/).
	  -  Azure Database for MySQL is easy to set up, manage and scale. It automates the management and maintenance of your infrastructure and database server, including routine updates,backups and security. Build with the latest community edition of MySQL, including versions 5.6, 5.7 and 8.0.
	  - One option is to create Database using Azure Portal.
	  - Navigate to same resource group, Click on Add resource and select Azure Database for MySQL server .
	  - Give the instance details such as name and region and fill the other mandatory details such as servername,login name and password.
	  ![image](ss/database.png)
	  
	  - Alternatively, database can be created using Azure CLI commands
  

		```
		az mysql server create --resource-group myresourcegroup --name mydemoserver --location westus --admin-user myadmin --admin-password <server_admin_password> --sku-name GP_Gen5_2

		ex: az mysql server create --resource-group migration_option2 --name mysql-migration --location eastu --admin-user myadmin --admin-password <server_admin_password> --sku-name GP_Gen5_2
		```
-  **Configure firewall:**
	- Azure Databases for MySQL are protected by a firewall. By default, all connections to the server and the databases inside the server are rejected. Before connecting to Azure Database for MySQL for the first time, configure the firewall to add the client machine's public network IP address (or IP address range).

  

		```
		az mysql server firewall-rule create --resource-group myresourcegroup --server mydemoserver --name AllowMyIP --start-ip-address 192.168.0.1 --end-ip-address 192.168.0.1
		```
	- Click your newly created MySQL server, and then click Connection security.

	  ![connectionSecurity SS](images/connection_security.png)

	 - You can Add My IP, or configure firewall rules here. Click on save after you have created the rules.

  - You can now connect to the server using mysql command-line tool or MySQL Workbench GUI tool.

  

-  **Get connection information:**
	- From the MySQL server resource page, note down Server Name and Server admin login name. You may click the copy button next to each field to copy to the clipboard.

	  ![Connection Info ss](images/mysql-workbench.png)

	 - For example, the server name is mydemoserver.mysql.database.azure.com, and the server admin login is myadmin@mydemoserver.

  

-  **Virtual Machine**
	- A virtual machine is a computer file, typically called an image, which behaves like an actual compute [Virtual machine](https://azure.microsoft.com/en-in/overview/what-is-a-virtual-machine/).
	- Before creating Virtual machine create an SSH key pair.
	- To generate [SSH keys](https://docs.microsoft.com/e1n-us/azure/virtual-machines/linux/create-ssh-keys-detailed).

	- If you already have an SSH key pair, you can skip this step.

	- Go to the PuTTY installation directory (the default location is C:\Program Files\PuTTY) and run: puttygen.exe

	- In the PuTTY Key Generator window, set Type of key to generate to RSA, and set Number of bits in a generated key to 2048.

	  ![putty keygen SS](images/puttykeygen.png)

  
	
	- Select Generate.

	- To generate a key, in the Key box, move the pointer randomly.

	- When the key generation has finished, select Save public key, and then select Save private key to save your keys to files.

  
		![putty keygen ss 1](images/puttykeygen2.png)

	- The public and private key is generated

-  **Creating Virtual Machine**
	- Create a VM with ubuntu 16.04 operating system with SSH public key
	- Select the default subscription and same resource group and give name for virtual machine.
	- Keep the availability options as default.
	- Select the Image of the virtual machine and Select the disk size.
	- Select Authentication type SSH, give the username give the SSH key generated in previous step.
	- Select the inbound rule for SSH as 22 and HTTP as 80.
	- Click next on Disk section.

		 ![image](ss/vm.png)
	- Select the OS disk type. There are 3 choices Standard SSD, Premium SSD, Standard HDD
	- Keep the other parameters as default.

	    ![image](ss/vm01.PNG)

	- Click next on networking and select the virtual network created in above step and the public IP and keep the above parameters as default.

		![image](ss/vm02.PNG)
	- Click on next for management and keep the parameters as default.
	- Keeping the other parameters as default Click on review and create.
	- Alternatively Virtual Machine can be created using AZ CLI command

  

		```
		az vm create --resource-group myResourceGroup --name myVM --image UbuntuLTS --admin-username azureuser --authentication-type ssh --generate-ssh-keys

		ex: az vm create --resource-group migration_option2 --name controller-vm --image UbuntuLTS --admin-username azureadmin --authentication-type ssh --generate-ssh-keys
		```

	 - Login into this controller machine using any of the free open-source terminal emulator or serial console tools.
	- Copy the public IP of controller VM and paste as host name and expand SSH in navigation panel and click on Auth and browse the same SSH key file given while deployment. Click on Open and it will prompt to give the username as azureadmin same as given while deployment that is azureadmin
	 - [Putty general FAQ/troubleshooting questions](https://documentation.help/PuTTY/faq.html).


  
		![putty ss1](images/puttyloginpage.PNG)
	 	![putty ss1](images/puttykeybrowse.PNG)

  

  
  

# Migrate Moodle

 -  **Install prerequisites for Moodle**.
	- To install prerequisites for Moodle run the following commands
	
	- If you are installing PHP greater than or equal to 7.2 then upgrade ppa package
		```
		sudo -s
		sudo add-apt-repository ppa:ondrej/php -y > /dev/null 2>&1
		sudo apt-get update > /dev/null 2>&1
		```
	-	Update and install rsyslog, unzip, mysql-client, php and php extensions.
		```
		sudo apt-get -y update
		sudo apt-get -y install unattended-upgrades
		sudo apt-get -y install python-software-properties unzip rsyslog
		sudo apt-get -y install mysql-client git
		sudo apt-get -y install php$phpVersion
		```

	 - Install Php extensions and varnish.
	 	```
		# set php version to a variable 
		phpVersion=`/usr/bin/php -r "echo PHP_VERSION;" | /usr/bin/cut -c 1,2,3`
		echo $phpVersion

		sudo apt-get -y install varnish php$phpVersion php$phpVersion-cli php$phpVersion-curl php$phpVersion-zip php-pear php$phpVersion-mbstring php$phpVersion-dev mcrypt
		sudo apt-get -y --force-yes install php$phpVersion-fpm
		sudo apt-get install -y --force-yes graphviz aspell php$phpVersion-common php$phpVersion-soap php$phpVersion-json php$phpVersion-redis
		sudo apt-get install -y --force-yes php$phpVersion-bcmath php$phpVersion-gd php$phpVersion-xmlrpc php$phpVersion-intl php$phpVersion-xml php$phpVersion-bz2 
		```

	- Install Missing PHP extensions.
		- ARM template install the following PHP extensions 
			- fpm, cli, curl, zip, pear, mbstring, dev, mcrypt, soap, json, redis, bcmath, gd, mysql, xmlrpc, intl, xml and bz2.
		- To know the PHP extensions which are installed on on-premises run the below command on on-premises virtual machine to get the list.
			```
			php -m
			```
		- Note: If on-premises has any additional PHP extensions which are not present in Controller Virtual Machine can be installed manually.
			```
			sudo apt-get install -y php-extensionName
			```
	- PHP with 7.2 and higher versions are installing apache2 by default.
		- This documentation will support only nginx and if apache is installed then mask the apache service.
		- Check the apache service is installed by below command.
			```
			apache2 -v
			```
		- If the apache service is installed it will show up the service version, so you can identify that apache service is installed.
		- Run the below commands to mask the apache2 service.
			```
			sudo systemctl stop apache2
			sudo systemctl mask apache2
			```	
	- Install nginx webserver
		```
		sudo apt-get -y --force-yes install nginx
		```
	-   Install Azure CLI on a host inside the on-premises infrastructure for all Azure related tasks.
		```
		curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
		```

  -  **Create Moodle Shared directory**
		- Create a moodle shared directory to install Moodle (/moodle)
			```
			mkdir -p /moodle
			mkdir -p /moodle/moodledata
			mkdir -p /moodle/html
			mkdir -p /moodle/certs
			```
		-   Install Azure CLI on a host inside the on-premises infrastructure for all Azure related tasks.
			```
			curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
			```
		- Mount shared [moodle directory with storage account](https://github.com/asift91/Manual_Migration/blob/master/azurefiles.md) for more information.

  -  **Download On-Premises archive file**
		- Download the On-Premises archived data from Azure Blob storage to VM such as Moodle, Moodledata, configuration directory with database backup file to /home/azureadmin location

 		- Execute the below commands to install AzCopy
			```
			sudo -s
			wget https://aka.ms/downloadazcopy-v10-linux
			tar -xvf downloadazcopy-v10-linux
			sudo rm /usr/bin/azcopy
			sudo cp ./azcopy_linux_amd64_*/azcopy /usr/bin/
			```
		-   Download the compressed backup file(storage.tar.gz) from blob storage to Controller virtual Machine at /home/azureadmin/ location.
			```
			sudo -s
			cd /home/azureadmin/
			azcopy copy 'https://storageaccount.blob.core.windows.net/container/BlobDirectoryName<SASToken>' '/home/azureadmin/'

			Example: azcopy copy 'https://onpremisesstorage.blob.core.windows.net/migration/storage.tar.gz?sv=2019-12-12&ss=' '/home/azureadmin/storage.tar.gz'
			```
		- Extract the compressed content to a directory.
			```
			cd /home/azureadmin
			tar -zxvf storage.tar.gz
			```

		- Storage directory contains Moodle, Moodledata and configuration directory along with database backup file.

  
 -  **Migrate On-Premises Moodle:**


	- Copy and replace moodle directory with On-Premises moodle directory
		```
		cd /home/azureadmin/
		cp -rf storage/moodle /moodle/html/moodle
		```

	- Copy and replace moodledata directory with On-Premises moodle directory
		```
		cd /home/azureadmin/
		cp -rf storage/moodledata /moodle/moodledata
		```

-  **Configuring permissions**
	- Set the Moodle and Moodledata directory permissions.
	- Set 755 and www-data owner:group permissions to Moodle directory
		```
		sudo chmod 755 /moodle
		sudo chown -R www-data:www-data /moodle
		```

	 - Set 770 and www-data owner:group permissions to Moodledata directory
		```
		sudo chmod 755 /moodle/moodledata
		sudo chown -R www-data:www-data /moodle/moodledata
		```

-  **Importing Database**
	
  - Importing the moodle Database to Azure moodle DB.
	- Before importing database, make sure that Azure Database for MySQL server details are handy.
		- Navigate to Azure Portal and go to the created Resource Group.
		- Select the Azure Database for MySQL server resource.
		- In the overview panel find Azure Database for MySQL server details such as Server name, Server admin login name.
		- Reset the password by clicking the Reset Password button at top let of the page.
		- Use above gathered database server details in the below commands.

	- Import the on-premises database to Azure Database for MySQL.
	- Create a database to import on-premises database.
		```    
		mysql -h $server_name -u $server_admin_login_name -p$admin_password -e "CREATE DATABASE $moodledbname CHARACTER SET utf8;"
		```
	- Assign right permissions to database.
		```
		mysql -h $server_name -u $server_admin_login_name -p$admin_password -e "GRANT ALL ON $moodledbname.* TO '$server_admin_login_name' IDENTIFIED BY '$admin_password';"
		``` 
	- Import the database.
		```
		mysql -h $server_name -u $server_admin_login_name -p$admin_password $moodledbname < /home/azureadmin/storage/database.sql
		```
	
	- [Database general FAQ/troubleshooting questions](https://docs.azure.cn/en-us/mysql-database-on-azure/mysql-database-tech-faq)

- Update the database details in moodle configuration file (/moodle/config.php).
	- Update the following parameters in config.php.
	- Prior to this make sure that DNS name is handy.
		- Navigate to Azure Portal and go to the created Resource group.
		- Find the Load Balancer public IP and get the DNS name from overview panel. 
	- dbhost, dbname, dbuser, dbpass, dataroot and wwwroot
		```
		cd /moodle/html/moodle/
		nano config.php
		# Update the database details and save the file.
		#
		# Example:
		# $CFG->dbhost    = 'localhost';                - change the localhost with servername.
		# $CFG->dbname    = 'moodle';                   - change moodle to newly created database name.
		# $CFG->dbuser    = 'root';                     - change root with Server admin login name.
		# $CFG->dbpass    = 'password';                 - change password with Server admin login password.
		# $CFG->wwwroot   = 'http://on-premises.com';   - change on-premises with DNS name.
		# $CFG->dataroot  = '/var/moodledata';          - change the path to '/moodle/moodledata'
			# On-premises dataroot directory can be at any location.
		# 
		# After the changes, Save the file. 
		# Press CTRL+o to save and CTRL+x to exit.
		```
-	**Configuring Php & WebServer**
	- Update the nginx conf file
		```
		sudo mv /etc/nginx/sites-enabled/* /home/azureadmin/backup/
		cd /home/azureadmin/storage/configuration/nginx/sites-enabled/
		sudo cp *.conf /etc/nginx/sites-enabled/

		```
	 - Update the php config file
		```
		sudo mv /etc/php/<phpVersion>/fpm/pool.d/www.conf /home/azureadmin/backup
		cd /home/azureadmin/storage
		sudo cp configuration/php/<phpVersion>/fpm/pool.d/www.conf /etc/php/<phpVersion>/fpm/pool.d/
		```
	  - Restart the web servers

		```
		sudo systemctl restart nginx
		sudo systemctl restart php(phpVersion)-fpm
		ex: sudo systemctl restart php7.4-fpm
		```
-	**Copy of Configuration files:**
     -   Copying php and webserver configuration files to shared location.
        -   Configuration files can be copied to VMSS instance(s) from the shared location easily.
        -   Creating directory for configuration in shared location.
            ```
            mkdir -p /moodle/config
            mkdir -p /moodle/config/php
            mkdir -p /moodle/config/nginx
            ```
    -   Copying the php and webserver config files to configuration directory.
          
		 ```
            cp /etc/nginx/sites-enabled/* /moodle/config/nginx
            cp /etc/php/$_PHPVER/fpm/pool.d/www.conf /moodle/config/php
        ```
-	**Setup Cron Job:**
- To update local instance in VMSS instance(s), need to have a script file which should be executed from the Controller Vitual Machine.
- Execute the following commands.
- Create a file setup_cron_on_vm.sh
```
cd /home/azureadmin/
nano setup_cron_on_vm.sh
# Above command will create a new file.
```
- Copy the below content to the setup_cron_on_vm.sh
```
SERVER_TIMESTAMP_FULLPATH="/moodle/html/moodle/.last_modified_time.moodle_on_azure"
LOCAL_TIMESTAMP_FULLPATH="/var/www/html/moodle/.last_modified_time.moodle_on_azure"

LAST_MODIFIED_TIME_UPDATE_SCRIPT_FULLPATH="/usr/local/bin/update_last_modified_time.moodle_on_azure.sh"
function create_last_modified_time_update_script {

    mkdir -p $(dirname $LAST_MODIFIED_TIME_UPDATE_SCRIPT_FULLPATH)
    cat <<EOF > $LAST_MODIFIED_TIME_UPDATE_SCRIPT_FULLPATH
#!/bin/bash
echo \$(date +%Y%m%d%H%M%S) > $SERVER_TIMESTAMP_FULLPATH
EOF
    cd $(dirname $LAST_MODIFIED_TIME_UPDATE_SCRIPT_FULLPATH)
    chmod +x $LAST_MODIFIED_TIME_UPDATE_SCRIPT_FULLPATH

}


function run_once_last_modified_time_update_script {
  $LAST_MODIFIED_TIME_UPDATE_SCRIPT_FULLPATH
}

create_last_modified_time_update_script
run_once_last_modified_time_update_script

```
- Save the file with the below commands.
```
Press Ctrl + O and enter to save the file.
Press Ctrl + X to come out the file.
```

- Execute the below command to set cron job.
```
cd /home/azureadmin/
bash setup_cron_on_vm.sh
```
  
-  **Scale Set:**
	- A virtual machine scale set allows you to deploy and manage a set of auto-scaling virtual machines. You can scale the number of VMs in the scale set manually or define rules to auto scale based on resource usage like CPU, memory demand, or network traffic. An Azure load balancer then distributes traffic to the VM instances in the scale set. For more information on [Scaleset](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/quick-create-portal).
	- Create a scale set in same resource group.
	- Prerequisites is the to create a public Standard Load Balancer.
	- The name and public IP address created are automatically configured as the load balancer's front end.
	- Scale set creates an VM instance with Ubuntu 16.04 OS.
	- Search Virtual machine scale sets. Select Create on the Virtual machine scale sets page, which will open the Create a virtual machine scale set page.
	- In the Basics tab, under Project details, select the subscription and then choose to same resource.
	- Type name as the name for your scale set.
	- In Region, select the same region.
	- Leave the default value of Scale Set VMs for Orchestration mode.
	- Enter your desired username and select which authentication type as SSH and give the same SSH key and username as azureadmin.
	- Select the image or browse the image for the scale set

		![image](ss/ss1.png)
	- Select the size for the disk.
	- Click Next for the disk tab select the OS disk type as per choice
				
		 ![image](ss/ss2.png)

	- Click Next for the networking section
	- Select the created virtual network.

		![image](ss/ss3.png)

	- Give the instance count and the scaling policy as manual or custom.
	- set the rules to scale up/down a VM based on the average cpu percentage.

		 ![image](ss/ss4.png)

	- Select Next and keep the other things as default.

		![image](ss/ss5.png)

	- Click on review and create and the scale set.
	- Scale set can be created from Azure CLI
		```
		az vmss create -n MyVmss -g MyResourceGroup --public-ip-address-dns-name my-globally-dns-name --load-balancer MyLoadBalancer --vnet-name MyVnet --subnet MySubnet --image UbuntuLTS --generate-ssh-keys
		```
	- VMSS will create a VM instance with an internal IP. User need to have a VPN gateway to access the VM.
	- To setup the Virtual Network Gateway access the [document](https://github.com/asift91/Manual_Migration/blob/master/vpngateway.md).

  
-  **Configuring VMSS**

-  **Install prerequisites for Moodle**.
	- To install prerequisites for Moodle run the following commands
	
	- If you are installing PHP greater than or equal to 7.2 then upgrade ppa package
		```
		sudo -s
		sudo add-apt-repository ppa:ondrej/php -y > /dev/null 2>&1
		sudo apt-get update > /dev/null 2>&1
		```
	-	Update and install rsyslog, unzip, mysql-client, php and php extensions.
		```
		sudo apt-get -y update
		sudo apt-get -y install unattended-upgrades
		sudo apt-get -y install python-software-properties unzip rsyslog
		sudo apt-get -y install mysql-client git
		sudo apt-get -y install php$phpVersion
		```

	 - Install Php extensions and varnish.
	 	```
		# set php version to a variable 
		phpVersion=`/usr/bin/php -r "echo PHP_VERSION;" | /usr/bin/cut -c 1,2,3`
		echo $phpVersion

		sudo apt-get -y install varnish php$phpVersion php$phpVersion-cli php$phpVersion-curl php$phpVersion-zip php-pear php$phpVersion-mbstring php$phpVersion-dev mcrypt
		sudo apt-get -y --force-yes install php$phpVersion-fpm
		sudo apt-get install -y --force-yes graphviz aspell php$phpVersion-common php$phpVersion-soap php$phpVersion-json php$phpVersion-redis
		sudo apt-get install -y --force-yes php$phpVersion-bcmath php$phpVersion-gd php$phpVersion-xmlrpc php$phpVersion-intl php$phpVersion-xml php$phpVersion-bz2 
		```

	- Install Missing PHP extensions.
		- ARM template install the following PHP extensions 
			- fpm, cli, curl, zip, pear, mbstring, dev, mcrypt, soap, json, redis, bcmath, gd, mysql, xmlrpc, intl, xml and bz2.
		- To know the PHP extensions which are installed on on-premises run the below command on on-premises virtual machine to get the list.
			```
			php -m
			```
		- Note: If on-premises has any additional PHP extensions which are not present in Controller Virtual Machine can be installed manually.
			```
			sudo apt-get install -y php-extensionName
			```
	- PHP with 7.2 and higher versions are installing apache2 by default.
		- This documentation will support only nginx and if apache is installed then mask the apache service.
		- Check the apache service is installed by below command.
			```
			apache2 -v
			```
		- If the apache service is installed it will show up the service version, so you can identify that apache service is installed.
		- Run the below commands to mask the apache2 service.
			```
			sudo systemctl stop apache2
			sudo systemctl mask apache2
			```	
	- Install nginx webserver
		```
		sudo apt-get -y --force-yes install nginx
		```
-  **Create Moodle Shared directory**
	- Create a moodle shared directory (/moodle)
		```
		mkdir -p /moodle
		mkdir -p /moodle/moodledata
		mkdir -p /moodle/html
		mkdir -p /moodle/certs
		```
-  **Mounting File Share**
	- Mount Azure File share in VM instance
	- Follow the [documentation](https://github.com/asift91/Manual_Migration/blob/master/azurefiles.md) to set the Azure File Share on VMSS


-  **Copy Php & WebServer Configuration files:**
	- Update nginx configuration file (/moodle/config.php)

		```
		mkdir -p /home/azureadmin/backup/
		sudo mv /etc/nginx/sites-enabled/* /home/azureadmin/backup/
		cd /moodle/config/nginx/
		sudo cp *.conf /etc/nginx/sites-enabled/
		```
	- Update the php config file.
		```
		sudo mv /etc/php/<phpVersion>/fpm/pool.d/www.conf /home/azureadmin/backup
		cd /moodle/config/php/
		sudo cp www.conf /etc/php/<phpVersion>/fpm/pool.d/
		```
	- Restart the servers. 
		```
		sudo systemctl restart nginx
		sudo systemctl restart php(phpVersion)-fpm
		ex: sudo systemctl restart php7.2-fpm

		```

-  **Set a cron job**

- A cron job will be set to update a local copy of /moodle/html/ to webroot directory (/var/www/html/) by updating a time stamp.

- Cron job will run for every minute, It will check for time stamp update and local copy of VMSS get updated.

- Create a file to set up a cron job by following command.
	```
	cd /home/azureadmin/
	sudo nano setup_cron.sh
	# Above command will open a new file.
	```
- Copy the below code into the created setup_cron.sh file.
```

# setup_html_local_copy_cron_job

LOCAL_TIMESTAMP_FULLPATH="/var/www/html/moodle/.last_modified_time.moodle_on_azure"
SERVER_TIMESTAMP_FULLPATH="/moodle/html/moodle/.last_modified_time.moodle_on_azure"

function setup_html_local_copy_cron_job {
if [ "$(whoami)" != "root" ]; then
echo "${0}: Must be run as root!"
return 1
fi

local SYNC_SCRIPT_FULLPATH="/usr/local/bin/sync_moodle_html_local_copy_if_modified.sh"
mkdir -p $(dirname ${SYNC_SCRIPT_FULLPATH})

local SYNC_LOG_FULLPATH="/var/log/moodle-html-sync.log"

cat <<EOF > ${SYNC_SCRIPT_FULLPATH}
#!/bin/bash
sleep \$((\$RANDOM%30))
if [ -f "$SERVER_TIMESTAMP_FULLPATH" ]; then
  SERVER_TIMESTAMP=\$(cat $SERVER_TIMESTAMP_FULLPATH)
  if [ -f "$LOCAL_TIMESTAMP_FULLPATH" ]; then
    LOCAL_TIMESTAMP=\$(cat $LOCAL_TIMESTAMP_FULLPATH)
  else
    logger -p local2.notice -t moodle "Local timestamp file ($LOCAL_TIMESTAMP_FULLPATH) does not exist. Probably first time syncing? Continuing to sync."
    mkdir -p /var/www/html
  fi
  if [ "\$SERVER_TIMESTAMP" != "\$LOCAL_TIMESTAMP" ]; then
    logger -p local2.notice -t moodle "Server time stamp (\$SERVER_TIMESTAMP) is different from local time stamp (\$LOCAL_TIMESTAMP). Start syncing..."
    if [[ \$(find $SYNC_LOG_FULLPATH -type f -size +20M 2> /dev/null) ]]; then
      truncate -s 0 $SYNC_LOG_FULLPATH
    fi
    echo \$(date +%Y%m%d%H%M%S) >> $SYNC_LOG_FULLPATH
    rsync -av --delete /moodle/html/moodle /var/www/html >> $SYNC_LOG_FULLPATH
  fi
else
  logger -p local2.notice -t moodle "Remote timestamp file ($SERVER_TIMESTAMP_FULLPATH) does not exist. Is /moodle mounted? Exiting with error."
  exit 1
fi
EOF

chmod 500 ${SYNC_SCRIPT_FULLPATH}

local CRON_DESC_FULLPATH="/etc/cron.d/sync-moodle-html-local-copy"
cat <<EOF > ${CRON_DESC_FULLPATH}
* * * * * root ${SYNC_SCRIPT_FULLPATH}
EOF

chmod 644 ${CRON_DESC_FULLPATH}

# Addition of a hook for custom script run on VMSS from shared mount to allow customised configuration of the VMSS as required
local CRON_DESC_FULLPATH2="/etc/cron.d/update-vmss-config"
cat <<EOF > ${CRON_DESC_FULLPATH2}
* * * * * root [ -f /moodle/bin/update-vmss-config ] && /bin/bash /moodle/bin/update-vmss-config
EOF

chmod 644 ${CRON_DESC_FULLPATH2}
}

mkdir -p /var/www/html
rsync -av --delete /moodle/html/moodle /var/www/html
htmlRootDir="/var/www/html/moodle"
setup_html_local_copy_cron_job

```

- Save the above content by pressing keys.
```
Ctrl + O and press enter to save file.
Ctrl + X to come out of the file.
``` 
- Set the cron job by executing the below commands.
	```
	cd /home/azureadmin/
	bash setup_cron.sh
	```

-  **Restart Servers**
	- Restart nginx server & php-fpm server
		```
		sudo systemctl restart nginx
		sudo systemctl restart php(phpVersion)-fpm
		```
	- With the above steps Moodle infrastructure is ready.

-	**Log Paths**
		-   on-premises might be having different log path location and those paths need to be updated with Azure log paths. 
			-   Ex: /var/log/syslogs/moodle/access.log
			-   Ex: /var/log/syslogs/moodle/error.log 
		- Update log files location
			```
			nano /etc/nginx/nginx.conf
			# Above command will open the configuration file.
			# 
			# Change the log path location.
			# Find access_log and error_log and update the log path.
			#
			# After the changes, Save the file. 
			# Press CTRL+o to save and CTRL+x to exit.
			``` 
-	**Restart servers**
	
	- Restart the nginx and php-fpm servers
		```
		sudo systemctl restart nginx
		sudo systemctl restart php<phpVersion>-fpm
		```
</details>

## **Post Migration:** 
- Post Migration involves the following tasks.
	- Post migration tasks that include application configuration.
	- Update general configuration (e.g. log file destinations).
	- Update any cron jobs / scheduled tasks.
	- Configuring certificates.
	- Restarting PHP and nginx servers.
	- Mapping DNS name with the Load Balancer public IP
 <details> 
 <summary>(For detailed steps click on expand!)</summary>

- Post migration of Moodle application user need to update the log paths, SSL Certificates, update time stamp and restart servers as follows.

- **Virtual Machine Scale Set:**
- Perform the below steps in VMSS instance(s).
-  **Log Paths**
	-   on-premises might be having different log path location and those paths need to be updated with Azure log paths. 
		-   Ex: /var/log/syslogs/moodle/access.log
		-   Ex: /var/log/syslogs/moodle/error.log 
		- Update log files location
			```
			nano /etc/nginx/nginx.conf
			# Above command will open the configuration file.
			# 
			# Change the log path location.
			# Find access_log and error_log and update the log path.
			#
			# After the changes, Save the file. 
			# Press CTRL+o to save and CTRL+x to exit.
			``` 

-  **Restart servers:**
	- Restart the nginx and php-fpm servers
		```
		sudo systemctl restart nginx
		sudo systemctl restart php<phpVersion>-fpm
		```

- **Virtual Machine:**
- Perform the below steps in Controller Virtual Machine.

-   **Certs:**
	-   *SSL Certs*: The certificates for your Moodle application reside in /moodle/certs/
	- Copy over the .crt and .key files over to /moodle/certs/. The file names should be changed to nginx.crt and nginx.key in order to recognize by the configured nginx servers. Depending on your local environment, you may choose to use the utility scp or a tool like WinSCP to copy these files over to the cluster controller virtual machine.
	- You can also generate a self-signed certificate, useful for testing only:
		```
		openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /moodle/certs/nginx.key -out /moodle/certs/nginx.crt -subj "/C=US/ST=WA/L=Redmond/O=IT/CN=mydomain.com"
		```
	- It's recommended that the certificate files be read-only to owner and that these files are owned by www-data:www-data.
		```
		chown www-data:www-data /moodle/certs/nginx.*
		chmod 400 /moodle/certs/nginx.*
		```

 -  **Restart servers:**
	- Restart the nginx and php-fpm servers
		```
		sudo systemctl restart nginx
		sudo systemctl restart php<phpVersion>-fpm
		```

-  **Update Time Stamp:**
	- A cron job is running in the VMSS which will check the updates in time stamp for every minute. If there is an update in time stamp then local copy of VMSS (/var/www/html/moodle) is updated from shared directory (/moodle/html/moodle).
	- Update the time stamp to update the local copy in VMSS instance.
		```
		sudo -s
		/usr/local/bin/update_last_modified_time.moodle_on_azure.sh
		```

-  **Auto Scaling Rules**
	- Go to the Virtual Machine Scale Set Resource in Azure portal.
	
	- To enable autoscale on a scale set, first define an autoscale profile.
	- This profile defines the default, minimum, and maximum scale set capacity.
		```
		az monitor autoscale create --resource-group myResourceGroup --resource myScaleSet --resource-type Microsoft.Compute/virtualMachineScaleSets --name autoscale --min-count 2 --max-count 10 --count 2

		# Example: az monitor autoscale create --resource-group migration_option2 --resource migrationss --resource-type Microsoft.Compute/virtualMachineScaleSets --name autoscaling-profile --min-count 2 --max-count 10 --count 2
		```
	
	- In Scaling section, add a scale condition, user can add a rule to scale up and scale down an instance based up on the VM load.
		```
		# Auto scaling rules can be created by scaling precentage or by the scaling count.
		
		# Scale to 5 instances when the CPU Percentage across instances is greater than 75 averaged over 10 minutes.
		az monitor autoscale rule create -g myResourceGroup --autoscale-name {myvmss} --scale to 5 --condition "Percentage CPU > 75 avg 10m"
		
		# Example: az monitor autoscale rule create -g migration_option2 --autoscale-name autoscaling-profile --scale to 5 --condition "Percentage CPU > 75 avg 10m"
		
		
		# Scale down 50% when the CPU Percentage across instances is less than 25 averaged over 15 minutes.
		az monitor autoscale rule create -g myResourceGroup --autoscale-name {myvmss} --scale in 50% --condition "Percentage CPU < 25 avg 15m"
		
		# Example: az monitor autoscale rule create -g migration_option2 --autoscale-name autoscaling-profile --scale in 50% --condition "Percentage CPU < 25 avg 15m"
		```
-  **Mapping IP:**
	- Map the load balancer IP with the DNS name.
	-   Disable Moodle website from Maintenance mode.
         - Run the below command in Controller Virtual Machine.
              ```
                sudo /usr/bin/php admin/cli/maintenance.php --disable
             ```
         - To check the status of the moodle site run the below command.
             ```
                sudo /usr/bin/php admin/cli/maintenance.php
             ```
 	- Hit the load balancer DNS name to get the migrated moodle web page.
</details>

**General FAQ/Troubleshooting Questions**

<details> 
<summary>(For Similar questions click on expand!)</summary>

1.   Error: database connection failed: If you get errors like "database connection failed" or "could not connect to the database you specified", here are some possible reasons and some possible solutions.
	
		- 	Your database server is not installed or running. To check this for MySQL try typing the following command line
			```
			$telnet database_host_name 3306
			```
		- You should get a cryptic response which includes the version number of the MySQL server.
		- If you are attempting to run two instances of Moodle on different ports, use the ip address of the host (not localhost) in the $CFG->dbhost setting, e.g. $CFG->dbhost = 127.0.0.1:3308.
		- You have not created a Moodle database and assigned a user with the correct privileges to access it.
        - The Moodle database settings are incorrect. The database name, database user or database user password in your Moodle configuration file config.php are incorrect. 
		- Check that there are no apostrophes or non-alphabetic letters in your MySQL username or password.
2.  Error: "500:Internal Server Error": There are several possible causes for this error. It is a good idea to start by checking your web server error log which should have a more comprehensive explanation. However, here are some known possibilities....
			
	- There is a syntax error in your .htaccess or httpd.conf files. The way in which directives are written differs depending on which file you are using. You can test for configuration errors in your nginx files using the command:
		```
		nginx -t
		```
    - you may also see a 403: Forbidden error
	-  the webserver executes under your own username and all files have a maximum permissions level of 755. Check that this is set for your Moodle directory in your control panel or (if you have access to the shell) use this command:
		```
		#chmod -R 755 moodle
		```
3. Error "403: Forbidden"

	- This error means that the php memory_limit value is not enough for the php script. The memory_limit value is the "allowed memory size" - 64M in the example above (67108864 bytes / 1024 = 65536 KB. 65536 KB / 1024 = 64 MB). You will need to increase the php memory_limit value until this message is not shown anymore. There are two methods of doing this.
	- On a hosted installation you should ask your host's support how to do this. However, many allow .htaccess files. If yours does, add the following line to your .htaccess file (or create one in the moodle directory if it does not already exist):
		```
		php_value memory_limit <value>M
		Example: php_value memory_limit 40M
		```
	- If you have your own server with shell access, edit your php.ini file (make sure it is the correct one by checking in your phpinfo output) as follows:
		```
		memory_limit <value>M
		Example: memory_limit 40M
		```
	- Remember that you need to restart your web server to make changes to php.ini effective. An alternative is to disable the memory_limit by using the command memory_limit 0.

4. I can not log in - I just stay stuck on the login screen
	
	- This may also apply if you are seeing “Your session has timed out. Please login again” or "A server error that affects your login session was detected. Please login again or restart your browser" and cannot log in.
	- The following are possible causes and actions you can take.
	- Check first that your main admin account (which will be a manual account) is also a problem. If your users are using an external authentication method (e.g. LDAP) that could be the problem. Isolate the fault and make sure it really is Moodle before going any further.
	- Check that your hard disk is not full or if your server is on shared hosting check that you have not reached your disk space quota. This will prevent new sessions being created and nobody will be able to log in.
	- Carefully check the permissions in your 'moodledata' area. The web server needs to be able to write to the 'sessions' subdirectory. 
5. Fatal error: $CFG->dataroot is not writable, admin has to fix directory permissions! Exiting.

	- Check the moodle and moodledata permissions and make sure they are www-data:www-data only. If not change the group and ownership permissions.
	- Command to update the permissions

		```
		sudo chown -R /moodle/moodledata
		```

6. Could not find a top level course

	- If this appears immediately after you have attempted to install Moodle it almost certainly means that the installation did not complete. A complete installation will ask you for the administrator profile and to name the site just before it completes. Check your logs for errors. Then drop the database and start again. If you used the web-based installer try the command line one.

7. I log in but the login link does not change. I am logged in and can navigate freely.

	- Make sure the URL in your $CFG->wwwroot setting is exactly the same as the one you are actually using to access the site.

8. Error when uploading a file

 	- If you obtain a 'File not found' error when uploading a file, it indicates that slash arguments are not enabled on your web server. Please try enabling it.
	- If your web server does not support slash arguments, its use in Moodle can be disabled by un-ticking the checkbox 'Use slash arguments' in Administration > Site administration > Server > HTTP.
	- Warning: Disabling the use of slash arguments will result in SCORM packages not working and slash arguments warnings being displayed!

9. Site is stuck in maintenance mode

	- Sometimes Moodle gets stuck in maintenance mode and you will see the message "This site is undergoing maintenance and is currently unavailable" despite your attempts to turn-off maintenance mode.
	- When you put Moodle into maintenance mode it creates a file called maintenance.html in moodledata/maintenance.html (the site files directory). To fix this try the following:
	- Check that the web server user has write permissions to the moodledata directory.
	- Manually delete the maintenance.html file.
10. Where to find the logs

	- Syslog
        - While accessing a page either error or access log are generated.
        - They are captured at /var/log/nginx/ location.
	- Cron Log
        - Cron job will be running and it will update the local copy in instance.
        -  The path is  /var/log/sitelogs/moodle/cron.log.

</details> 
