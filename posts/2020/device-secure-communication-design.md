.. title: Device Authentication and Secure Communication - Design
.. slug: device-secure-communication-design
.. date: 2020-07-08 15:32:21 UTC+05:30
.. tags: Systems, Security
.. category:
.. link:
.. status:
.. description:
.. author: Abhijit Gadgil
.. type: text
.. summary: Here, we discuss an overview of a secure mechanism, where a set of devices (acting as REST clients) are authenticated using client side certificates. This scenario is quite common in many IOT deployments. The discussed approach easily scales to a large number of devices (hundreds to thousands).

# Introduction

In a recent project, we were developing a device management system utilizing REST. That is - the device would communicate with management Server using a protocol that was based on REST. These end-points were for device configuration and management as well as alarms and statistics reporting. Since the communication is based on REST, we needed a mechanism for authenticating individual devices that were making API calls. In this post, we discuss in details design of such a system that is based on self signed Client Side TLS Certificates.

# Requirements

The system was designed to manage a few hundreds of devices that are deployed at geographically different locations. Mostly the devices will be in a controlled environment (that is the devices are not there totally out in the wild), but still it's better to assume that it is not impossible that someone unauthorized can gain an access to one of the devices and the device can then be compromised. If the device gets compromised, it needs to be pulled off the network. It should also be possible to authenticate the devices uniquely, possibly using Mac Address or any such reasonable mechanism that identifies a device uniquely. A detailed list of requirements can be summarized as below -

1. It should be possible to identify a device uniquely that is making the API Calls.
2. If a device is known to be compromised, it should be possible to revoke access to REST API for that particular device without affecting any other devices.
3. The mechanism should easily scale to a large number of devices - several hundreds to a few thousands at-least.
4. The devices will be typically be managed by a cloud hosted management system, that is the API endpoints are publicly available.
5. It should work in a typical cloud based web application deployment - using some load balancers front-ending web applications.

There are other requirements pertaining to device management, but those are not enumerated here since those are out of scope for the current discussion about securing the access to API.

# High Level Design Choices

## Device Certificates signed by Self Hosted CA

