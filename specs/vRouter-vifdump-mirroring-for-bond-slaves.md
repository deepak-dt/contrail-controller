
#1. Introduction 
==================
This blueprint describes high level changes required to enhance Contrail vRouter ‘vifdump’ tool (like tcpdump) for analyzing packets on individual member links of a bond interface configured on a DPDK based vRouter.


#2. Problem Statement
==================
The Kernel level tools, like tcpdump, do not work when vRouter is deployed in DPDK mode, since the Linux network stack is bypassed. There exist already a tcpdump like command (i.e. vifdump), which works on the bond interface but it can’t be used to monitor its slave/member interfaces; so there is no way to debug each slave individually.


#3. Proposed solution
==================
List of changes in vRouter application to enhance ‘vifdump’ tool [MUA1] for monitoring slave interaces are described below.

##3.1 “vif --list” command output
--------
Enhance output of “vif --list” command to display slave interface attached to a bond interface. With slave interfaces also visible in command output, user will be able to specify a slave interface as interface to be monitored in ‘vifdump’ command.


##3.2 No change proposed for "vifdump"
--------
No change is proposed in ‘contrail-vrouter/utils/vifdump’ shell script for this enhancement. Current logic in this script shall work as it is even when monitored interface is a slave interface of a bond interface. 


##3.3  Transmitted Packet Monitoring
--------
In current handling of ‘vifdump’ command, Vif interface manager sends “vr_interface_req” sandesh message to vRouter to add the interface for monitoring, and vRouter DPDK driver then creates a mirror interface (i.e. KNI/tuntap) towards the kernel.

vRouter DPDK driver logic shall be modified to check if the monitored interface is a slave-interface. If not, then existing handling will be used, i.e. enable the monitoring for the interface by marking it in vr_dpdk.monitorings[] array and set its VIF_FLAG_MONITORED flag to true. If yes, a new logic shall be added, which is as follows.

Check whether mirroring is already active for the specified slave interface. If true, then return with error[MUA2] . If above check
returns false, then call “rte_eth_add_tx_callback” API of DPDK to register a call-back function for the specified slave interface. This call-back function shall be called by DPDK library whenever a packet is transmitted on the slave interface, and the callback function will perform mirroring of the packet data onto the KNI/Tuntap interface towards the kernel.

At the time of Tx packet processing—

a. As per existing handling vRouter application calls the rte_port_ethdev_writer_tx API of DPDK to transmit the packet over a bond interface. This API in turn calls rte_eth_tx_burst API for the bond interface. 

b. DPDK PMD bond driver then decides which slave interface to use for transmitting the package based on the configured bonding mode. Finally for the selected slave interface, rte_eth_tx_burst API is called. 

c. When rte_eth_tx_burst API is called for a slave interface which was being monitored then it would have a “pre_tx_burst_cbs” callback registered as explained above. The pre_tx_burst_cbs function will mirror the packet towards the KNI/Tuntap mirror interface in same way as currently done in “dpdk_if_tx” function.


##3.4   Received Packet Monitoring
--------
In current handling, vRouter application on receiving the packet from DPDK PMD bond driver (i.e. vr_dpdk_lcore_vroute and dpdk_if_rx functions) does the mirroring for the ports which are listed in vr_dpdk.monitorings[] array and have their VIF_FLAG_MONITORED flag set to true.

In the same functions, vRouter shall check if the interface for which packet is received from DPDK PMD bond driver, is configured for monitoring or not. If yes, then it shall do the mirroring of the received packet towards the KNI/Tuntap interface. 


##3.5 Alternatives Design
--------
In the Rx path, an alternative was to use “rte_eth_add_rx_callback” function to register a “post_rx_burst_cbs” for the monitored slave interface and perform the mirroring in this function. However, as Rx port (slave port) information is not modified by DPDK bonding driver while handling over collected packet to application, so it is possible to mirror received packet on the basis of slave
port on which it has been received without having to register a “post_rx_burst_cbs” for slave interface. 

Pros – aligned to the approach followed in transmit path

Cons – This avoids the overhead to have a call-back function getting called for each
receive operation.


##3.6 API schema changes 
--------
None


##3.7 User workflow impact
--------
a. Impact to “vif” command syntax:   
--------
    --list (Display all of the interfaces of which the vrouter is aware.)
         The output shall also display all the slave/member interfaces for a bond interface.    

    --Get (Displays a specific interface)
         The output shall also display all the slave/member interfaces for a bond interface.
          
          
b. No impact to “vifdump” command syntax:
---------
There is no change to the “vifdump” command syntax as such. However, now the user can also specify slave interface as vif to start or stop the monitoring on it.

    vRouter/DPDK vif tcmpdump script
    Usage:      
        vifdump [-i] <vif> [tcpdump arguments]        
            - to run tcpdump on a specified vif      

        vifdump stop <monitoring_if>        
            - to force stop and clean up the monitoring interface

    "Example:      
        vifdump -i vif0/1 -nvv"


##3.4 UI Changes
--------
None


##3.5 Operations and Notification
--------
Refer to section #3.3


#4. Implementation
==================
##4.1 Assignee(s)
--------
##4.2 Work Items
--------
a. Modification in vif interface manager to display slave interfaces in ‘vif -- list’ and ‘vif --get’ commands

b. No changes required in vifdump

c. Modifications in vRouter code for monitoring configuration; use “rte_eth_add_tx_callback” function to register a “pre_tx_burst_cbs” for the monitored slave interface whenever vif interface manager asks to add a monitoring interface for a slave interface of a bond. 

d. Modifications in vRouter code to perform the mirroring of Tx packet in the above call-back function towards the KNI/Tuntap interface

e. Modifications in vRouter code to perform the mirroring of Rx packet received from DPDK PMD bond driver towards the KNI/Tuntap interface

Refer to section 3 for more details 


#5. Performance and scaling impact
==================
vRouter performance with mirroring configured on slave interface shall be same as vRouter performance when mirroring is configured on normal VIF interface 


##5.1 API and control plane
--------
None 


##5.2 Forwarding Performance
--------
None 


#6. Upgrade
==================
None 


#7. Deprecations
==================
None 


#8. Dependencies
==================
None 


#9. Testing 
==================
##9.1 Unit tests
--------
##9.2 Dev tests
--------
##9.3 System tests
--------
Proposed test-setup
--------
1. To be tested with KVM hypervisor on compute node
2. To be tested with 10GBE NIC with PMD/Vhost interface on DPDK 
3. To be tested for different LAG distribution modes


#10. Documentation Impact
==================
Modifications would be needed to Contrail-feature-guide document to update the syntax and semantics for ‘vif’ and ‘vifdump’ commands


#11. References
==================
1. DPDK Link Bonding Poll Mode Driver Library
http://dpdk.org/doc/guides/prog_guide/link_bonding_poll_mode_drv_lib.html 

2. DPDK Kernel NIC Interface
http://dpdk.org/doc/guides-16.04/prog_guide/kernel_nic_interface.html 

3. Contrail Feature Guide
