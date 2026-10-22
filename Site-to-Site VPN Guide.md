>[!danger] AI formatted this guide

>[!hint] Redacted :)


Source: https://tailscale.com/docs/features/site-to-site#visual-editor

# Prerequisites
>[!warning] `Requirements for site-to-site networking`
>1. The subnets must **not** have identical [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) ranges. If you need multiple subnet routers advertising the exact same subnet (for example, two routers both advertising `10.0.0.0/24`), use [high availability](https://tailscale.com/docs/how-to/set-up-high-availability) or [4via6 subnet routing](https://tailscale.com/docs/features/subnet-routers/4via6-subnets).
>2. Both subnet routers must use a Linux-based operating system.

---
# Create a site-to-site connection
---
## **Step 1: Select a subnet router**

The first step is to select a device within the subnet to act as the designated subnet router. This device must use a Linux-based operating system.

---
## **Step 2: Configure the subnet router**

After selecting a device in the subnet to function as the subnet router, you must configure it to act as a subnet router. 
- Configuring the subnet router involves [installing the Tailscale client](https://tailscale.com/docs/install), enabling IP address forwarding, starting the Tailscale client with the correct configuration options, and configuring the device's `iptables` settings.

>[!hint] You should clamp the `MSS` to the `MTU`
>`MSS` : Maximum Segment Size -> Largest segment of `TCP data` (payload) that a device can receive in a single TCP packet
>`MTU` : Maximum Transmission Unit -> Largest size of an IP packet (including headers and data) that can be transmitted over a network link
>
>**If either are larger than the other `fragmentation` will occur lowering the speed.**

Configuring a device as a subnet router involves specifying the: 
1. Subnet routes to advertise 
	1. `--advertise-routes=<CIDR>`
2. Disabling source network address translation (SNAT) 
	1. `--snat-subnet-routes=false` (Linux only)
3. Enabling the device to accept routes. 
	1. `--accept-routes`

```json
//Initial Setup
tailscale up --advertise-routes=<CIDR> --snat-subnet-routes=false --accept-routes

//Modify Setup
tailscale set --snat-subnet-routes=false
```

You can use the [`tailscale up`](https://tailscale.com/docs/reference/tailscale-cli#up) or [`tailscale set`](https://tailscale.com/docs/reference/tailscale-cli#set) CLI commands to set these configuration options.

---
## **Step 3: Approve the subnet router**

After selecting and configuring the subnet router, you must approve it from the Machines page of the admin console. If you used an [auto approver](https://tailscale.com/docs/reference/syntax/policy-file#autoapprovers) policy in the tailnet policy file for the device, you don't need to approve the subnet router manually.

1. Open the [Machines](https://console.tailscale.com/admin/machines) page of the admin console.
2. Locate the subnet router devices by locating the **Subnets** badge or using the [`property:subnet`](https://console.tailscale.com/admin/machines?q=property%3Asubnet) filter.
3. For each subnet router:
    1. Select the `...` menu > **Edit route settings**.
    2. Approve the device.

>[!hint] You might prefer to [disable key expiry](https://tailscale.com/docs/features/access-control/key-expiry) on your subnet routers to avoid having to periodically reauthenticate. If you are using [tags](https://tailscale.com/docs/features/tags), [key expiry is disabled by default](https://tailscale.com/docs/features/tags#key-expiry).

---
## **Step 4: Update tailnet access control policies**
You must update the [access control policies](https://tailscale.com/docs/features/access-control) for your tailnet to allow communication between the subnets by creating access rules ([grants](https://tailscale.com/docs/features/access-control/grants)) to permit the subnets to connect.

You can do this from the [Access controls](https://console.tailscale.com/admin/acls) page of the admin console using either the [visual editor](https://tailscale.com/docs/features/visual-editor) or the JSON editor.

The following examples permit all traffic between the subnets.

>[!DANGER] I found that if you click save at this point, and you are using the tailscale network to connect to your homelab, the connection will break... 
>**Solution: Wait until Step `<?>` to save**


#### [JSON editor](https://tailscale.com/docs/features/site-to-site#json-editor)

To use the JSON editor to create access rules between the subnets, you must create two access rules: one for each direction of traffic between the subnets.

1. Open the [Access controls](https://console.tailscale.com/admin/acls/file) page of the admin console.
2. Select the **JSON editor** tab to update the tailnet policy file using JSON.
3. Locate the `grants` section of the policy file.
4. Add two access rules to the `grants` section of the policy file: one for each direction of traffic between the subnets.

The end result should look similar to the following example, which permits all traffic between two subnets. Replace `<first-subnet-CIDR>` and `<second-subnet-CIDR>` with the actual CIDR ranges of the subnets.

```json
// Default Config
// Comment out to stop using
{"src": ["*"], "dst": ["*"], "ip": ["*"]},

// Connect subnets, 10.0.0.0/24 to 192.168.1.0/24
			{
				"src": ["10.0.0.0/24"],
				"dst": ["192.168.1.0/24"],
				"ip":  ["*"],
			},
			{
				"src": ["192.168.1.0/24"],
				"dst": ["10.0.0.0/24"],
				"ip":  ["*"],
			},
```

---
## **Step 5: Configure the other subnet devices**

After configuring and approving the subnet router, make sure the devices in the subnet use the designated subnet router for the correct routes.

>[!danger] Create your own commands!
>The `ip route` commands do not persist after rebooting. You must run them again after each reboot. Depending on your setup, you can make the route settings persistent by adding them to your network manager or [`netplan`](https://netplan.readthedocs.io/en/stable/) configuration. Alternatively, you can manage route settings with a DHCP server on your network.

---
# Opnsense Config

| OPNsense             | Route (Network Address)     | Gateway (Tailscale Host) |
| -------------------- | --------------------------- | ------------------------ |
| Gnome (10.0.0.1)     | 100.64.0.0/10 (*Tailscale*) | 10.0.0.101               |
| Gnome (10.0.0.1)     | 192.168.1.0/24 (*Zaysec*)   | 10.0.0.101               |
| ZaySec (192.168.1.1) | 100.64.0.0/10 (*Tailscale*) | 192.168.1.102            |
| ZaySec (192.168.1.1) | 10.0.0.0/24 (*Gnome*)       | 192.168.1.102            |

## Add Gateway
1. System -> Gateway -> Configuration
	1. Click the plus button on the right
	2. Name: `Tailscale site2site`
	3. Interface: *Lan or something similar*
	4. Click Save and close

## Add Routes
1. System -> Routes -> Configuration
	1. *Tailscale Route*
		1. Network Address: `100.64.0.0/10`
		2. Gateway: `Tailscale site2site`
		3. Description: `Tailscale Route`
	2. *Gnome Route*
		1. Network Address: `10.0.0.0/24`
		2. Gateway: `Tailscale site2site`
		3. Description: `Gnome Route`

## Bypass Firewall
1. Firewall -> Settings -> Advanced
	1. Enabled `Bypass firewall rules for traffic on the same interface`
