Managing Multiple Interfaces in Windows
==========================================

Let us suppose we have two interfaces with given default Gateways:

inetrfaces:

- 9: 192.168.10.1
- 13: 172.16.18.1

We want to create a routing plan where the interface 13 is used as default gateway unless an explicit route is set.

METHOD 1:
-------------

Changing Interface metrics. Since changing the default gateway is producing some unexpected errors.

1. Check the Interface metrics at the begining:

.. code-block:: bash


    ifIndex InterfaceAlias                  AddressFamily NlMtu(Bytes) InterfaceMetric Dhcp     ConnectionState PolicyStore
    ------- --------------                  ------------- ------------ --------------- ----     --------------- -----------
    13      Ethernet1                       IPv6                  1500              25 Enabled  Connected       ActiveStore
    9       Ethernet0                       IPv6                  1500              25 Enabled  Connected       ActiveStore
    1       Loopback Pseudo-Interface 1     IPv6            4294967295              75 Disabled Connected       ActiveStore
    13      Ethernet1                       IPv4                  1500              25 Disabled Connected       ActiveStore
    9       Ethernet0                       IPv4                  1500              25 Disabled Connected       ActiveStore
    1       Loopback Pseudo-Interface 1     IPv4            4294967295              75 Disabled Connected       ActiveStore



2. both the routes are default

3. change the metric of interface 9: 192.168.10.1 to 3 for now. I will explain why i chose this in the below steps.

.. code-block:: bash

    Powershell> Set-NetIPInterface -InterfaceIndex 9 -InterfaceMetric 3
    # Use the gui for now. If you are not much comfortable with Powershell.

4. change the metric of interface 13: 172.16.18.1 to 2 for now.

.. code-block:: bash
    
    Powershell> Set-NetIPInterface -InterfaceIndex 13 -InterfaceMetric 2
    # Use the gui for now. If you are not much comfortable with Powershell.

5. check the interface metrics using Get-NetIPInterface:

.. code-block:: bash
    
    Powershell> Get-NetIPInterface

    ifIndex InterfaceAlias                  AddressFamily NlMtu(Bytes) InterfaceMetric Dhcp     ConnectionState PolicyStore
    ------- --------------                  ------------- ------------ --------------- ----     --------------- -----------
    13      Ethernet1                       IPv6                  1500               2 Enabled  Connected       ActiveStore
    9       Ethernet0                       IPv6                  1500               3 Enabled  Connected       ActiveStore
    1       Loopback Pseudo-Interface 1     IPv6            4294967295              75 Disabled Connected       ActiveStore
    13      Ethernet1                       IPv4                  1500               2 Disabled Connected       ActiveStore
    9       Ethernet0                       IPv4                  1500               3 Disabled Connected       ActiveStore
    1       Loopback Pseudo-Interface 1     IPv4            4294967295              75 Disabled Connected       ActiveStore

6. Check the priority of the nic using route print

.. code-block:: bash

    ===========================================================================
    Interface List
    13...00 50 56 8e dc d0 ......Intel(R) 82574L Gigabit Network Connection #2
    9...00 50 56 8e 7c 8a ......Intel(R) 82574L Gigabit Network Connection
    1...........................Software Loopback Interface 1
    ===========================================================================

Here we can see that Interface 13 has more priority than Interface 9. This was what we wanted

7. Checking your routes:

.. code-block:: bash

    Powershell>tracert -d 8.8.8.8

    Tracing route to 8.8.8.8 over a maximum of 30 hops

    1     1 ms    <1 ms    <1 ms  public_ip_of_if_13
    2     1 ms     1 ms     1 ms  124.41.240.1
    3     1 ms     1 ms     1 ms  202.79.40.140
    4     1 ms     1 ms     1 ms  202.79.40.184
    5     2 ms     1 ms     3 ms  103.225.212.161
    6    44 ms    43 ms    43 ms  8.8.8.8

8. Adding Exceptions for the routes:

What if you want an exception for the route to go to the interface 9?

You can add any static routes that you put make it to metric below 1 so that they will be executed before the default interfaces

.. code-block:: bash

    Powershell> route add 8.8.8.8 mask 255.255.255.255 192.168.10.1 METRIC 1 IF 9 -p


.. code-block:: bash

    Powershell>tracert -d 8.8.8.8

    Tracing route to 8.8.8.8 over a maximum of 30 hops

    1     1 ms    <1 ms    <1 ms  public_ip_of_if_9
    2     1 ms     1 ms     1 ms  124.41.240.1
    3     1 ms     1 ms     1 ms  202.79.40.140
    4     1 ms     1 ms     1 ms  202.79.40.184
    5     2 ms     1 ms     3 ms  103.225.212.161
    6    44 ms    43 ms    43 ms  8.8.8.8

Here as you can see that if you tracert then even if the defaut interface is the 172.x.x.x it goes through 192.x.x.x i.e public_ip_of_if_9.

Because we put the above rule through here with higher priority.


METHOD 2:
---------

Changing Gateway metrics instead of Interface metrics.


1. Check the routes gateway and their metrics at the begining:

.. code-block:: bash
    
    Powershell> route print

    ==================================================================
    Persistent Routes:

    Network Destination        Netmask    Gateway Address  Metric
                0.0.0.0          0.0.0.0    192.168.10.1     Default
                0.0.0.0          0.0.0.0    172.16.18.1     Default



1. both the routes are default

2. change the metric of gateway 9: 192.168.10.1 to 3 for now. I will explain why i chose this in the below steps.

.. code-block:: bash
    
    Powershell> route change 0.0.0.0 mask 0.0.0.0 192.168.10.1 METRIC 3 IF 9 -p
    # Use the gui for now. If you are not much comfortable with Powershell.


3. change the metric of gateway 13: 172.16.18.1 to 2 for now.

.. code-block:: bash
    
    Powershell> route change 0.0.0.0 mask 0.0.0.0 172.16.18.1 METRIC 2 IF 13 -p
    # Use the gui for now. If you are not much comfortable with Powershell.

4. Use above steps to check the routes using netstat.

5. Adding Exceptions for the routes:

What if you want an exception for the route to go to the interface 9? Same as above.

You can add any static routes that you put make it to metric below 1 so that they will be executed before the default interfaces

.. code-block:: bash
    
    Powershell> route add 8.8.8.8 MASK 0.0.0.0  192.168.10.1  metric 1



RECOMMENDATIONS:
-----------------
IF you choose to choose to create an script, follow the following recommendations:

1. Create the powershell script instead of batch file.
2. Change the default routes (donot delete them and add again) using route change
3. Add -p at the end to make the routing persistant


REFERENCES:
------------

1. https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/route_ws2008
2. https://www.windowscentral.com/how-change-priority-order-network-adapters-windows-10
3. https://docs.microsoft.com/en-us/troubleshoot/windows-client/networking/default-gateway-route-not-appear
4. https://docs.microsoft.com/en-us/troubleshoot/windows-server/networking/automatic-metric-for-ipv4-routes
