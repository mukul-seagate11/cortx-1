
===============================
CORTX on Open Virtual Appliance
===============================
An Open Virtual Appliance (OVA) is a Virtual Machine (VM) image that consists of a pre-installed and pre-configured operating system as well as one or more applications packaged for easy deployment and testing.  This document describes how to use a CORTX OVA for the purposes of single-node CORTX testing.  The minimum recommended system should have at least 4 CPU cores, at least 8 GB of RAM, and at least 120 GB of local storage. Current version of the OVA requires DHCP server to assign IPs to all 3 network interfaces. For our Japanese community, this document has been translated and is available `here <https://qiita.com/Taroi_Japanista/items/0ac03f55dce3f7433adf>`_.

***********************
Recommended Hypervisors
***********************
All of the following hypervisors should work: `VMware ESX Server <https://www.vmware.com/products/esxi-and-esx.html>`_,
`VMware vSphere <https://www.vmware.com/products/vsphere.html>`_,
`VMware Fusion <https://www.vmware.com/products/fusion.html>`_,
`VMware Workstation <https://www.vmware.com/products/workstation-pro.html>`_, and
`Oracle VM VirtualBox <https://www.oracle.com/virtualization/>`_. 

**Important**: If you are running the VM in any of the VMWare hypervisors, it is not recommended to use VMware Tools, as CORTX may break due to kernel dependencies.  For the same reason, please do not update the operating system in the image as that also might cause it to fail.

**********
Procedure
**********
The procedure to install CORTX on OVA is mentioned below.

#. Download the `CORTX OVA <https://github.com/Seagate/cortx/releases/>`_ file from `our release page <https://github.com/Seagate/cortx/releases/latest>`_. This contains the virtual machine image.

#. Import the OVA image by referring to `these instructions <Importing_OVA_File.rst>`_. 

   - For VMware related troubleshooting, please refer to `VM Documents <https://docs.vmware.com/en/VMware-vSphere/index.html>`_. 
  
#. Open the VM console, and login with the below credentials.

   * Username: cortx 
   * Password: opensource!
  
