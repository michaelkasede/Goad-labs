# Deploying GOAD Labs on Ludus (Proxmox)

## Prerequisites

1. Ludus installed on your Proxmox server
2. Admin user created with API key
3. Windows Server 2016 and 2019 templates added to Ludus

## Step 0: Install Ludus


Create a user
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
Ex: GA.Sw=hWcCFUHs=32Lq8cDGNwtn_O2H%ft1ecRhAwVp -> GA.Sw=hWcCFUHs=32Lq8cDGNwtn_O2H%%ft1ecRhAwVp

Save the admin user's API key. Export it as a variable
```sh
export LUDUS_API_KEY='GA.Sw=hWcCFUHs=32Lq8cDGNwtn_O2H-ft1ecRhAwVp'
```

## Step 1: Add Required Templates

On the Ludus server, add the Windows templates:
Clone the ludus repo

```sh
git clone https://gitlab.com/badsectorlabs/ludus
cd ludus/templates
ls -la
# Add any missing templates
# Make sure all templates in the folder are added
ludus templates add -d win2019-server-x64 
ludus templates add -d win2016-server-x64
ludus templates list
ludus templates build
```

Wait for templates to build (check with `ludus templates list`).

## Step 2: Clone and Setup GOAD

```sh
git clone https://github.com/Orange-Cyberdefense/GOAD.git
cd GOAD
pip install -r requirements.yml  # You have to use a virtual env witht his method
# or use poetry:
pip install poetry
poetry env use python3.11
poetry install
source $(poetry env info --path)/bin/activate
```

Note: If you get any errors about unmet dependencies - fix them with this command replacing the packages with what is provided in the output

```sh
poetry run pip install --upgrade urllib3 chardet charset-normalizer requests
```

## Method 1 - LUDUS

### Option A: Deploy GOAD Full (5 VMs) with a user for each range to isolate them 

1. First create the user in Ludus:
```sh
ludus user add -n 'GOAD Full' -i GOADFULL --url https://127.0.0.1:8081

```

2. Export the user API key before running:
```sh
export LUDUS_API_KEY='GOADFULL.7QWRe4cKmcz9HGgi+Gid1j5ZCs=bLE0Eo@kNtXtr'

```
3. Get the GOAD range config
```sh
ludus range config get > goad-full-config.yml

```
Here you can edit the config and add Elastic, Kali e.t.c. Then set the config

```sh
ludus range config set -f goad-full-config.yml
# Open a new terminal to run the Goad script from the Orange CyberDefense Repo
# First, enter the Goad script virtual environment

cd GOAD
source $(poetry env info --path)/bin/activate
./goad.sh -l GOAD -p ludus -ip 10.2.10

check
install
```

### Option B: Create GOAD Full + Elastic SIEM
1. Add ansible playbooks for Elastic
```sh
ludus ansible roles add badsectorlabs.ludus_elastic_container
ludus ansible roles add badsectorlabs.ludus_elastic_agent
```

2. Edit the range config file
```sh
# Note: You have to add the "ludus_elastic_agent" role to all VMs
ludus range config set -f goad-full-elastic.yml
```

### Method 2 - GOAD Script: Orange CyberDefense Sc GOAD create users automatically

```sh
# Export the Goad Admin user API key and run the Goad script
# Using the python script:
cd GOAD
python3 goad.py -l GOAD -p ludus -ip 10.2.10 -m install
```

## Deploying GOAD-Light (3 VMs)

### Option A: Create two GOAD Labs running simulteneously but isolated - you need to have 2 users
1. Create a 2nd user for GOAD Lite

```sh
ludus user add -n 'GOAD Lite' -i GOADLITE --url https://127.0.0.1:8081

```
Export the GOAD Admin user API key

```sh
export LUDUS_API_KEY='GA.xxxxxxxxxxxxxxxxxxxxxxx'
python goad.py -l GOADLITE -p ludus -m install
```

## Option D: Creating Multiple Ranges with two different users

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