After looking at the requirements above, securing the API communication using Client Side Certificates seems to be a good option and it was decided that we should go ahead with this option. Since the devices will be managed under a same administrative domain, it was also decided to use self signed certificates, since cost implications of certificates for such a large number of devices would be very high if we use certificates trusted issued by well known third party CA (note: for this particular it's not possible to use [let's encrypt](https://letsencrypt.org/how-it-works/)){:target="\_blank"}. Since the devices will be communicating with the Server that is managed by the same entity that is deploying/authenticating the devices, authentication using public CA would not add much more value than a well secured self hosted CA. Also, hosting our own CA would offer some flexibility in terms of re-issuing device certificates if required. So following mechanism was chosen - Using a securely hosted CA (ie. essentially a machine on premises that is detached from the network.), issuing device certificates. To uniquely identify the device Common Name (CN) of the certificates will be associated with the device MAC Address.

## Assigning Certificates to the Device - Over the Air

The next question is how do we actually deploy the certificates to individual devices. It is possible to configure those certificates during the manufacturing of the devices, but in such scenarios, re-issuing the certificate for a device that was possibly compromised but is now recovered would require having an access to the device and would require bringing the device back. It should be possible to re-issue a certificate to a device over the air, as some of the locations where the devices will be located could be remote.

## Device Certificate Assignment - Protocol

A device accessing the Rest based API, follows a particular protocol where a set of messages are exchanged in a specific order. Exact details of the protocol are not discussed here, as many of the messages relate to device configuration and management. It is sufficient to distinguish that from the authentication point of view there are two sets of messages - Fist an *Initialize* message and Second "All messages other than the *Initialize* Message".

It should also be noted that - through an admin console, it is required to explicitly provision the device in the management system. That is before the device can start making any API calls, it should be provisioned in the management system by an authenticated user. This provisioning is a one time activity and is a part of device deployment workflow. If a device that is not provisioned in the management system already, any communication from the device can be replied with an HTTP Status Code of 404 - Not Found. That is an un-provisioned device will not be able to make any unsafe API Calls (API Calls that update the database). Note: this mechanism alone is not sufficient to guarantee the authenticity of the device. A device (or to be specific any User Agent - even a program) that is masquerading as a device can still be able to make DOS attack on the management system by simply trying to make API calls at a very high rate. This can be mitigated by using - 1. client specific throttling of API calls and/or IP address black-listing. Mitigating such DOS attacks is beyond the scope for the particular problem being discussed. For the remainder of the discussion, it is sufficient to assume that a device that is not provisioned will not be able to make unsafe (`POST`, `PUT`) API calls to the management system.

Coming back to the original problem of issuing the certificate to the device, instead of issuing the certificate to the device during manufacturing, the certificate is issued to the device as a part of protocol state machine. Following sequence of events achieves this -

1. Device is provisioned on the management system using suitable admin console.
2. A self-signed device certificate associated with MAC address is generated using Self hosted CA. The association with the Mac Address is achieved by using the certificate common name of the form `MacID-AA:BB:CC:DD:EE:FF` where `AA:BB:CC:DD:EE:FF` is the MAC address of the device.
3. During the manufacturing of the device, a special certificate tied to MAC Address of all Zeros (that is identifying All devices) is copied on the device. This means all the devices can be manufactured in an automated way without requiring any device specific changes during the manufacturing. It is not even required to know the MAC address of the device during manufacturing for the purpose of issuing the certificate.
4. A provisioned device starts communicating with the API Server using the All MAC Addresses Certificate. Only "Initialization" message is accepted from the device if the device uses 'All MAC Addresses' certificate. All other messages are responded with 403 - Forbidden Status. Thus, the device is prevented from making any unsafe API calls with "All MAC Addresses" Certificate.
5. Once the device is provisioned on the API server and the device MAC address specific certificate is generated, this certificate is uploaded to the API Server. Note: only authenticated users with sufficient privileges can upload this certificate using admin console. The communication with the admin console is always secured using TLS using a standard HTTPS based mechanism.
6. If an *Initialize* Message is received from a device, MAC address for the device is obtained from the message (the details of how this is achieved will be discussed in the subsequent post), if a certificate for this MAC address is already uploaded, this certificate is transferred to the device as a response to this *Initialize* message. Since this communication also uses TLS, the certificate is transferred to the device in a secured way. Also, note: this certificate is not transferred on *every* *Initialize* message, but only once.
7. For all subsequent communication the device uses this newly issued certificate.
8. Once it is known at the API Server side that the device certificate is issued to the device even subsequent *Initialize* messages MUST use the device specific certificate only. If the device uses "All Devices MAC" certificate the *Initialize* message will also be replied with "403 - Forbidden".
9. During the course of operation, if it is known that this particular device may be 'compromised', there exists a mechanism implemented using 'admin console' to 'revoke' the certificate. Note: This 'revoke' function is not identical to actual revocation of TLS certificate. But it achieves a similar purpose. Also, it is worth noting that if a certificate of a single device is revoked, all other devices continue to operate without any intervention if they have valid certificates.
10. Once a certificate for a device is 'revoked', the device is as good as in a state similar to the initial state, that is just after manufacturing. A device whose certificate is 'revoked' cannot communicate with the API Server again unless a new certificate is uploaded to the device by the administrator using the admin console. The device can still send *Initialize* message(s).

## Possible Limitations

The above mechanism achieves the desired properties of secure communication and meets the requirements specified above, it is worthwhile mentioning some potential limitations of this approach and their impact -

1. As discussed already, this mechanism does not defend against DOS attacks since, it is always possible for the attacker to gain access to "All MAC Addresses" certificate (if the attacker manages to get a Console access to the device) and the use this certificate to launch a DOS attack. However, for the considered deployment scenario, the possibility of such an attack is considerably low.
2. There is another possible attack that is possible assuming an attacker gains access to the certificate as discussed above. The authentication mechanism works by looking at the Mac Address that is part of the *Initialize* message, it is not impossible to generate an initialize message. (Note: Schema of this communication is not public, but it is reasonable to assume an attacker having access to the device can decipher the Schema for the *Initialize* message and can forge one). And the MAC Address in the *Initialize* message is just believed to be valid one by the API Server. As such it is impossible to actually determine the MAC address of the client in a typical cloud deployment. There is a finite window during which if an attacker manages to send *Initialize* message once the device specific certificate is uploaded and attacker has gained access to "All Devices MAC" certificate and send the *Initialize* message to the API Server. API Server in this case would end up sending the certificate to the attacker and attacker can communicate with the API Server masquerading as a device. However if it is identified that the device is indeed compromised, the certificate can simply be 'revoked' and this particular device is then not allowed to make further API calls.
3. Another possible limitation, not specific to this particular design but worth mentioning is - All device certificates are stored in the database associated with the API Server. If it is possible to gain access to the database, it's possible to leak the device certificate. But this particular limitation is not of the protocol or the mechanism itself, however important to know this particular attack surface. The standard mechanisms to mitigate this risk should be employed.

# Summary

To conclude, we have discussed a design for secure device communication with client side TLS certificates and communication protocol. In an actual cloud based deployment, there are other moving parts that come into picture like the load-balancers provided by the cloud provider, a web server and actual web application that is serving the API. In the subsequent post, we will discuss some deployment aspects using AWS Load Balancer, nginx based web server front-ending a Django REST Framework powered API Server using Let's Encrypt HTTPS Certificates.

