# S2S VPN a Cisco CSR and Active/Active Azure VPN Gateway
This lab guide illustrates how to build a basic IPSEC/IKEv2 VPN tunnel between a CSR and active/active Azure VPN gateways. Each tunnel is using BGP plus VTIs for tunnel termination. Traffic sourcing from either side of the tunnel will ECMP load share. The CSR configs are broken out for each tunnel and can be combined as needed. The lab includes 2 test Linux VMs for additional verification and testing. On prem is simulated with a VNET so no hardware is needed. All username/password are azureuser/Msft123Msft123.
# Base Topology

![alt text](https://github.com/jwrightazure/lab/blob/master/images/csr-topo.PNG)

![alt text](https://github.com/jwrightazure/lab/blob/master/images/csr-bgp.PNG)

**Before deploying CSR in the next step, you may have to accept license agreement unless you have used it before. You can accomplish this through deploying a CSR in the portal or Powershell commands via Cloudshell**
<pre lang="...">
Sample Powershell:
Get-AzureRmMarketplaceTerms -Publisher "Cisco" -Product "cisco-csr-1000v" -Name "17_2_1-byol"
Get-AzureRmMarketplaceTerms -Publisher "Cisco" -Product "cisco-csr-1000v" -Name "17_2_1-byol" | Set-AzureRmMarketplaceTerms -Accept
</pre>

**Build Resource Groups, VNET and Subnets for the hub and onprem networks. The Azure VPN gateways will take about 20 minutes to deploy.**
<pre lang="...">
RESOURCE_GROUP_NAME="VPN"
AZURE_LOCATION="eastus2"

az group create --name ${RESOURCE_GROUP_NAME} --location ${AZURE_LOCATION}
az network vnet create --resource-group ${RESOURCE_GROUP_NAME} --name VPNhub --location ${AZURE_LOCATION} --address-prefixes 10.0.0.0/16 --subnet-name VPNhubVM --subnet-prefix 10.0.10.0/24
az network vnet subnet create --address-prefix 10.0.0.0/24 --name GatewaySubnet --resource-group ${RESOURCE_GROUP_NAME} --vnet-name VPNhub
az network vnet create --resource-group ${RESOURCE_GROUP_NAME} --name onprem --location ${AZURE_LOCATION} --address-prefixes 10.1.0.0/16 --subnet-name onpremVM --subnet-prefix 10.1.10.0/24
az network vnet subnet create --address-prefix 10.1.0.0/24 --name zeronet --resource-group ${RESOURCE_GROUP_NAME} --vnet-name onprem
az network vnet subnet create --address-prefix 10.1.1.0/24 --name onenet --resource-group ${RESOURCE_GROUP_NAME} --vnet-name onprem
az network public-ip create --name Azure-VNGpubip1 --resource-group ${RESOURCE_GROUP_NAME} --allocation-method Dynamic
az network public-ip create --name Azure-VNGpubip2 --resource-group ${RESOURCE_GROUP_NAME} --allocation-method Dynamic
az network vnet-gateway create --name Azure-VNG --public-ip-address Azure-VNGpubip1 Azure-VNGpubip2 --resource-group ${RESOURCE_GROUP_NAME} --vnet VPNhub --gateway-type Vpn --vpn-type RouteBased --sku VpnGw1 --asn 65001 --no-wait 
az network public-ip create --name CSRPublicIP --resource-group ${RESOURCE_GROUP_NAME} --idle-timeout 30 --allocation-method Static
az network nic create --name CSROutsideInterface -g ${RESOURCE_GROUP_NAME} --subnet zeronet --vnet onprem --public-ip-address CSRPublicIP --ip-forwarding true --private-ip-address 10.1.0.4
az network nic create --name CSRInsideInterface -g ${RESOURCE_GROUP_NAME} --subnet onenet --vnet onprem --ip-forwarding true --private-ip-address 10.1.1.4
az vm create --resource-group ${RESOURCE_GROUP_NAME} --location ${AZURE_LOCATION} --name CSR --size Standard_DS3_v2 --nics CSROutsideInterface CSRInsideInterface --image cisco:cisco-csr-1000v:17_2_1-byol:17.2.120200508 --admin-username azureuser --admin-password Msft123Msft123 --no-wait 
az network public-ip create --name VPNhubVM --resource-group ${RESOURCE_GROUP_NAME} --location ${AZURE_LOCATION} --allocation-method Dynamic
az network nic create --resource-group ${RESOURCE_GROUP_NAME} -n VPNVMNIC --location ${AZURE_LOCATION} --subnet VPNhubVM --private-ip-address 10.0.10.10 --vnet-name VPNhub --public-ip-address VPNhubVM --ip-forwarding true
az vm create -n VPNVM -g ${RESOURCE_GROUP_NAME} --image UbuntuLTS --admin-username azureuser --admin-password Msft123Msft123 --nics VPNVMNIC --no-wait 
az network public-ip create --name onpremVM --resource-group ${RESOURCE_GROUP_NAME} --location ${AZURE_LOCATION} --allocation-method Dynamic
az network nic create --resource-group ${RESOURCE_GROUP_NAME} -n onpremVMNIC --location ${AZURE_LOCATION} --subnet onpremVM --private-ip-address 10.1.10.10 --vnet-name onprem --public-ip-address onpremVM --ip-forwarding true
az vm create -n onpremVM -g ${RESOURCE_GROUP_NAME} --image UbuntuLTS --admin-username azureuser --admin-password Msft123Msft123 --nics onpremVMNIC --no-wait 
az network route-table create --name vm-rt --resource-group ${RESOURCE_GROUP_NAME}
az network route-table route create --name vm-rt --resource-group ${RESOURCE_GROUP_NAME} --route-table-name vm-rt --address-prefix 10.0.0.0/16 --next-hop-type VirtualAppliance --next-hop-ip-address 10.1.1.4
az network vnet subnet update --name onpremVM --vnet-name onprem --resource-group ${RESOURCE_GROUP_NAME} --route-table vm-rt
</pre>

**Document the public IPs for the test VMs, CSR and Azure VPN gateways and copy them to notepad. If the public IPs for the Azure VPN gateways return "none", do not continue. The gateways are deployed when they return a public IP.**
<pre lang="...">
az network public-ip show --resource-group ${RESOURCE_GROUP_NAME} --name Azure-VNGpubip1 --query [ipAddress] --output tsv
az network public-ip show --resource-group ${RESOURCE_GROUP_NAME} --name Azure-VNGpubip2 --query [ipAddress] --output tsv
az network public-ip show --resource-group ${RESOURCE_GROUP_NAME} --name CSRPublicIP --query [ipAddress] --output tsv
az network public-ip show --resource-group ${RESOURCE_GROUP_NAME} --name VPNhubVM --query [ipAddress] --output tsv
az network public-ip show --resource-group ${RESOURCE_GROUP_NAME} --name onpremVM --query [ipAddress] --output tsv
</pre>

**Document BGP information for the VPN gateway.**
<pre lang="...">
az network vnet-gateway list --query [].[name,bgpSettings.asn,bgpSettings.bgpPeeringAddress] -o table --resource-group ${RESOURCE_GROUP_NAME}
</pre>

**Build the tunnel connection to the CSR. Replace "CSRPublicIP" with the public IP you copied to notepad. Note- only the /32 of the CSR loopback is allowed across the tunnel. All traffic will be allowed across the tunnel.**
<pre lang="...">
az network local-gateway create --gateway-ip-address "CSRPublicIP" --name to-onprem --resource-group ${RESOURCE_GROUP_NAME} --local-address-prefixes 192.168.1.1/32 --asn 65002 --bgp-peering-address 192.168.1.1
az network vpn-connection create --name to-onprem --resource-group ${RESOURCE_GROUP_NAME} --vnet-gateway1 Azure-VNG -l ${AZURE_LOCATION} --shared-key Msft123Msft123 --local-gateway2 to-onprem --enable-bgp
</pre>

**SSH to the CSR and paste in the below config. Make sure to change "Azure-VNGpubip1" and "Azure-VNGpubip2" . After this step, the test VMs will be able to reach each other.**
<pre lang="...">
ip route 10.1.10.0 255.255.255.0 10.1.1.1
crypto ikev2 proposal Azure-Ikev2-Proposal 
 encryption aes-cbc-256
 integrity sha1
 group 2
!
crypto ikev2 policy Azure-Ikev2-Policy 
 match address local 10.1.0.4
 proposal Azure-Ikev2-Proposal
!         
crypto ikev2 keyring to-onprem-keyring
 peer "Azure-VNGpubip1"
  address "Azure-VNGpubip1"
  pre-shared-key Msft123Msft123
 !
 peer "Azure-VNGpubip2"
  address "Azure-VNGpubip2"
  pre-shared-key Msft123Msft123
 !
crypto ikev2 profile Azure-Ikev2-Profile
 match address local 10.1.0.4
 match identity remote address "Azure-VNGpubip1" 255.255.255.255 
 match identity remote address "Azure-VNGpubip2" 255.255.255.255 
 authentication remote pre-share
 authentication local pre-share
 keyring local to-onprem-keyring
 lifetime 28800
 dpd 10 5 on-demand

crypto ipsec transform-set to-Azure-TransformSet esp-gcm 256 
 mode tunnel
!
!
crypto ipsec profile to-Azure-IPsecProfile
 set transform-set to-Azure-TransformSet 
 set ikev2-profile Azure-Ikev2-Profile
!
interface Loopback11
 ip address 192.168.1.1 255.255.255.255
!
interface Tunnel11
 ip address 192.168.2.1 255.255.255.255
 ip tcp adjust-mss 1350
 tunnel source 10.1.0.4
 tunnel mode ipsec ipv4
 tunnel destination "Azure-VNGpubip1"
 tunnel protection ipsec profile to-Azure-IPsecProfile
!
interface Tunnel12
 ip address 192.168.3.1 255.255.255.255
 ip tcp adjust-mss 1350
 tunnel source 10.1.0.4
 tunnel mode ipsec ipv4
 tunnel destination "Azure-VNGpubip2"
 tunnel protection ipsec profile to-Azure-IPsecProfile

router bgp 65002
 bgp router-id 192.168.1.1
 bgp log-neighbor-changes
 neighbor 10.0.0.4 remote-as 65001
 neighbor 10.0.0.4 ebgp-multihop 255
 neighbor 10.0.0.4 update-source Loopback11
 neighbor 10.0.0.5 remote-as 65001
 neighbor 10.0.0.5 ebgp-multihop 255
 neighbor 10.0.0.5 update-source Loopback11
 !
 address-family ipv4
  network 10.1.10.0 mask 255.255.255.0
  neighbor 10.0.0.4 activate
  neighbor 10.0.0.5 activate
  maximum-paths 2
 exit-address-family

ip route 10.0.0.4 255.255.255.255 Tunnel11
ip route 10.0.0.5 255.255.255.255 Tunnel12
</pre>

**Verify tunnel connections**

*Check if the state of your IKE Security Association is READY*
<pre lang="...">
CSR#show crypto ike sa
 IPv4 Crypto IKEv2  SA

Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         10.1.0.4/4500         52.254.49.162/4500    none/none            READY
      Encr: AES-CBC, keysize: 256, PRF: SHA1, Hash: SHA96, DH Grp:2, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 28800/254 sec

Tunnel-id Local                 Remote                fvrf/ivrf            Status
3         10.1.0.4/4500         52.254.49.170/4500    none/none            READY
      Encr: AES-CBC, keysize: 256, PRF: SHA1, Hash: SHA96, DH Grp:2, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 28800/592 sec

 IPv6 Crypto IKEv2  SA
</pre>

*Explore IPsec SA for encrypted/decrypted packages*
<pre lang="...">
CSR#show crypto ipsec sa | i pkt
    #pkts encaps: 47, #pkts encrypt: 47, #pkts digest: 47
    #pkts decaps: 68, #pkts decrypt: 68, #pkts verify: 68
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #pkts encaps: 53, #pkts encrypt: 53, #pkts digest: 53
    #pkts decaps: 35, #pkts decrypt: 35, #pkts verify: 35
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
</pre>

*Check BGP table*
<pre lang="...">
CSR#show ip bgp
BGP table version is 7, local router ID is 192.168.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *m   10.0.0.0/16      10.0.0.4                               0 65001 i
 *>                    10.0.0.5                               0 65001 i
 *>   10.1.10.0/24     10.1.1.1                 0         32768 i
</pre>