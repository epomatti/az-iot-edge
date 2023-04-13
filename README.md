# az-iot-edge

Following the [quickstart](https://learn.microsoft.com/en-us/azure/iot-edge/quickstart-linux?view=iotedge-1.4) official tutorials.

## X.509 Certificate


Procecure for [X.509](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-provision-single-device-linux-x509?view=iotedge-1.4&tabs=azure-portal%2Cubuntu). Additional docs:

- How IoT Edge [uses certificates](https://learn.microsoft.com/en-us/azure/iot-edge/iot-edge-certs?view=iotedge-1.4).
- [Scalable X.509](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-provision-devices-at-scale-linux-x509?view=iotedge-1.4&tabs=individual-enrollment%2Cubuntu) using DPS
- [Test certificates](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-create-test-certificates?view=iotedge-1.4&tabs=linux)
- [IoT Hub root certificate migration](https://learn.microsoft.com/en-us/azure/iot-hub/migrate-tls-certificate?tabs=cli)
- [Transparent gateway](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-create-transparent-gateway?view=iotedge-1.4&tabs=iotedge)
- [CA-signed certificates](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-create-test-certificates?view=iotedge-1.4&tabs=linux#ca-signed-certificates)

Create the hub:

```sh
az group create --name IoTEdgeResources --location westus2
az iot hub create --resource-group IoTEdgeResources --name iothub789 --sku F1 --partition-count 2 --mintls "1.2"

# Upgrade the root if required
az iot hub certificate root-authority set --hub-name iothub789 --certificate-authority v2
az iot hub certificate root-authority show --hub-name iothub789
```

Create the certificates. Definition from the docs:

> You need the following files for manual provisioning with X.509:
> 
> - Two of device identity certificates with their matching private key certificates in .cer or .pem formats.
> 
>   One set of certificate/key files is provided to the IoT Edge runtime. When you create device identity certificates, set the certificate common name (CN) with the device ID that you want the device to have in your IoT hub.
> 
> - Thumbprints taken from both device identity certificates.
> 
>   Thumbprint values are 40-hex characters for SHA-1 hashes or 64-hex characters for SHA-256 hashes. Both thumbprints are provided to IoT Hub at the time of device registration.

Example for local certificates:

```
cd wrkdir

sudo bash certGen.sh create_root_and_intermediate

sudo bash certGen.sh create_edge_device_identity_certificate myEdgeDevice

sudo bash certGen.sh create_device_certificate "myEdgeDevice-primary"
sudo bash certGen.sh create_device_certificate "myEdgeDevice-secondary"
```

Create the device identity:

```
az iot hub device-identity create --device-id device_id_here --hub-name hub_name_here --edge-enabled --auth-method x509_thumbprint --primary-thumbprint primary_SHA_thumbprint_here --secondary-thumbprint secdonary_SHA_thumbprint_here



az iot hub device-identity create --device-id myEdgeDevice --edge-enabled --hub-name iothub789
az iot hub device-identity connection-string show --device-id myEdgeDevice --hub-name iothub789
```

Create the device (Ubuntu Server):

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

Create the device (Ubuntu Server):

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
