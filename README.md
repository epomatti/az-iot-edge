# Azure IoT Edge

Following the [quickstart](https://learn.microsoft.com/en-us/azure/iot-edge/quickstart-linux?view=iotedge-1.4) official tutorials.

## X.509 Certificate attestation

Procecure for [X.509](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-provision-single-device-linux-x509?view=iotedge-1.4&tabs=azure-portal%2Cubuntu). Additional docs:

- [How IoT Edge uses certificates](https://learn.microsoft.com/en-us/azure/iot-edge/iot-edge-certs?view=iotedge-1.4)
- [Scalable X.509 using DPS](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-provision-devices-at-scale-linux-x509?view=iotedge-1.4&tabs=individual-enrollment%2Cubuntu)
- [Test certificates](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-create-test-certificates?view=iotedge-1.4&tabs=linux)
- [IoT Hub root certificate migration](https://learn.microsoft.com/en-us/azure/iot-hub/migrate-tls-certificate?tabs=cli)
- [Transparent gateway](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-create-transparent-gateway?view=iotedge-1.4&tabs=iotedge)
- [CA-signed certificates](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-create-test-certificates?view=iotedge-1.4&tabs=linux#ca-signed-certificates)
- [IoT Hub Mutual-TLS](https://learn.microsoft.com/en-us/azure/iot-hub/iot-hub-tls-support?view=iotedge-1.4#mutual-tls-support)

Create the hub:

```sh
# IoT Hub
az group create --name IoTEdgeResources --location westus2
az iot hub create --resource-group IoTEdgeResources --name iothub789 --sku F1 --partition-count 2 --mintls "1.2"

# Upgrade the root to V2 if required
az iot hub certificate root-authority set --hub-name iothub789 --certificate-authority v2
az iot hub certificate root-authority show --hub-name iothub789
```

Create the Iot Edge (Ubuntu Server):

```
az vm create \
  --resource-group IoTEdgeResources \
  --name vmiotedge \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --public-ip-sku Standard
```

Connect to the VM:

```
ssh azureuser@<public-ip>
```

Update the packages:

```
sudo apt update
sudo apt upgrade -y
```

Restart the VM and connect again:

```
az vm restart -n vmiotedge -g IoTEdgeResources
```

Confirm that the IoT Hub root is available:

```sh
# IoT Hub Root V1
sudo ls -l /etc/ssl/certs | grep Baltimore

# IoT Hub Root V2
sudo ls -l /etc/ssl/certs | grep DigiCert
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



```sh
mkdir wrkdir
cd wrkdir

# Download the raw files for testing
curl "https://raw.githubusercontent.com/Azure/iotedge/main/tools/CACertificates/certGen.sh" --output certGen.sh

curl "https://raw.githubusercontent.com/Azure/iotedge/main/tools/CACertificates/openssl_root_ca.cnf" --output openssl_root_ca.cnf

# Root
sudo bash certGen.sh create_root_and_intermediate

# IoT Edge
sudo bash certGen.sh create_edge_device_identity_certificate myEdgeDevice
```

Generate the thumbprint:

> Using a single one here for testing only

```
openssl x509 -in iot-edge-device-identity-myEdgeDevice.cert.pem -fingerprint -noout -nocert
```

➡️ Remove the colons (`:`)

Create the IoT Edge device:

```
az iot hub device-identity create --device-id myEdgeDevice --hub-name iothub789 --edge-enabled --auth-method x509_thumbprint --primary-thumbprint primary_SHA_thumbprint_here --secondary-thumbprint secdonary_SHA_thumbprint_here
```

Add Microsoft packages:

```
sudo wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo rm packages-microsoft-prod.deb
```

Install Docker:

```
sudo apt-get update; \
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
sudo systemctl status docker
```

Install the Edge runtime:

```
sudo apt-get update; \
   sudo apt-get install aziot-edge
```

Provision the device:

```
sudo cp /etc/aziot/config.toml.edge.template /etc/aziot/config.toml
sudo vim /etc/aziot/config.toml
```

Certificate directory:

```sh
# Create the certificates directory
mkdir /etc/iotedge

# Copy the files
cp /home/azureuser/wrkdir/private/iot-edge-device-identity-myEdgeDevice.key.pem /tmp/iotedge/pk.pem
cp /home/azureuser/wrkdir/certs/iot-edge-device-identity-myEdgeDevice.cert.pem /tmp/iotedge/cert.pem

# Give the IoT Edge user permission
sudo chown -R iotedge: /tmp/iotedge
```

Edit the "Manual provisioning with X.509 certificate" section:

```toml
# Manual provisioning with X.509 certificate
[provisioning]
source = "manual"
iothub_hostname = "iothub789.azure-devices.net"
device_id = "myEdgeDevice"

[provisioning.authentication]
method = "x509"

# identity certificate private key
identity_pk = "file:///tmp/iotedge/pk.pem"

# identity certificate
identity_cert = "file:///tmp/iotedge/cert.pem"
```

Apply the configuration:

```
sudo iotedge config apply
```

Run the verification commands:

```
sudo iotedge system status
sudo iotedge system logs
sudo iotedge check
```

Using the Portal, add a marketplace Edge module, then check again:

```
sudo iotedge list
```

## Symmetric Key attestation

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

## Reference

https://www.youtube.com/watch?v=9Pe1ZF_KAfI