#. **For Oracle VM VirtualBox Users ONLY**:
   
   .. raw:: html

    <details>
      <summary><a>Click here to view the steps.</a></summary>


   You need to change the Network Device Name from enp0s3, enp0s8, enp0s9 to ens32, ens33 and ens34:
   
      **Note:** 
      
         - The Network Device names may not be the exact same as listed above, for example, enp0s8, enp0s9, enp0s17 instead of enp0s3, enp0s8, enp0s9.
         - As the network is setup using DHCP, IP changes are bound to happen during restart, static IP configuration could be harder to maintain as it may not work for different VM with different network setups. 
   
   
   #. Use the following command to get your Network Device MAC address (Shown after **link/ether**)

      * **ip a l**

   #. Record the MAC addresses and go to the following directory:

      * **cd /etc/sysconfig/network-scripts/**
      * **vi ifcfg-ens32**
      * Add a new line under **BOOTPROTO=dhcp**
      * Add a new parameter with the MAC Address *HWADDR=<MAC-Address>*
      * Repeat the steps for enp0s8 and enp0s9 respectively
      * **vi ifcfg-ens33**
      * **vi ifcfg-ens34**

      Sample output **cat ifcfg-ens34**:
      
      ::
      
         DEVICE="ens34"
         USERCTL="no"
         TYPE="Ethernet"
         BOOTPROTO="dhcp"
         HWADDR=08:00:27:25:65:74
         ONBOOT="yes"
         PREFIX="24"
         PREDNS="no"
         DEFROUTE="no"
         NM_CONTROLLED="no"
         ZONE=trusted

   #. Poweroff the machine by using **Poweroff** command and start again.

   #. To verify the change in Network Device Name, run the following command:

      * **ip a l**

   #. The Date/Time would sometimes be incorrect as it uses local time as UTC

      * **date**

      If the time displayed is incorrect, use the following command to adjust time for timezone as necessary (Otherwise, you might face SSL certificate problems later). 
      * **date --set "[+/-]xhours [+/-]yminutes"**
      
      For instance if your timezone is `4:30:00` ahead of UTC, then run the following command in VM. Note the `-` before minutes as well. Similarly if your timezone is behind of UTC, use +ve hours and +ve minutes to make the adjustment.

      * **date --set "-4hours -30minutes"**
      
   .. raw:: html
   
      </details>

#. Start the CORTX services by running this bootstrap.sh script:
   
   ::
   
      sh /opt/seagate/cortx/provisioner/cli/virtual_appliance/bootstrap.sh
     
   Run the bootstrap script to ensure all the necessary services are operational.
   
#. **Before you begin:**
   
   - Ensure that you have configured your ipv4 network.

      - If you do not see an ipv4 network configured, you might need to change your virtual networking configuration using  `these instructions <troubleshoot_virtual_network.rst>`_.

   - From the Virtual Network Editor dialog, ensure you uncheck Automatic Settings and select the correct VMNet connection and NIC.

      - Once you select an NIC, ensure that you do not have conflicting NICs selected. 
      
#. (Optional) To configure the static IPs instead of DHCP:

   - For Management Network static IP, run the following command:

      ::

         # Set Management Network
         provisioner pillar_set "cluster/srvnode-1/network/mgmt/public_ip" \"<IP address for management network>\"
         provisioner pillar_set "cluster/srvnode-1/network/mgmt/netmask" \"<Netmask for management network>\"
         provisioner pillar_set "cluster/srvnode-1/network/mgmt/gateway" \"<IP address for management network gateway>\"
         salt-call state.apply components.system.network.mgmt.public

   - For Data Network static IP, run the following command:

      ::
      
         # Set Data Network
         provisioner pillar_set "cluster/srvnode-1/network/data/public_ip" \"<IP address for public network>\"
         provisioner pillar_set "cluster/srvnode-1/network/data/netmask" \"<Netmask for public data network>\"
         salt-call state.apply components.system.network.data.public

   **Note:** To verify the static IPs are configured, run the following command:

      ::

         cat /etc/sysconfig/network-scripts/ifcfg-ens32 |grep -Ei "ip|netmask|gateway"
         cat /etc/sysconfig/network-scripts/ifcfg-ens33 |grep -Ei "ip|netmask|gateway"

#. Check the health of CORTX using `hctl <https://github.com/Seagate/cortx/blob/main/doc/checking_health.rst>`_ by running this command
   
   ::
   
      cortx --help
   
#. Run **ip a l** and record the IP addresses of the following interfaces:

   * ens32 - Management IP
   * ens33 - Public data IP
   * ens34 - Private data IP (if present)

   .. image:: images/networks.png
   
   **Note:** To verify the interface zones, run the following command:

   :: 

      firewall-cmd --get-active-zones

   
   .. image:: images/FirewallZones.png
      
   
#. Using the management IP from the **ip a l** command, refer to these instructions to `configure the CORTX GUI <Preboarding_and_Onboarding.rst>`_. 

#. Now that you have the complete system up and running, using the data IP from the **ip a l** command, use these instructions `to test the system <Performing_IO_Operations_Using_S3Client.rst>`_ and observe activity in the GUI. For example, the below picture shows a CORTX dashboard after a user did an *S3 put* followed by an *S3 get*.

   .. image:: images/dashboard_read_write.png

#. Please use these instructions which describe how to use the `command line interface to query and monitor <https://github.com/Seagate/cortx/blob/main/doc/checking_health.rst>`_ the configuration, health, and activity of your CORTX system.

#. BOOM. You're all done and you're AWESOME. Thanks for checking out the CORTX system; we hope you liked it. Hopefully you'll stick around and participate in our community and help make it even better.

**Note:** The Lyve Pilot (LP) will be available in the future releases.
 
*************
Miscellaneous
*************

If you have a firewall between CORTX and the rest of your infrastructure, including but not limited to S3 clients, web browser, and so on, ensure that the ports mentioned below are open to provide access to CORTX.
  
+----------------------+-------------------+---------------------------------------------+
|    **Port number**   |   **Protocols**   |   **Destination network on CORTX**          |
+----------------------+-------------------+---------------------------------------------+
|          22          |        TCP        |           Management network                |
+----------------------+-------------------+---------------------------------------------+ 
|          53          |      TCP/UDP      | Management network and Public Data network  |
+----------------------+-------------------+---------------------------------------------+ 
|         123          |      TCP/UDP      |              Management network             |
+----------------------+-------------------+---------------------------------------------+
|         443          |       HTTPS       |             Public Data network             |
+----------------------+-------------------+---------------------------------------------+
|         9443         |       HTTPS       |              Public Data network            |
+----------------------+-------------------+---------------------------------------------+
|         28100        |   TCP (HTTPS)     |              Management network             |
+----------------------+-------------------+---------------------------------------------+

   
Known Issues
--------------

.. raw:: html

    <details>
   <summary><a>Click here to view the known issues.</a></summary>

#. After configuring the CORTX GUI, if any system alerts are displayed. You can ignore these system alerts. 

   .. image:: images/AlertsError.png

#. As the Consul service is not running, you will encounter the below depicted error.

   .. image:: images/consul.PNG

   **Workaround:** Run the followind mentioned commands:
   
   ::
   
    sed -i '11s/host:/host: 127.0.0.1/' /etc/csm/database.yaml
    
    systemctl restart csm_agent 


.. raw:: html
   
   </details>


Tested by:

- May 10, 2021: Shiji Zhang (shiji.zhang@tusimple.ai) using OVA release 1.0.4 on KVM 5.1

- Apr 30, 2021: Ashwini Borse (ashwini.borse@seagate.com) using OVA release 1.0.4 on Vsphere.

- Apr 12, 2021: Mukul Malhotra (mukul.malhotra@seagate.com) using OVA release 1.0.3 on MAC running VMWare Fusion 12.1.0.

- April 6, 2021: Harrison Seow (harrison.seow@seagate.com) using OVA release 1.0.3 on Windows 10 running VMware Workstation 16 Player.

- Mar 25, 2021: Mukul Malhotra (mukul.malhotra@seagate.com) using OVA release 1.0.3 on Windows 10 running Oracle VirtualBox & VMware Workstation 6.1.16.

- Mar 24, 2021:  Harrison Seow (harrison.seow@seagate.com) using OVA release 1.0.2 on Windows running Oracle VM VirtualBox 6.1.16.

- Mar 18, 2021: Jalen Kan (jalen.j.kan@seagate.com) using OVA release 1.0.2 on a Windows laptop running VMWare Workstation.

- Feb 4, 2021:  Tim Coulter (timothy.r.coulter@seagate.com) using OVA release 1.0.2 on MAC running VMWare Fusion 12.1.0

- Jan 13, 2021: Mayur Gupta (mayur.gupta@seagate.com) using OVA release 1.0.2 on a Windows laptop running VMWare Workstation.

- Jan 6, 2021: Patrick Hession (patrick.hession@seagate.com) using OVA release 1.0.2 on a Windows laptop running VMWare Workstation.

- Dec 10, 2020: Suprit Shinde (suprit.shinde@seagate.com) using OVA release 1.0.2 on a Windows laptop running VMWare Workstation.

- Nov 3, 2020: Justin Woo (justin.woo@seagate.com) using OVA release 1.0.2 on a Windows laptop running VMWare Workstation.

- Oct 26, 2020: Gregory Touretsky (gregory.touretsky@seagate.com) using OVA release 1.0.2 on a Windows laptop running VMWare Workstation.

- Oct 11, 2020: Saumya Sunder (saumya.sunder@seagate.com) using OVA release 1.0.2 on a Windows laptop running VMWare Workstation.

- Oct 5, 2020: Andriy Tkachuk (andriy.tkachuk@seagate.com) using OVA release 1.0.2 by running VMWare Fusion 11.

- Sep 18, 2020: Sarang Sawant (sarang.sawant@seagate.com) using OVA release 1.0.2 on a Windows laptop running VMWare Workstation.

- Sep 19, 2020: Divya Kachchwaha Kachchwaha (divya.kachhwaha@seagate.com) using OVA release 1.0.1 on a Windows laptop running VMWare Workstation.

- Sep 19, 2020: Venkataraman Padmanabhan (venkataraman.padmanabhan@seagate.com) using OVA release 1.0.0 and 1.0.1 on a Windows laptop running VMWare Workstation.

- Sep 12, 2020: Mukul Malhotra (mukul.malhotra@seagate.com) using OVA release 1.0.0 and 1.0.1 on a Windows laptop running VMWare Workstation.

- Sep 12, 2020: Puja Mudaliar (puja.mudaliar@seagate.com) using OVA release 1.0.0 on a Windows laptop running VMWare Workstation.

- Sep 12, 2020: Gaurav Chaudhari (gaurav.chaudhari@seagate.com) using OVA release 1.0.0 on a Windows laptop running VMWare Workstation.


