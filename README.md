# az-iot-edge

Following the [quickstart](https://learn.microsoft.com/en-us/azure/iot-edge/quickstart-linux?view=iotedge-1.4) official tutorials.

## Symmetric Key

Procedure for [symmetric key](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-provision-single-device-linux-symmetric?view=iotedge-1.4&tabs=azure-cli%2Cubuntu) attestation.

Create the IoT Hub:

```
az group create --name IoTEdgeResources --location westus2
az iot hub create --resource-group IoTEdgeResources --name iothub789 --sku F1 --partition-count 2 --mintls "1.2"
```

Create the device identity:

```
az iot hub device-identity create --device-id myEdgeDevice --edge-enabled --hub-name iothub789
az iot hub device-identity connection-string show --device-id myEdgeDevice --hub-name iothub789
```

Create the device (Ubuntu Serve):

```
az vm create \
  --resource-group IoTEdgeResources \
  --name vmiotedge \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --public-ip-sku Standard
```

Update the packages:

```
sudo apt update
sudo apt upgrade -y
```

Install Docker:

```
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb

sudo apt-get update
sudo apt-get install moby-engine
```

Create `/etc/docker/daemon.json` and add:

```json
{
  "log-driver": "local"
}
```

Then restart:

```
sudo systemctl restart docker
```

Install IoT Edge:

```
sudo apt-get update
sudo apt-get install aziot-edge
```

Config the device:

```
sudo iotedge config mp --connection-string 'PASTE_DEVICE_CONNECTION_STRING_HERE'
```

Run the checks:
```
sudo iotedge system status
sudo iotedge system logs
sudo iotedge check
```
