# Phase 1 Decisions:

**Decision:** Chose Debian 13 on IONOS Linux M+ (2 vCPU, 4 GB, 120 GB NVMe); Cloudflare for DNS  
**Reason:** Debian as a reliable OS with extensive docs, and compatibility. IONOS reliable provider. Cloudflare reliable provider and personal existing experience with it.  

**Decision:** SSH Key pair: SSH keys: ed25519, a dedicated key per server, passphrase-protected  
**Reason:** SSH key pair per server access to prevent unauthorized access through password compromise. Modern, fast, short key type picked: ed25519. Local machine key gneration with passphrase protection for secure file storage.

**Decision:** Non-root admin user with sudo added as primary working user for VPS.  
**Reason:** admin user using sudo is more secure to not compromise root access and leaves a sudo audit trail.  

**Decision:** SSH hardended locking server access strictly to the admin user created using ssh key pair connection from local machine.  
**Reason:** Locks all access strictly to the admins machine using the ssh key pair to block unauthorized access or access attempts through user or root + password credential attempts.  

**Decision:** IONOS Firewall Setup: Added web-baseline Firewall rules to the IONOS VPS in the IONOS control panel. IONOS Firewall now only opens 22, 80, 443; SSH is not IP-restricted accounting for dynamic IP. 
**Reason:** Added baseline web based Firewall rules to IONOS VPS Firewall using the control panel and Debian Firewall rules inside the VPS.

**Decision:** Debian Firewall Setup: Use ufw over Debian's native layer nftables for simplicity sake. Set default rules mirroring the IONOS based rules. Default deny incoming and default allow outgoing.
**Reason:** ufw choice based on simple solutions rule for the project and my current familiarity with ufw. Mirrored IONOS firewall rules on the Debian host firewall. The two firewalls act as redundancy against compromise on one firewall or the other. Default deny incoming and allow outgoing rules based on allowing the IONOS VPS to make its own outbound connections for updates, certificate renewal while denying all inbound traffic on the host. Simply put, this allows the server to always be able to start an outbound connection for itself, but only allowed inbound connections are permitted. (Server can always start a conversation, but nobody can start a conversation with the server unless it is already allowed.)  

**Decision:** Automaic security updates setup using the tool unattended-upgrades.  
**Reason:** unattended-upgrades provide a tool to automatically check and install Debian security upgrades to avoid risk of manual upgrades not being implemented routinely.
