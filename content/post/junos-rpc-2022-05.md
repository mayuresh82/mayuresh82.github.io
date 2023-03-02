---
title: "Junos Automation 101"
layout: post
date: 2022-05-20T09:00:00-08:00
published: true
author: "mayuresh82"
subtitle: "A look into the basics of Junos Automation using XML RPC"
image: "/img/epe-pt1.jpg"
tags:
    - ui
    - junos
    - network-automation
URL: "/2022/05/20/junos-xml-rpc"
categories: [ "Nautomation" ]
---

# Junos Network Device Automation 101

If you are exploring ways to automate Junos based devices, you will likely use one of two open source frameworks today:

1. [Junos PyEZ](https://github.com/Juniper/py-junos-eznc) which is Juniper's "official" Python library for managing Junos devices
2. [Napalm](https://github.com/napalm-automation/napalm) which is an abstracted vendor agnostic framework that basically uses PyEZ under the hood.

Its easy to simply `pip install` one of these high level libraries, import the module and get done with your automation task. However, have you ever wondered what goes on under the hood ? I am a strong proponent of mastering the basics when it comes to learning new technologies. For all the Network Engineers reading, would you be able to design an MPLS VPN today without knowing the basics of OSI ? Similarly it helps immensely to understand the basics of Juniper's basic XML API if you are going to be automating Junos devices.

Perhaps one of the most well known and most robust of RPCs in existence today of all the vendors is Juniper's Junos XML API. This API has existed for years and Juniper has gone to great lengths to keep it well maintained and bug free. Today it is the de-facto way ( or at least Juniper wants it to be ) to interact with a Junos based system. This API is capable of remotely performing most of the operations that you could normally invoke via the CLI. In this post, we will write a Python script from scratch using only primitive techniques to manage the configuration of a Junos device using the XML API. It assumes the reader has a basic knowledge of Python and XML.

## Junos XML API

Junos' XML Management protocol is an XML based protocol, meaning that it uses XML encoded data in a request-response fashion to invoke Remore Procedure Calls (RPCs) on the remote device. The [official docs](https://www.juniper.net/documentation/us/en/software/junos/automation-scripting/topics/concept/junos-script-automation-junos-xml-protocol-and-api-overview.html) provide a high level overview. The system comprises of both a protocol (or RPC) which defines the mechanism with which a client can interact with the server, as well as an Application Programming Interface or API which defines the methods and queries to use to fetch data or perform specific operations on the device. This API is essentially an XML representation of the CLI capabilities of the device. In both cases, the request and response data is encoded using XML. These two are sometimes collectively also known as the JunosScript API.

In the early days, the JunosScript API operated as a standalone API using its own proprietary protocol ( which is still supported ). Over time, the developers added NETCONF as the preferred communication protocol between the client and the server. RFC 4742 details the NETCONF over SSH protocol which is what Junos also implements.

At a high level, the NETCONF protocol works as follows:

1. The client application establishes a connection to the NETCONF server using SSH and opens the NETCONF session.

2. The NETCONF server and client application exchange initialization (hello) information, which is used to determine if they are using compatible versions of the Junos OS and the NETCONF XML management protocol.

3. The client application sends one or more requests to the NETCONF server and parses its responses.

4. The client application closes the NETCONF session and the connection to the NETCONF server.

One of the well known Python libraries that implements NETCONF is [ncclient](https://github.com/ncclient/ncclient) which abstracts away most of the above logic and is the underlying force powering things like Junos PyEZ. Since we are here to learn the basics, we will write a basic Python client from scratch that does each of the four steps detailed above.  Interestingly, Junos accepts both the new NETCONF style RPCs as well as the legacy JunosScript RPCs over the same NETCONF session. Unfortunately this can lead to a lot of confusion on the correct RPCs to use.

## Python script

From our requirements list above , we essentially need to support the following in our Python script:

- The ability to create an SSH session to a device
    - We dont want to implement the SSH protocol, so use the popular Paramiko library to handle SSH for us :)
- The ability to send XML encoded configuration data over the SSH session to the device
    - Establish a NetConf session to the Junos device
    - Send the configuration as XML encoded RPC requests (enclosed with the `<rpc>` tags)
- The ability to receive and parse XML encoded data from the device
    - Read XML encoded RPC responses (enclosed with the `<rpc-reply>` tags) and look for errors etc.

Here is the complete library that we can use for these operations, followed by a breakdown of each individual function:

```python
import socket
from lxml import etree # XML Element tree library
from paramiko.client import SSHClient, AutoAddPolicy # for basic SSH connections

class JunosRPC:

    def __init__(self, address, username, password, port=830,
                 timeout=30.0, private_key_file=None):
        self.address = address
        self.port = port
        self.username = username
        self.password = password
        self.private_key_file = private_key_file
        self.timeout = timeout
        self.channel = None

    def open(self):
        # open an SSH session to the device using Paramiko.
        print("[+] Opening SSH Connection")
        client = SSHClient()
        client.set_missing_host_key_policy(AutoAddPolicy())
        client.connect(
            self.address, port=self.port, username=self.username,
            password=self.password, key_filename=self.private_key_file,
            look_for_keys=False, timeout=self.timeout
        )
        # open a raw ssh channel back because we want to send custom (XML)
        # data back and forth
        chan = client.get_transport().open_session()
        chan.settimeout(2.0)
        chan.invoke_subsystem("netconf")
        hello = chan.recv(1024)
        self.channel = chan

    def close(self):
        self.channel.close()

    def send_data(self, data):
        # sends XML data over the SSH channel and returns the received response
        self.channel.send(data)
        recv = ""
        while True:
            try:
                resp = self.channel.recv(1024)
                if len(resp) == 0:
                    return recv
                recv += resp.decode()
                print(recv)
                if recv.endswith("]]>]]>"):
                    return recv
            except socket.timeout:
                return recv

    def cleanup_response(self, resp):
       if not resp:
           raise Exception("No response received")
       # remove the XML sync character from response
       resp = resp.replace("]]>]]>", "")
       try:
           root = etree.fromstring(resp)
       except Exception as ex:
           print("[-] {}: {}".format(ex, resp))
           raise
       # remove namespaces from the response
       for elem in root.getiterator():
            elem.tag = etree.QName(elem).localname
       etree.cleanup_namespaces(root)
       return root

    def netconf_open(self):
        # opens a netconf session using a hello
        print("[+] Opening NetConf session")
        hello = '''
<hello>
<capabilities>
    <capability>urn:ietf:params:netconf:base:1.0</capability>
</capabilities>
</hello>
]]>]]>
'''
        self.send_data(hello)

    def netconf_close(self):
        print("[+] Closing NetConf session")
        req = '''
<rpc>
    <close-session/>
</rpc>
]]>]]>
'''
        root = self.cleanup_response(self.send_data(req))
        if not root.find("ok"):
            raise Exception("Failed to close the session")

    def edit_config(self, config, operation='replace'):
        # loads a candidate config on the target device
        print("[+] Loading candidate config")
        req = '''
<rpc>
     <edit-config>
        <target><candidate/></target>
        <default-operation>{operation}</default-operation>
        <error-operation>stop-on-error</error-operation>
        <config-text>
            {config}
        </config-text>
    </edit-config>
</rpc>
]]>]]>
'''.format(operation=operation, config=config)
        root = self.cleanup_response(self.send_data(req))
        if not root.find("ok"):
            # look for any errors in the returned data
            errors = root.find("load-error-count")
            raise Exception("Failed to load config: {} errors reported".format(errors))

    def compare_config(self):
        # compares a previously loaded candidate config against the current active config
        print("[+] Compare config")
        req = '''
<rpc>
  <get-configuration compare="rollback" database="candidate" />
</rpc>
]]>]]>
'''
        root = self.cleanup_response(self.send_data(req))
        return root.find("configuration-information/configuration-output").text

    def commit_config(self):
        # commits the previously loaded candidate config on the device
        print("[+] Committing Candidate config")
        req = '''
<rpc>
    <commit/>
</rpc>
]]>]]>
'''
        root = self.cleanup_response(self.send_data(req))
        if not root.find("ok"):
            raise Exception("Failed to commit the config")

    def discard_config(self):
        print("[+] Discarding Changes")
        req = '''
<rpc>
    <discard-changes/>
</rpc>
]]>]]>
'''
        root = self.cleanup_response(self.send_data(req))
        if not root.find("ok"):
            raise Exception("Failed to discard the config")
```

We have a basic Python class set up here that accepts SSH connection details about the device and uses Paramiko to connect and open an SSH channel (the secure tunnel to the device) to the device. We have a method `send_data` that accepts raw data (in our case string XML requests), sends it over the channel, waits for a response and returns it. Notice the use of `chan.invoke_subsystem("netconf")`. This causes the SSH channel to be attached to the netconf subsystem on Junos devices. Read more about SSH subsystems [here](https://docstore.mik.ua/orelly/networking_2ndEd/ssh/ch05_07.htm). With this basic structure set up, we can implement the NetConf protocol on top.

The `cleanup_response()` method will remove any special characters from the response, parse the response using the `etree` library and remove any enclosed XML namespaces before returning the response. This is required because the Netconf RFC specifies the use of a special character sequence to mark the end of well formed XML documents, which must be removed. This cleaned response can then be used for other XML etree operations such as finding specific tags and elements. We then define a series of NetConf operational methods required to manipulate the config on the device:

- `netconf_open()` and `netconf_close()` methods are used opening and closing a Netconf session. When the netconf subsystem is invoked on the Junos device, it immediately sends a Netconf "Hello" message. The `netconf_open()` method is then used to send a client side "Hello" message which lists the capabilities of the client. 

- `edit_config()` is used for editing the active running config by using a candidate config. You can specify the edit operation here as either a `merge` or a `replace` ( the default being `merge`). You also specify that the operation needs to be halted in case of any errors, and you supply the configuration as a text blob.

- `compare_config()` is used for comparing the candidate config to the active config akin to issuing the CLI command `show | compare`. It provides a unix-like "diff" output that can be used to sanity check the changes. This is a powerful tool that allows you to verify the exact changes that will be applied before committing the config. Note that for this, we are using the older JunosScript RPC called `get-configuration` and supplying the XML attributes `compare=rollback` and `database=candidate` which indicates that we want to compare the candidate config against the current active configuration. I could not find the equivalent NETCONF `get-config` RPC to use for this use case.

- `commit_config()` is used for finally committing the candidate config in case the comparison checks out.

- `discard_config()` is used to discard the candidate config in case any errors are seen in the config.

We can now use this `JunosRPC` class inside a CLI script that accepts arguments from the user and performs a configuration change operation using NetConf:

```python
import argparse

def main():
    p = argparse.ArgumentParser()
    p.add_argument("--hostname", help="Device hostname")
    p.add_argument("--port", default=830, help="Netconf Port")
    p.add_argument("--username", help="SSH Username")
    p.add_argument("--password", help="SSH Password")
    p.add_argument("--keyfile", help="SSH Private key file")
    p.add_argument("--config", help="Junos Config File")

    a = p.parse_args()

    rpc = JunosRPC(a.hostname, a.username, a.password, port=a.port, private_key_file=a.keyfile)
    rpc.open()
    rpc.netconf_open()
    with open(a.config, "r", encoding="utf-8") as f:
        config_data = f.read()
    rpc.edit_config(config_data)
    diff = rpc.commit_config()
    print(diff)
    reply = input("\nDoes this look OK ? (Y/N)")
    if reply.lower() in ["yes", "y", "ye"]:
        rpc.commit_config()
    else:
        rpc.discard_config()
    rpc.netconf_close()
    rpc.close()

if __name__ == "__main__":
        main()
```

We can now run this script against a lab device running Junos:

