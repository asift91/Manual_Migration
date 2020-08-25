
**_Virtual Network GateWay Creating & Connecting through private IP_**

This document explains how to setup Virtual Network Gateway in Azure.

-	Create a virtual network gate in Azure Portal way by the following steps:

	-	Search for Virtual network gate way and click on that.
	-	Click on create and it will open a window.
	-	Fill all the details like Name, Region, Gateway type, sku, vnet and keep the rest to default values.
	-	Click on create then it will start deploying.
-	Create Virtual Network gateway by Azure CLI command
	```
	az network vnet-gateway create -g MyResourceGroup -n MyVnetGateway --public-ip-address MyGatewayIp --vnet MyVnet --gateway-type Vpn --sku VpnGw1 --vpn-type RouteBased --no-wait
	```
-	


