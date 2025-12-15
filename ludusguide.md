## Deploying GOAD Labs with Ludus on Proxmox

The best setup for GOAD labs in a homelab is on a Mini-PC.
You can setup GOAD labs in the cloud and on general purpose server hardware for specific usecases.
This guide is suitable for a homelab and caters to individuals who need to practice and 

## Requirements

1. Proxmox server resources:
- Storage 500GB+ , 1TB (Recommended)
- 8 Cores minimum, more is better (to run multiple labs simulteneously)
- 32GB RAM minimum
2. Ludus installed on your Proxmox server
4. Admin user created with API key
5. Windows and Linux OS templates added to Ludus

## Step 1: Install Ludus
Install Ludus on Proxmox
```sh
# Use root privileges
curl -s https://ludus.cloud/install | bash
chmod +x install.sh
./install.sh
```

Create a user
Ludus creates a root user
- get the Root user API key
```sh
ludus-install-status
# Ouput
Root API key: ROOT.yJ+vlqUzwzV%8UiWrPmaQiVuV-wwN8JTnenARbTG
```
Create Goad Admin user
```sh
export LUDUS_API_KEY='ROOT.yJ+vlqUzwzV%8UiWrPmaQiVuV-wwN8JTnenARbTG'

ludus user add --name "Goad Admin" --userid GA --admin --url https://127.0.0.1:8081
```
If the API key conatins a % - add another % sign to escape the symbol
Ex: GA.Sw=hWcCFUHs=32Lq8cDGNwtn_O2H%ft1ecRhAwVp -> export LUDUS_API_KEY='GA.Sw=hWcCFUHs=32Lq8cDGNwtn_O2H%%ft1ecRhAwVp'

Save the admin user's API key. Export it as a variable
```sh
export LUDUS_API_KEY='GA.Sw=hWcCFUHs=32Lq8cDGNwtn_O2H-ft1ecRhAwVp'
```

## Step 2: Add Required Templates

On the Ludus server, add the Windows templates:
Clone the ludus repo

```sh
git clone https://gitlab.com/badsectorlabs/ludus
cd ludus/templates
ls -la
ludus templates list
# Add any missing templates
# Make sure all templates in the folder are added
ludus templates add -d win2019-server-x64 
ludus templates add -d win2016-server-x64
# Check the templates added to ludus
ludus templates list
# Build templates
ludus templates build
```

Wait for templates to build 
```sh
# In another terminal check progress
ludus templates list
```

### The recommended strategy to deploy GOAD Ranges/Labs
1. Setup Ludus to manage the Proxmox server (the infrastructure layer)
2. Use the GOAD Script from Orange CyberDefense to manage the GOAD Ranges/Labs
It's a wrapper script with an interactive menu and relies on ludus to provision labs

## Method 1 - Using a Single Account to create GOAD Ranges

### Option A: Create a GOAD lab with the first user created 
1. You already created your first user with the admin role and exported the API key variable into the current terminal session. That's enough to get started with your first lab or second.

Note:
- You can edit the templates created in Proxmox to customize the VM specifications.
- You want to limit the VM resources to what your proxmox server can accomodate.
- Give the VMs 4GB RAM, 2 vCPU if you have minimum 32GB RAM and 8 vCPU
- Enter the Proxmox dashboard, click on each template and edit the hardware specs for RAM and Processors (vCPU)

2. Clone and Setup GOAD Script

```sh
git clone https://github.com/Orange-Cyberdefense/GOAD.git
cd GOAD
# Recommended to use poetry to create python virtual environment
pip install poetry
poetry env use python3.11
source $(poetry env info --path)/bin/activate
poetry install
```

Note: If you get any errors about unmet dependencies - fix them with this command replacing the packages with what is provided in the output

```sh
poetry run pip install --upgrade urllib3 chardet charset-normalizer requests
```



```sh
# Ensure you are using a user with root/admin role. Export the Goad Admin user API ky and run the Goad script
# Specify the IP address range to use - 10.x.x, 10.2.10, 10.3.10, 10.4.10 e.t.c
cd GOAD
python3 ./goad.sh -p ludus -ip 10.2.10
```





---------------------------------------------
```sh
ludus range config get > goad-full.yml
# You can edit goad-full.yml to set VM resources
```
Sample VM configuraiton:

```sh
ludus:
  - vm_name: "{{ range_id }}-GOAD-DC01"
    hostname: "{{ range_id }}-DC01"
    template: win2019-server-x64-template
    vlan: 10
    ip_last_octet: 10
    ram_gb: 4
    cpus: 2
    windows:
      sysprep: true
```
Deploy the range

```sh
ludus range config set -f goad-full.yml
ludus range deploy
ludus range logs -f
```
You can deploy multiple ranges by creating a user for each range to isolate them OR you can deploy multiple ranges with one user account that has the admin role.

1. First create the user in Ludus:

At this point you can create another user to deploy your GOAD lab specifically for that user or use the first user you created.

