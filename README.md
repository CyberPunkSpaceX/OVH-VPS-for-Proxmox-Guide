his is the complete technical blueprint for your Debian 13 Proxmox setup on OVH. This configuration ensures IP Symmetry (VM2 uses the Secondary IP for both in/out traffic), Strict Lockdown (Management ports are hidden), and Automation (FQDNs update automatically).
Phase 1: Host Network Configuration

Edit your Proxmox host network file to create the internal "lanes" for your VMs.

    Open the file: nano /etc/network/interfaces

    Append the following bridges (keep your existing eth0 and vmbr0 settings as they are):

Plaintext

# Bridge for Primary IP Traffic
auto vmbr1
iface vmbr1 inet static
        address 10.0.1.1
        netmask 255.255.255.0
        bridge-ports none
        bridge-stp off
        bridge-fd 0

# Bridge for Secondary IP Traffic
auto vmbr2
iface vmbr2 inet static
        address 10.0.2.1
        netmask 255.255.255.0
        bridge-ports none
        bridge-stp off
        bridge-fd 0

    Apply the changes: ifup vmbr1 && ifup vmbr2

Phase 2: Install Dependencies

Ensure your Debian 13 host has the tools required to run the automation script:
Bash

apt update
apt install cron dnsutils iptables-persistent -y

Phase 3: The Master Management Script

This script is your "control center." It handles the logic for both IPs and the security lockdown.

    Create the script: nano /usr/local/bin/gatekeeper.sh

    Paste the content below (Update the IP/Domain variables!):

Bash

#!/bin/bash

# --- CONFIGURATION ---
PRIMARY_IP="YOUR_MAIN_OVH_IP"
SECONDARY_IP="YOUR_SECOND_OVH_IP"

VM_PRIMARY="10.0.1.2"   # VM on vmbr1
VM_SECONDARY="10.0.2.2" # VM on vmbr2

# Management Access
ADMIN_IPS="1.2.3.4 5.6.7.8"        # Space separated
ADMIN_DOMAINS="home.example.com"    # Your DDNS FQDN

# --- RESET ---
iptables -F
iptables -t nat -F

# 1. ALLOW ADMINS
for ip in $ADMIN_IPS; do
    iptables -A INPUT -p tcp -s $ip -m multiport --dports 22,8006 -j ACCEPT
done
for domain in $ADMIN_DOMAINS; do
    RESOLVED_IP=$(dig +short $domain | tail -n1)
    if [[ ! -z "$RESOLVED_IP" ]]; then
        iptables -A INPUT -p tcp -s $RESOLVED_IP -m multiport --dports 22,8006 -j ACCEPT
    fi
done

# 2. LOCKDOWN
iptables -A INPUT -p tcp -m multiport --dports 22,8006 -j DROP

# 3. INBOUND (DNAT)
# Primary -> VM1 (Everything BUT 22/8006)
iptables -t nat -A PREROUTING -d $PRIMARY_IP -p tcp -m multiport ! --dports 22,8006 -j DNAT --to-destination $VM_PRIMARY
iptables -t nat -A PREROUTING -d $PRIMARY_IP -p udp -j DNAT --to-destination $VM_PRIMARY

# Secondary -> VM2 (Everything)
iptables -t nat -A PREROUTING -d $SECONDARY_IP -j DNAT --to-destination $VM_SECONDARY

# 4. OUTBOUND (SNAT)
# Force VM2 to use Secondary IP for internet
iptables -t nat -A POSTROUTING -s 10.0.1.0/24 -o vmbr0 -j SNAT --to-source $PRIMARY_IP
iptables -t nat -A POSTROUTING -s 10.0.2.0/24 -o vmbr0 -j SNAT --to-source $SECONDARY_IP

echo "Rules Applied Successfully."

    Make it executable: chmod +x /usr/local/bin/gatekeeper.sh

    Run it once: /usr/local/bin/gatekeeper.sh

Phase 4: Automation (Crontab)

This ensures that if your home IP changes (FQDN), the firewall updates within 5 minutes.

    Open crontab: crontab -e

    Add this line: */5 * * * * /bin/bash /usr/local/bin/gatekeeper.sh > /dev/null 2>&1

Phase 5: VM Configuration

Finally, configure your Virtual Machines to "talk" to the host.
Feature	VM 1 (Primary IP)	VM 2 (Secondary IP)
Proxmox Bridge	vmbr1	vmbr2
Static IP	10.0.1.2	10.0.2.2
Subnet Mask	255.255.255.0	255.255.255.0
Gateway	10.0.1.1	10.0.2.1
Final Verification Checklist

    From your Home IP: Can you reach https://PRIMARY_IP:8006? (Should be YES)

    From your Phone (LTE): Can you reach https://PRIMARY_IP:8006? (Should be NO)

    Inside VM 2: Run curl ifconfig.me. Does it show your Secondary IP? (Should be YES)

    Port Forwarding: Try reaching a service (like a web server) on your Secondary IP. It should hit VM 2 immediately.
