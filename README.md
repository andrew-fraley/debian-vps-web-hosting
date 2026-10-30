# Debian VPS for Web Hosting 

## Project Overview:

### Purpose:

Setup a Debian 13 based Virtual Private Server used as a practical hosting platform for websites (personal and client), along with API backends to power certain functionality within the websites.

### Primary Goal:

Deploy, operate, and maintain Hugo websites using Caddy on a hosted Debian server, with secure remote administration and HTTPS. Additional goals are to expand the to support multiple websites and Python/FastAPI backend(s) on the same server.

### Guiding Principles:

- Prefer free and open-source software.  
- Keep architecture simple and understandable.  
- Documentation of procedures and verified results.  
- Add complexity only if a real requirement justifies it.  
- Maintain control over project files, configurations, and data.  
- Ensure security and availability are prioritized in the environment.  

### Compute Specifications:

- Platform: IONOS  
- OS: Debian 13  
- DNS: Cloudflare  
- Compute: 2vCPU, 4GB, 120 GB NVMe

# Project Plan

## Phase 1 Project Preparation and Debian Baseline

### Overview: 

This was the initial VPS and configuation of the IONOS virtual private server and the Debian host machine.  

This phase established a secure admin access through SSH key pairs between the client machine and the VPS Debian host. Also disallowed access for root user and any user using a normal SSH user/password login, this restricted strict SSH access between the client and server using only the keypair setup. 

Also, configured and verified firewall rules for the IONOS server layer firewall and the Debian host layer firewall. This adds a redundancy to the firewall for the setup in case of one firewall layer being compromised.  

Next, setup automatic security upgrades for the Debian host to ensure security updates and upgrades are checked and installed automatically thorugh the automated tool.  

All of the above is documented and verified through the process of this phase.  

### Technical tools, systems, and software used:

- IONOS as the hosting provider for the VPS.  
- Debian 13 as the operating system on the server host.  
- SSH for secure client access to the server.   
- IONOS Firewall for the server layer firewall.  
- ufw "uncomplicated firewall" for the Debian host layer firewall.
- unattended-upgrades for the automated security upgrades.