`NOTE: The subsequent users you create to deploy labs do not need to have the admin role. Keep in mind, you do not need to create a different user for each lab.`

Deploy GOAD Full (5 VMs).

```sh
ludus user add -n 'GOAD Full' -i GOADFULL --url https://127.0.0.1:8081

```
- If you intend to create mulitple labs, create a second user (without the admin role) for another lab like GOAD Light or GOAD NHA (1 user for each lab)
- 


2. Export the user API key before running ludus commands every time you're creating an isolated lab:
- Record your API keys in a text file to refer to them easily

```sh
export LUDUS_API_KEY='GOADFULL.7QWRe4cKmcz9HGgi+Gid1j5ZCs=bLE0Eo@kNtXtr'

```
3. Get the GOAD range config
```sh
ludus --user GOADFULL range config get > goad-full-config.yml

```
The range config defines the lab specifications. You can edit the config and add Elastic, Kali e.t.c. Then set the config for ludus to provision.

```sh
ludus --user GOADFULL range config set -f goad-full-config.yml
ludus --user GOADFULL range deploy

# You cam monitor the lab deployment using:
ludus --user GOADFULL range logs -f
```

### Option B: Create GOAD Full + Elastic SIEM
1. Add ansible playbooks for Elastic SIEM
```sh
ludus ansible roles add badsectorlabs.ludus_elastic_container
ludus ansible roles add badsectorlabs.ludus_elastic_agent
```

2. Edit the range config file
```sh
# Note: Edit the range config file to add specifications for Elastic SIEM. You have to add the "ludus_elastic_agent" role to all VMs
ludus --user GOADFULL range config get > goad-full-elastic.yml
ludus --user GOADFULL range config set -f goad-full-elastic.yml
```

## Method 2 - GOAD Script: Orange CyberDefense GOAD create users automatically

### Step 1: 
## Deploying GOAD-Light (3 VMs)

### Option A: Create two GOAD Labs running simulteneously but isolated - you need to have 2 users
1. The GOAD Script provides an interactive menu to create GOAD Labs using ludus under the hood. You only need to have one account with the admin role created with ludus.
2. Make sure to edit the ~/.goad/goad.ini config. 

```sh

```

```sh
ludus user add -n 'GOAD Lite' -i GOADLITE --url https://127.0.0.1:8081

```
Export the GOAD Admin user API key

```sh
export LUDUS_API_KEY='GA.xxxxxxxxxxxxxxxxxxxxxxx'
python goad.py -l GOADLITE -p ludus -m install
```

## Accessing Multiple Ranges with one user account (should have the admin role)

To run both GOAD and GOAD-Light simultaneously with a single user:

### Deploy GOAD with user SA:
```sh
export LUDUS_API_KEY='SA.v3gfHmYWIgFh11p+LHhXX+7jsyztP9ddnG%%ulkuw'
ludus --user SA range config set -f ludus-ranges/sa-config.yml

python goad.py -l GOAD -p ludus -m install
```

### Deploy GOAD-Light with user GL:
```sh
export LUDUS_API_KEY='GL.QmMj@HsTLJjuqV_-JDBm_oevurdH+c92A6KOgzuj'
ludus --user GL range config set -f ludus-ranges/gl-config.yml
python goad.py -l GOAD-Light -p ludus -m install
```

## Monitoring Deployment

Watch the deployment logs:
```sh
ludus --user SA range logs -f   # For SA user
ludus --user GL range logs -f   # For GL user
```

Check range status:
```sh
ludus --user SA range status
ludus --user GL range status
```

## Post-Deployment

### Take Snapshots
```sh
ludus --user SA range snapshot -n "clean-install"
ludus --user GL range snapshot -n "clean-install"
```

### Access the Lab
Connect via WireGuard, then access:
- GOAD network: `10.RANGENUMBER.10.X`
- Kali (if deployed): `https://10.RANGENUMBER.10.99:8444` (kali:password)

## Lab Specs

| Lab | VMs | Forests | Domains | RAM Required |
|-----|-----|---------|---------|--------------|
| GOAD | 5 | 2 | 3 | ~24GB |
| GOAD-Light | 3 | 1 | 2 | ~16GB |

## Troubleshooting

If DNS errors occur during provisioning:
```powershell
# On affected VM, remove the 10.ID.10.254 DNS entry
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses @("10.X.10.1")
```
## Removing Wireguard

```sh
 wg-quick down wg0
 systemctl stop wg-quick@wg0
 systemctl disable wg-quick@wg0
 ip link delete dev wg0
 rm -fr /etc/wireguard/
 apt remove wireguard wireguard-tools

 ```

 ## Remove a user's range

ludus range rm --user GOADFULL

ludus range rm --user GOADLITE

ludus user rm -i GOADFULL --url https://127.0.0.1:8081

ludus user rm -i GOADFULL --url https://127.0.0.1:8081