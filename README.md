# Cybus Learn - AWS IoT

This repository describes how to connect AWS IoT Core or AWS IoT Greengrass.

The purpose is to give Cybus Connectware users a guideline how to integrate
Connectware with AWS IoT Core endpoints directly 
or with an AWS IoT Greengrass Core acting as a gateway to AWS IoT.

## Prerequisites

The Cybus Connectware and an AWS IoT Core endpoint are required.

The latter could be either a direct cloud connection to AWS IoT Core
or a connection to an AWS IoT Greengrass edge device, which is meant to run an 
AWS IoT Greengrass runtime near the network environment of the devices to be integrated.

This guide shows two examples:
- Using the AWS IoT Core directly
- Using an AWS IoT Greengrass Core as the gateway to AWS IoT Core

### Directly communicating with MQTT instead of using the AWS (IoT) SDK

This goes without using the AWS (IoT) SDK and thus without a specific Cybus 
connector service so that there is no need for an additional docker container.

Cybus Connectware is fully capable of using MQTT (with or without TLS) with AWS.
The SDK is helpful for managing the environment and working more easily for
example with well-defined Device Shadows and the like.

For more information see 
[AWS IoT SDKs](https://docs.aws.amazon.com/iot/latest/developerguide/iot-sdks.html)

## Configure the Connectware for an AWS IoT endpoint

Follow these general settings in your AWS IoT service commissioning file:
 
- Get a [Cybus Connectware](https://www.cybus.io/) (Version 1.0.13 or higher) and a license key.
- Prepare a service commissioning file
- Configure the required `Aws_IoT_Endpoint_Address`
- Configure the machine topic to publish events for (placeholder `machineTopic`)
- Configure the PEM formatted certificates and keys for an AWS IoT device

To find out the AWS IoT endpoint defined for your AWS account, use the AWS CLI:
```
aws iot describe-endpoint --endpoint-type iot:Data-ATS
```

The response contains the AWS account specific ATS (Amazon Trust Services) 
endpointAddress to be used as the MQTT hostname:
```
{
  "endpointAddress": "a7t9.....1pi-ats.iot.eu-central-1.amazonaws.com"
}
```

To use the required certificates for a secure TLS connection to AWS IoT the
following elements need to be configured in the service commissioning file:
- caCert: the root certificate provided by Amazon (AmazonRootCA1.pem)
- clientCert: the device certificate
- clientPrivateKEy: the device private ky

AWS IoT offers different ways to create device certificates. It is also possible
to do this by a proper CSR process or using an own CA.

The client resources are obtained from AWS IoT after creating a device.
To be able to successfully establish a MQTT connection this device must be activated
and its certificate must be associated with a policy document allowing specific
resources to use specific operations on AWS IoT Core, as an example:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iot:Publish",
        "iot:Subscribe",
        "iot:Connect",
        "iot:Receive"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "greengrass:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```
  
The placeholder `Cybus::MqttRoot` is replaced with the MQTT root topic of the service,
which is defined as `services/<serviceId>` with serviceId derived from the metadata name.
This is for example in case of `AWS IoT Core Test` finally `services/awsiotcoretest`.

- Publish to the MQTT topic with some payload from your device (or a Node-RED flow):
see the [Node-RED flow](node-red-flow-test-event.json) for use with the Connectware workbench.

An event produced in the Node-RED flow:
```
{
  "DeviceData": {
    "Temperature": 78.567,
    "Position": {
      "X": 35.02,
      "Y": 12.62,
      "Z": 3.45
    },
    "Status": "operational"
  }
}
```

is sent to the respective endpoint as (in this example):
```
{
  "deviceId": "TestDevice",
  "payload": {
    "Temperature": 78.567,
    "Position": {
      "X": 35.02,
      "Y": 12.62,
      "Z": 3.45
    },
    "Status": "operational"
  }
}
```

In the example we choose a source topic `${Cybus::MqttRoot}/test/#topic` to demonstrate 
a transformation rule within the Cybus::Mapping resource.
Subscribing `#topic` and publishing to `$topic `is meant as an implicit wildcard mapping
meaning, that any topic is mapped to the target connection without a name change.

Also, the transformation rule contains a `$` meaning, that the payload is also transferred as is
in the payload element without any change (It's up to the operator to apply any other rules, 
filters, and the like matching any edge to cloud use case).

### Connect to AWS IoT Core

For IoT Core connections create a device in AWS IoT to get a device certificates,
and assign an [AWS IoT Core policy](https://docs.aws.amazon.com/iot/latest/developerguide/iot-policies.html) access policy.
 
- Prepare a service commissioning file: 
see the [AWS Iot Core Service Example](aws-iot-core-connectware-service.yml)

- Set the `Aws_IoT_Endpoint_Address` for a Cybus MQTT connection resource in the format: 
  `<aws-account-iot-core-id>-ats.iot.eu-central-1.amazonaws.com`

After the respective certificates mentioned above are configured 
there is nothing more to configure for working with the AWS IoT Hub via MQTT over TLS.
You may then configure Endpoint and Mapping resources following the
[Cybus resource documentation](https://docs.cybus.io/latest/user/resources/index.html).

### Connect to AWS IoT Greengrass Core

For IoT Greengrass connections create and deploy an AWS IoT Greengrass Core environment
onto a machine capable of running a Greengrass Core Runtime.

This requires Setup of a Greengrass Group in the IoT Console, where Greengrass Cores,
devices, subscriptions, Lambda functions, connectors and resources like Machine Learning
models and secrets are configured, and then provisioned to the greengrass core instance.

Like for a direct connection create a device, get proper certificates and a policy
and use the respective AWS IoT Greengrass Core (GGC) endpoint address.

To get events from the GGC into the AWS IoT Cloud you need to provision a subscription
defining a GGC device as a source and IoT Cloud as a target.

To get events from the connected device to the GGC you may need to add the Hostname or
IP Address of the GGC to an additional Core endpoint definition in the IoT Console.
This is a possible configuration step, if e.g. a EC2 instance has a public IP a/o
or custom Hostname the GGC doesn't during the setup. Manually adding the new core endpoint
address is good practice instead of manipulating the Connectware hosts /etc/hosts file.

The GGC is acting as an extensible gateway capable of hosting AWS Lambda functions,
ML resource and connectors like [AWS IoT Site](https://aws.amazon.com/iot-sitewise/)
in order to provide additional capabilities to this edge environment (e.g. OPC-UA support).
 
- Prepare a service commissioning file: 
see the [AWS IoT Greengrass Service Example](aws-iot-greengrass-connectware-service.yml)

- Set the `Greengrass_Core_Endpoint_Address` for a Cybus MQTT connection resource in the format: 
  `<the-greengrass-core-endpoint-hostname-or-ip>`

- Set the `awsGreengrassClientId` to the name of the AWS IoT device associated with
the Greengrass group, this is required to be authorized after the mutual authentication.
A connection to GGC is unique for a device identified by its name and certificates,
there are no concurrent connections possible for the same device.

The configuration of the device certificates data remains the same as mentioned above.
Instead of the Amazon Root CA the Greengrass Group Certificate Authority has to be used.
This can be obtained using the Amazon CLI using the Greengrass groupId, for example:
```
aws greengrass list-groups
aws greengrass list-group-certificate-authorities --group-id "4824e138-f042-42be-addc-fcbde34587e7"
aws greengrass get-group-certificate-authority --group-id "4824e138-f042-42be-addc-fcbde34587e7" \
   --certificate-authority-id "2f50c373ee3ab10b039ea4a99eaf667746849e3fd87940cb3afd3e1c8de054af"
```

The JSON Output contains a field `PemEncodedCertificate` containing the requested information

The operator creating the greengrass connector service must have this information, 
the client certificate data for the device associated with the Greengrass group,
and the Device Name as the MQTT clientId.

There is nothing more to configure for working with the AWS IoT Greengrass Core via MQTT over TLS.
You may then configure Endpoint and Mapping resources following the
[Cybus resource documentation](https://docs.cybus.io/latest/user/resources/index.html).

## Verify successful integration

To see a successful integration you may use the AWS IoT Console and test by a subscription to #.
See [View MQTT Messages](https://docs.aws.amazon.com/iot/latest/developerguide/view-mqtt-messages.html)
for details.

## Using Device Shadows and higher level AWS IoT tools

Using this guide associates a "Classic Shadow" to the unspecified device.
You can work with any topic and any payload that may be produced by the Connectware.

However, the actual purpose of cloud-based IoT integrations is to apply advanced post-processing
for Analytics or other complex fully-managed operations in a properly set up IoT use case also
using other Cloud resources like Lambda functions, persistence solutions and the like.

For these it is from an architectural point of view a good practice to define a proper domain model
and respective data structures those services should work with. Having well-specified data objects
defined as Device Shadows allow the cloud infrastructure to operate on these devices in case of a
disconnected edge environment hosting the physical device resp. the gateway to AWS IoT.

This causes the Device definition to expect this data structure as a constraint when receiving
data from the southbound resources. In our case, the Connectware accessing AWS IoT must ensure
to sent properly formatted data (mostly json), which is easily achievable with Cybus Connectware 
with any kind of connected devices over nearly arbitrary communication protocols.

## References

- [Using the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide)
- [AWS IoT Homepage](https://aws.amazon.com/iot)
- [AWS IoT Core](https://aws.amazon.com/iot-core)
- [AWS IoT Greengrass](https://aws.amazon.com/greengrass)
- [How to install Greengrass Core](https://docs.aws.amazon.com/greengrass/latest/developerguide/install-ggc.html)
- [Device authentication and authorization for AWS IoT Greengrass](https://docs.aws.amazon.com/greengrass/latest/developerguide/device-auth.html)
- [Greengrass Device Connection Workflow](https://docs.aws.amazon.com/greengrass/latest/developerguide/gg-sec.html#gg-sec-connection)
- [Cybus Homepage](https://www.cybus.io/)
- [Cybus Connectware documentation](https://docs.cybus.io)
- [Cybus Learn Service Basics](https://learn.cybus.io/lessons/mqtt-basics/)
