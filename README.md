# Configuring IoT Edge Nested Edge between two EFLOW Devices

## Overview

Below are the steps required to configure a nested IoT Edge deployment using EFLOW for both the top and lower device.  In order to simplify the deployment, the steps required have been extracted from multiple Azure IoT documentation locations including:
* [Create and provision an IoT Edge for Linux on Windows device using symmetric keys](https://docs.microsoft.com/en-us/azure/iot-edge/how-to-provision-single-device-linux-on-windows-symmetric?view=iotedge-2020-11&tabs=azure-portal%2Cpowershell)
* [Tutorial: Create a hierarchy of IoT Edge devices](https://docs.microsoft.com/en-us/azure/iot-edge/tutorial-nested-iot-edge?view=iotedge-2020-11)

These instructions will configure the devices to connect via symmetric keys to the Azure IoT Hub, and will create development certificates for the communication between the devices.  

**Before going to production:**
* **Test the deployment in a development environment**
* **Use production certificates for communication to the IoT Hub and between the nested devices**

## Steps

### Step 1 - Create the device installation scripts

* Log into [Azure Bash Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/quickstart).
* Extract the nested edge configuration tool to a directory.
  ``` 
  mkdir nestedIotEdgeTutorial
  cd ~/nestedIotEdgeTutorial
  wget -O iotedge_config.tar "https://github.com/Azure-Samples/iotedge_config_cli/releases/download/latest/iotedge_config_cli.tar.gz"
  tar -xvf iotedge_config.tar
  ```
* Edit the config utility file
  ```
  code ~/nestedIotEdgeTutorial/iotedge_config_cli_release/templates/tutorial/iotedge_config.yaml
  ```
  - Update iothub_hostname (hubName.azure-devices.net) and iothub_name (hubName)
  - Close the file
* Run the installation
  ```
  mkdir ~/nestedIotEdgeTutorial/iotedge_config_cli_release/outputs
  cd ~/nestedIotEdgeTutorial/iotedge_config_cli_release
  ./iotedge_config --config ~/nestedIotEdgeTutorial/iotedge_config_cli_release/templates/tutorial/iotedge_config.yaml --output ~/nestedIotEdgeTutorial/iotedge_config_cli_release/outputs -f
  ```
* Export the following files to your local C: drive
  ```
  ~/nestedIotEdgeTutorial/iotedge_config_cli_release/outputs/lower-layer.zip
  ~/nestedIotEdgeTutorial/iotedge_config_cli_release/outputs/top-layer.zip
  ```
* Extract the files from each zip file into their own directory
* Edit the ```install.sh``` file **in both directories** (ie. for both the top level and the lower level).  
  - Replace
    ```
    cp iotedge_config_cli_root.pem /usr/local/share/ca-certificates/iotedge_config_cli_root.pem.crt
    update-ca-certificates
    ```
    with
    ```
    cp iotedge_config_cli_root.pem /etc/pki/ca-trust/source/anchors/iotedge_config_cli_root.pem.crt
    update-ca-trust
    ```
  - Save the file
* Edit the config.toml file **in both directories**
  - Replace the following
    ```
    [listen]
    workload_uri = "fd://aziot-edged.workload.socket"
    management_uri = "fd://aziot-edged.mgmt.socket"
    ```
    with
    ```
    [listen]
    workload_uri = "unix:///var/run/iotedge/workload.sock"
    management_uri = "unix:///var/run/iotedge/mgmt.sock"
    ```
  - Save and close the file

### Step 2 - Setup networking

* Create an External Switch on the **top device**
  - Open Hyper-V Manager  
  - Under "Actions" select "Virtual Switch Manager..."
  - Choose "External"
  - Click on "Create Virtual Switch"
  - Give your switch a name.  e.g. "EFLOW"
  - Click on "External network"
  - Under "External network", choose the network you are currently using
  - Enable the "Allow management operating system to share this network adapter" check box
  - Leave all of the other defaults
  - Click "Apply"
  - Click "OK"
* Open Ports
  - On the **top device** open the following ports
    ```
    sudo iptables -A INPUT -p tcp --dport 5671 -j ACCEPT
    sudo iptables -A INPUT -p tcp --dport 8883 -j ACCEPT
    sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
    ```
  - To allow pings on the **top device** allow ICMP messages
    ```
    sudo iptables -A INPUT -p icmp --icmp-type 8 -s 0/0 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
    ```
  - To make the simulation module work on the **lower device** open the AMQP port
    ```
    sudo iptables -A INPUT -p tcp --dport 5671 -j ACCEPT
    ```
* If you are using a Hyper-V VM as the lower device and your PC as the top device, be sure to set the Hyper-V VM network to use the external switch you created above
  - Open Hyper-V Manager
  - Right click on your VM, choose "Settings..."
  - Click on "Network Adapter"
  - Select the external switch you created above
  - Click on "Apply", then "OK"
* Test connectivity from the lower device to the top device using a ping
  ```
  ping x.x.x.x
  ```
### Step 3 - Install EFLOW on both devices

Complete the following steps on both devices.
* Ensure that nested virtualization is enabled on the VM.  If the device is in a VM open PowerShell in admin mode and run the following while the VM is not running
  ```
  Set-VMProcessor -VMName <VMName> -ExposeVirtualizationExtensions $true
  ```
* Install EFLOW 
  - Open PowerShell in admin mode
  - Enable Hyper-V
    ```
    Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All)
    ```
  - Check if local mashin is AllSigned
    ```
    Get-ExecutionPolicy -List
    ```
  - If not run the following
    ```
    Set-ExecutionPolicy -ExecutionPolicy AllSigned -Force
    ```
  - Complete the install
    ```
    $msiPath = $([io.Path]::Combine($env:TEMP, 'AzureIoTEdge.msi'))
    $ProgressPreference = 'SilentlyContinue'
    Invoke-WebRequest "https://aka.ms/AzEFLOWMSI-CR-X64" -OutFile $msiPath

    Start-Process -Wait msiexec -ArgumentList "/i","$([io.Path]::Combine($env:TEMP, 'AzureIoTEdge.msi'))","/qn"
    ```
  - On the **top device**, specify the switch that was created in Step 2.  Please insert the name of your external switch in the snippet below
    ```
    Deploy-Eflow -vSwitchType "External" -vSwitchName "ExternalSwitchName"
    ```
  - On the **lower device** no parameters are necessary
    ```
    Deploy-Eflow
    ```
* Configure EFLOW
  - Copy the appropriate directory (top or lower) from Step 1 to the local machine's C: drive
  - Change directory in PowerShell to the directory where the files are located
  - Copy the files into the EFLOW Mariner VM
    ```
    Copy-EflowVmFile -fromFile *.* -tofile ~/ -pushFile
    ```
  - Start an SSH session in the Mariner VM
    ```
    Connect-EflowVm
    ```
  - Create the following directory
    ```
    sudo mkdir /etc/ca-certificates
    sudo chmod 755 /etc/ca-certificates
    ```
  - Update the permissions on the ```install.sh``` file
    ```
    sudo chmod 755 install.sh
    ```
  - Run install.sh
    ```
    sudo ./install.sh
    ```
    - Provide the current device's IP address when prompted for the Host name
    - If the device is the **lower device** then provide the ip address of the parent device
  - Run the following Linux commands to create required directories and set permissions.  
    > **Warning**
    > The permissions below should be tightened up before releasing to production.
    ```
    sudo mkdir /var/run/iotedge
    sudo chmod 777 /var/run/iotedge
    sudo chown iotedge /var/run/iotedge
    sudo chmod 777 /etc/aziot/certificates -R
    ```
  - Apply the configuration changes
    ```
    sudo iotedge config apply
    ```
## Step 4 - Confirm everything is working properly

* On the **top device** perform the following checks
  ```
  # Perform an IoT Edge check
  sudo iotedge check
  
  # Confirm all modules are running.  You should see:
  #   - edgeAgent
  #   - edgeHub
  #   - registry
  #   - IoTEdgeAPIProxy
  docker ps
  
  # Check the logs for all of the modules.  (This can also be done in the portal)
  docker logs edgeAgent
  docker logs edgeHub
  docker logs restistry
  docker logs IoTEdgeAPIProxy
  ```
* On the **lower device** perform the following checks
  ```
  # Perform an IoT Edge check
  sudo iotedge check
  
  # Confirm all modules are running.  You should see:
  #   - edgeAgent
  #   - edgeHub
  #   - simulatedTemperatureSensor
  docker ps
  
  # Check the logs for all of the modules.  (This can also be done in the portal)
  docker logs edgeAgent
  docker logs edgeHub
  docker logs simulatedTemperatureSensor
  ```
* Install the [Azure IoT Explorer](https://docs.microsoft.com/en-us/azure/iot-fundamentals/howto-use-iot-explorer) or the [Azure IoT CLI](https://github.com/Azure/azure-iot-cli-extension) and verify that messages are coming in from the simulation module.  If you are installing on a private network, try running the Azure CLI in the Azure Bash Shell.

## That's it - all done!
