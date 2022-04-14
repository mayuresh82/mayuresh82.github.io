---
title: "Building Dynamic Inventories in Nornir"
layout: post
date: 2020-12-21T15:05:46-08:00
published: true
author: "mayuresh82"
subtitle: "Extending Nornir's functionality to build dynamic inventories from Netbox"
image: "/img/nornir_inv.jpg"
tags:
    - nornir
    - netbox
    - inventory
    - dynamic
URL: "/2020/12/25/nornir_inventories"
categories: [ "Nautomation" ]
---

# Building Dynamic Inventories in Nornir

 Organizing your Nornir inventory into multiple groups based on device attributes is a useful feature that allows running tasks against specific groups of hosts. Other frameworks like Ansible ship with a powerful dynamic inventory management system that allows grouping and filtering. While Nornir 3.0 has a community driven [Netbox Plugin](https://github.com/wvandeun/nornir_netbox), it is pretty basic and includes a simple way to pull in hosts from Netbox. In this blog post, we will see how we can extend this Netbox plugin to build a dynamic inventory with powerful grouping and run-time filtering capabilities.

*Note that the intent of this exercise is not to replicate Ansible's functionality in Nornir but rather to showcase the benefits of learning and incorporating more Python into your automation stack.*   

## Creating Inventory Groups

In a production environment, you may want to organize your inventory into groups ( based on device attributes such as location, role and others ) so that tasks such as configuration pushes can be performed against specific sets of devices. While the Netbox inventory plugin for Nornir is able to build a Nornir inventory using the Netbox API, it does not provide any grouping functionality, quite possibly due to the fact that the how and why of grouping is usually very specific to individual network environments. However, using a little bit of YAML and Python, we can customize the inventory build in a way that can serve a vast majority of production environments.

Let start by installing Nornir and the Nornir-Netbox inventory plugin mentioned above along with a few other dependencies:

```
pip install nornir nornir-netbox ruamel.yaml dotty-dict
```

Lets define a custom configuration/settings file for the plugin in YAML as shown below:

<span style="font-family:monospace">my_config.yaml:</span>
```yaml
nb_url: "https://netbox.live"
nb_token: "123_NETBOX_API_TOKEN_456"
filter_by:
    device_role: ['leaf-switch', 'core-switch', 'spine-switch']
group_by:
    - name
    - device_role.slug
    - site.slug
    - platform.slug
```

So what do we have here ? The first two lines simply define the parameters needed for the plugin to reach Netbox. The next block `filter_by` defines query filters. In this case, we limit the queries to all Juniper and Cisco network devices of type spine, leaf and core. Filtering the Netbox queries is useful in cases where you also have other device types (e.g servers) that you dont want as part of your inventory. This functionality already exists in the plugin, and we can simply pass this data as is to the plugin. 

Next, we define the more interesting `group_by` clause. Here is where we define how we want Nornir groups to be created. If you are not familiar with groups in Nornir, you can refer to [Nornir Inventory](https://nornir.readthedocs.io/en/latest/tutorial/inventory.html). Essentially, Nornir allows each host in the inventory to be part of a set of groups. This will be the basis for our advanced filtering scheme at run-time. Now lets write some actual code that reads the config file and builds out our inventory from Netbox:

<span style="font-family:monospace">custom_inventory.py:</span>
```python
from nornir_netbox.plugins.inventory.netbox import NetBoxInventory2
from nornir.core.inventory import Inventory, Group

from dotty_dict import dotty

import ruamel.yaml


class CustomNetboxInventory(NetBoxInventory2):

    def __init__(self, config_file, **kwargs):
        with open(config_file, 'r') as f:
            config = ruamel.yaml.safe_load(f)
        super().__init__(
            nb_url=config['nb_url'],
            nb_token=config.get('nb_token'),
            filter_parameters=config['filter_by']
        )
        self.group_by = config['group_by']

    def load(self) -> Inventory:
        ''' Override the base load method so we can add group data
            to our hosts
        '''
        inventory: Inventory = super().load()
        for host in inventory.hosts.values():
            for item in self.group_by:
                dot = dotty(host.data)
                try:
                    group_name = dot.get(item)
                except Exception:
                    group_name = None
                if group_name:
                    host.groups.append(Group(name=group_name))
                    inventory.groups.setdefault(group_name, {})
        return inventory
```

We first import the Nornir inventory plugin. Normally you would pass the `NetboxInventory2` class directly into the `InitNornir` function as a custom inventory plugin but since we want to extend its functionality, we define our own `CustomNetboxInvetory` plugin that inherits from the base plugin. This class contructor reads the YAML configuration file and passes the base parameters over to the parent.

The Nornir inventory is returned in the `load()` method, which we override in our custom class to add the desired grouping functionality. First we call the parent `load()` method to get the base (ungrouped) inventory. Each host in the inventory has a data dict that stores host vars or attributes and a groups list that contains all the groups the host is part of. We iterate over each host, search for a grouping attribute within the host data and if a match is found, add that group to the group list of that host as well as to the inventory group dict. Note that we leave the group data empty for this use case. Using the `dotty-dict` library makes it easy to search for deeply nested dictionary data within the host object *( for e.g , given a= {'foo': {'bar': 'baz'}}, you can call a['foo.bar'] to get the value 'baz' )*. As a result given the below JSON data from Netbox, we can easily match nested attributes (such as `role.slug`) and add those to the groups.

```json
{
    "count": 1,
    "next": null,
    "previous": null,
    "results": [
        {
            "id": 100,
            "name": "dev1",
            "device_role": {
                "id": 58,
                "name": "leaf",
                "slug": "leaf"
            },
            "platform": {
                "id": 3,
                "name": "Junos",
                "slug": "junos"
            },
            "site": {
                "id": 1,
                "name": "site1",
                "slug": "site1"
            }
        }
    ]
}
```

Note that we are performing only minimal error checking or exception handling in the above code, which is something you should do more of in production code. We can then use our custom inventory class directly when isntantiating the Nornir object. From a Python shell:

```python
>>> from nornir import InitNornir
>> nr = InitNornir(
    runner={
        "plugin": "threaded",
        "options": {
            "num_workers": 100,
        },
    },
    inventory={
        'plugin': 'custom_inventory.CustomNetboxInventory',
        'options': {
            'config_file': 'my_config.yaml',
         }
    }
)
PluginNotRegistered                       Traceback (most recent call last)
<ipython-input-3-1fe3436d24eb> in <module>
...
PluginNotRegistered: plugin 'custom_inventory.CustomNetboxInventory' is not registered
```

If we try to use it the old fashioned way, we hit an exception. The reason for the above exception is detailed in the Nornir docs:

```
Starting with nornir3 some plugins need to be registered in order to be used. In particular:

inventory plugins
transform functions
connection plugins
runners
```

Thankfully, we can do it programatically using `InventoryPluginRegister`.

```python
>>> from nornir.core.plugins.inventory import InventoryPluginRegister
>>> from custom_inventory import CustomNetboxInventory
>>> InventoryPluginRegister.register("custom-inventory", CustomNetboxInventory)
>>> nr = InitNornir(
    runner={
        "plugin": "threaded",
        "options": {
            "num_workers": 100,
        },
    },
    inventory={
        'plugin': 'custom-inventory',
        'options': {
            'config_file': 'my_config.yaml',
         }
    }
)
>>> len(nr.inventory.hosts)
12

>>> nr.inventory.hosts.keys()
dict_keys(['core1', 'core2', 'leaf1', 'leaf2', 'leaf3', 'leaf4', 'router1', 'router2', 'spine1', 'spine2', 'spine3', 'spine4'])

>>> nr.inventory.groups.keys()
dict_keys(['core1', 'core-switch', 'lab1', 'core2', 'leaf1', 'leaf-switch', 'cisco-ios', 'leaf2', 'leaf3', 'leaf4', 'router1', 'router', 'juniper-junos', 'router2', 'spine1', 'spine-switch', 'arista-eos', 'spine2', 'spine3', 'spine4'])

>>> nr.inventory.hosts['spine1'].groups
['spine1', 'spine-switch', 'lab1', 'arista-eos']
```

You must be wondering how we actually have 12 devices in the inventory. If you noticed above in our YAML configuration, I am using the Netbox URL `https://netbox.live`. This is not a random URL , rather, a fully functional Netbox demo website ( complete with some test data ) that is hosted online by some good samaritans. This is awesome, since it allows us to test out our inventory code without having to run Netbox locally. From the above output, you can see that we have successfully grouped our devices by the parameters specified in the config. For example. the device `spine1` is now part of the groups `['spine1', 'spine-switch', 'lab1', 'arista-eos']` ( NB: We are also grouping by the device name itself, the significance of this will be apparant in the next section.)


## Adding run-time filtering

With the appropriate grouping done, we are now well equipped to run tasks on any specific group of devices that we choose. To illustrate how useful this is in practice, assume we have a simple task defined that simply prints out the IPv4 address of the given host: 

<span style="font-family:monospace">simple_task.py:</span>
```python

from nornir.core.task import Task, Result

def simple_task(task: Task) -> Result:
    host_data = task.host.data
    v4Addr = host_data['primary_ip4']['address']
    msg = f'Host {task.host.name} has ipv4 address {v4Addr}'
    return Result(host=task.host, result=msg)
```

Recall that a task is nothing but a function that takes in a `Task` object and returns a `Result`. Now if we wanted to run this task against every host, we would do:

```
>>> from simple_task import simple_task
>>> from nornir_utils.plugins.functions import print_result
>>> print_result(nr.run(task=simple_task))
simple_task*********************************************************************
* core1 ** changed : False *****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host core1 has ipv4 address 172.17.2.1/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* core2 ** changed : False *****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host core2 has ipv4 address 172.17.2.2/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* leaf1 ** changed : False *****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host leaf1 has ipv4 address 172.17.4.1/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* leaf2 ** changed : False *****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host leaf2 has ipv4 address 172.17.4.2/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* leaf3 ** changed : False *****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host leaf3 has ipv4 address 172.17.4.3/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* leaf4 ** changed : False *****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host leaf4 has ipv4 address 172.17.4.4/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* router1 ** changed : False ***************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host router1 has ipv4 address 172.17.1.1/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* router2 ** changed : False ***************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host router2 has ipv4 address 172.17.1.2/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* spine1 ** changed : False ****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host spine1 has ipv4 address 172.17.3.1/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* spine2 ** changed : False ****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host spine2 has ipv4 address 172.17.3.2/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* spine3 ** changed : False ****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host spine3 has ipv4 address 172.17.3.3/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* spine4 ** changed : False ****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host spine4 has ipv4 address 172.17.3.4/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

But how do we restrict the task to run only against say, spine-switches ? Or what if we only wanted to run against spines that are only Cisco-IOS ? Better yet, how do we only run against only non Cisco-IOS spines that are in site XYZ ?? The answer of course lies in Nornir's [F Object](https://nornir.readthedocs.io/en/latest/tutorial/inventory.html#Advanced-filtering), which is an advanced inventory filtering mechanism that ships with Nornir.

In a real world application or framework where you have a collection of tasks or scripts, you would normally want to allow the end user to express these constraints in a user-friendly way, ideally using a CLI argument. One such way is by using query expressions as shown below:

```
leaf1

spine-switch,leaf-switch&cisco-ios

spine-switch&SiteA&!cisco-ios
```

This may seem like a simpler version of Ansible's inventory patterns and it indeed is. For a majority of use cases, something as simple as this should suffice. Lets see how we can build an inventory filter using these queries.

The simplest use case of filtering by a single attribute can be implemented quite simply by using `nr.filter(F(k=v))`. However, a broader implementation that could also match on a group name could be achieved by using `nr.filter(F(k__contains=v))`. In both cases, k is either a simple or a nested device attribute or the word `groups` and v is the attribute value or the group name. The `__contains` filter normally serves to find an item inside an iterable, but also works for strings since a string itself is also an iterable in Python. The F object is quite handy in that it also provides the AND (&) , OR (|) and NEGTATE (~) operators so we can now mix and match:

```python
# match hosts that belong to site A and are spines
nr.filter(F(site__name__contains='A') & F(device_role___name__contains='spine'))  # based on actual netbox JSON data
** OR **
nr.filter(F(groups__contains='A') & F(groups__contains='spine'))  # since we grouped by site and device_role

# match hosts that is either a spine or a leaf
nr.filter(F(groups__contains='spine') | F(groups__contains='leaf'))

# match hosts that are belong to site A but are NOT spines
nr.filter(F(groups__contains='A') & ~F(groups__contains='spines'))
```

Note that using device attributes requires you to know the structure of the JSON data returned from netbox in order to create the correct nested hierarchy (separated by double underscores). Since we already grouped hosts based on some of these interesting attributes before, we can simply use the `groups__contains` filter parameter instead and make things simpler. Given a query string such as `spine-switch,leaf-switch&cisco-ios`, we can split the terms up using boolean operators to get:
```
(spine_switch OR leaf_switch) AND (cisco-ios)
```
Then its a simple matter of constructing an F object for each term and using Python's Boolean operators to reduce them down. i.e
```python
(F(groups__contains='spine_switch') | F(groups__contains='leaf-switch')) & F(groups__contains='cisco-ios')
```

We can now write a function that can accept a worded query as indicated above and return the filtered inventory based on this query using the built-in `or_`, `and_` operators and the `reduce` function:

```python
from nornir.core import Nornir
from nornir.core.filter import F
from operator import and_, or_
from functools import reduce

def filter_inventory(nr: Nornir, query_string: str):
    filters = []
    # split query into terms
    for term in query_string.split('&'):
        # a 'not' operation is indicated by a preceding !
        negate = True if term.startswith('!') else False
        # split up all values separated by commas - these will be OR'd
        values = term[1:].split(',') if negate else term.split(',')
        sub_filters = []
        for val in values:
            f_object = ~F(groups__contains=val) if negate else F(groups__contains=val)
            sub_filters.append(f_object)
        # reduce the list of F objects using boolean OR 
        ord_f_object = reduce(or_, sub_filters)
        filters.append(ord_f_object)
    # finally, reduce the query terms using boolean AND and filter by the final F object expression
    return nr.filter(reduce(and_, filters))
```

In this function, we first split up the query string into different terms (splitting on '&').

Next we perform a boolean OR on all values in a given term that are separated by commas. This is achieved using Python's `or_` which lets you write boolean operands as functions. The benefit of this is that you can then use the `reduce` function to reduce a list of F objects down to a single object using OR.

Finally, we reduce the individual query terms using boolean AND and filter the inventory using the final F object.

This method relies on accurate grouping of hosts for all possible filter combinations. Because we've made this configurable using a YAML file, it is easy to add more groupings in the future if needed. We can test the above filter using the sample data from netbox.live:

```
>>> from simple_task import simple_task
>>> from nornir_utils.plugins.functions import print_result
>>>
>>> filtered = filter_inventory(nr, 'spine-switch,leaf-switch&cisco-ios')
>>> filtered.inventory.hosts.keys()
dict_keys(['leaf1', 'leaf2', 'leaf3', 'leaf4'])
>>>
>>> print_result(filtered.run(task=simple_task))
simple_task*********************************************************************
* leaf1 ** changed : False *****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host leaf1 has ipv4 address 172.17.4.1/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* leaf2 ** changed : False *****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host leaf2 has ipv4 address 172.17.4.2/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* leaf3 ** changed : False *****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host leaf3 has ipv4 address 172.17.4.3/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* leaf4 ** changed : False *****************************************************
vvvv simple_task ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Host leaf4 has ipv4 address 172.17.4.4/32
^^^^ END simple_task ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

>>> filtered = filter_inventory(nr, 'spine-switch,leaf-switch&!cisco-ios')
>>> filtered.inventory.hosts.keys()
dict_keys(['spine1', 'spine2', 'spine3', 'spine4'])
```

As can be seen, only a subset of the inventory matching our filter is now returned. One can easily adapt this concept into a CLI tool or framework that performs tasks against specific groups of devices as specified by the user. Since everything is pure Python, one can also easily extend the functionality in any way imaginable - for example, to create nested groups, adding group specific data and so on. This was just one quick example of how we can extend the out-of-the-box functionality of Nornir to create custom workflows and patterns.

### References

- Nornir Inventories: https://nornir.readthedocs.io/en/latest/tutorial/inventory.html
- Live Netbox demo: https://netbox.live
- Ansible Pattern matching: https://docs.ansible.com/ansible/latest/user_guide/intro_patterns.html
