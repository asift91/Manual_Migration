## Azure Premium Files Manual Mounting
    -   This document explains how to mount a file share. 
## Prerequisites
-   Azure Storage account to be created.
-   A File Share to be created under the same Storage account.

-   **Create Storage Account from Azure Portal:**
    -   Now create the file storage with the following parameters in Azure portal 
    -   1.	Login to your Azure subscription and go the respective Resource Group
    -   2.	Create a storage account as follows
        -   Set the following parameters  
            - Storage account name: <storage account name>
            - Performance: Premium
            - Account Kind: FileStorage
            - Replication: Locally-redundant storage (LRS)

    - Click on Review + create button and the storage account will be created.
-   **Create Storage Account by Azure CLI:**
    -   Create a storage account from Azure CLI command
        ```
            az storage account create -n storageAccountName -g resourceGroupName --sku Premium_LRS --kind FileStorage -l eastus2euap
        ```
-   **Create File Storage:**
    -   Select the Storage resource from the resource group.
    -   Go to the storage account & click on the file storage.
    -   Now click on + File Share button to create a File Share.
    -   Fill the below parameters and create a file share
        •	Enter the name of file storage
        •	Select the capacity of file storage in the GiB
        •	Click on create 
    -   Azure CLI Command to create Azure File Share
        ```
            az storage share create --name moodle --account-name $storageAccountName --account-key $storageAccountKey --quota $diskSize
        ```
    -   It will create a file share with given name and given capacity.

-   **Mount Manually:**
    -   Below are the steps to mount Azure Premium Files manually.
    -   Now login into VM where you want to mount Azure Files and run the following steps.
            Note: Make sure that Azure Premium Files should be created before running the below steps.
    -   If moodle is installed then take backup of Moodle directory (/moodle/) and take the site to maintenance mode
    -   Run the below commands
        ``` 
            df -h           
            # This command will show the list of mounted systems
            umount /moodle  
            # This command will unmount the /moodle directory
            # Install cifs with the following command 
            sudo apt-get -y --force-yes install cifs-utils
            # Rename moodle directory to something else
            mv /moodle /moodle_old_delete_me
        ```
    -   Get the storageAccountName and storageAccountKey from Azure Portal
        -   storageAccountName : Storage account name 
        -   storageAccountKey : key1
-   **Set up and Mount Azure Files Share:**
    -   Create a credentials file with the moodle_azure_files.credential name
    -   Run the following commands
        ```
            cat <<EOF > /etc/moodle_azure_files.credential
            username=$storageAccountName
            password=$storageAccountKey
            EOF
            chmod 600 /etc/moodle_azure_files.credential
        ```
    -   It will create a file and change the permissions 
    -   Now run the following commands to set the azure premium files to fstab file
    -   Create a shell file “entry.sh” with the commands
        ```
            #!/bin/bash
                grep -q -s "^//$storageAccountName.file.core.windows.net/moodle\s\s*/moodle\s\s*cifs" /etc/fstab && _RET=$? || _RET=$?
                if [ $_RET != "0" ]; then
                    echo -e "\n//storageAccountName.file.core.windows.net/FileStorageName   /moodle cifs    credentials=/etc/moodle_azure_files.credential,uid=www-data,gid=www-data,nofail,vers=3.0,dir_mode=0770,file_mode=0660,serverino,mfsymlinks" >> /etc/fstab
                fi
                mkdir -p /moodle
                mount /moodle
        ```
    -   Run the entry.sh file with following command and Azure Premium Files will be mounted successfully.
        ```
        	bash entry.sh
        ```
    -   Move the local installation over to the Azure Files
        ```
            cp -a /moodle_old_delete_me/* /moodle || true
            rm -rf /moodle_old_delete_me
        ```
    -   Check the list of mounted systems.
        ```
            df -h
        ```
    -   Check the cron job status 
        ```
            sudo systemctl status cron
        ```
    -   Disable the site maintenance mode.
