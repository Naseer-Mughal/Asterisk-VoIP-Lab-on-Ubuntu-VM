# Asterisk VoIP System Setup on Ubuntu VM

This repository documents the process of setting up a basic Voice over IP (VoIP) system using Asterisk 18.x on an Ubuntu 22.04 LTS virtual machine, configuring SIP peers, and testing connectivity with Zoiper softphones. It also includes steps for enabling call transfer and call pickup features.

---

## Project Overview

This project involved:
-   Creating and configuring an Ubuntu Server VM on VMware Workstation.
-   Installing and setting up Asterisk 18.x.
-   Configuring SIP (Session Initiation Protocol) peers for multiple endpoints.
-   Implementing a basic dial plan for internal calling.
-   Enabling advanced features: **Call Transfer** (Blind) and **Call Pickup**.
-   Utilizing Zoiper softphones for end-to-end testing.

---

## Technologies Used

* **Virtualization:** VMware Workstation
* **Operating System:** Ubuntu Server 22.04 LTS
* **SSH Client:** PuTTY (for remote access to VM)
* **VoIP Platform:** Asterisk 18.x
* **SIP Protocol:** `chan_sip` module
* **Softphones:** Zoiper (on Mobile & Desktop)

---

## Setup Guide

Follow these steps to replicate the environment:

### 1. Ubuntu VM Setup on VMware Workstation

* Create a new virtual machine.
* Allocate sufficient resources (e.g., 1 CPU, 2GB RAM, 20GB HDD).
* Install Ubuntu Server 22.04 LTS.
* Configure network adapter for **Bridged Mode** (recommended for direct IP access from your LAN for softphones). Note down the VM's IP address.

### 2. SSH Access (Optional but Recommended)

* Install OpenSSH server on your Ubuntu VM:
    ```bash
    sudo apt update
    sudo apt install openssh-server
    ```
* Use PuTTY (or any SSH client) to connect to your VM's IP address.

### 3. Asterisk 18.x Installation

* Update package lists and install necessary dependencies:
    ```bash
    sudo apt update
    sudo apt upgrade -y
    sudo apt install -y build-essential git-core ncurses-dev libxml2-dev libsqlite3-dev libjansson-dev libssl-dev libsrtp2-dev libiksemel-dev uuid-dev libspeex-dev libspeexdsp-dev libogg-dev libvorbis-dev libcurl4-openssl-dev pkg-config
    ```
* Download Asterisk 18.x source (check for the latest 18.x version on `downloads.asterisk.org/pub/telephony/asterisk/`):
    ```bash
    cd /usr/src/
    sudo wget [https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-18-current.tar.gz](https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-18-current.tar.gz)
    sudo tar -zxvf asterisk-18-current.tar.gz
    cd asterisk-18.*/
    ```
* Install core prerequisites and configure build options:
    ```bash
    sudo contrib/scripts/get_mp3_source.sh ; # For MP3 support (optional)
    sudo contrib/scripts/install_prereq install
    sudo ./configure --with-jansson-bundled
    ```
* Run `menuselect` to select modules (ensure `chan_sip`, `app_dial`, `app_pickup` are selected):
    ```bash
    sudo make menuselect
    ```
    (Navigate and select modules. Save & Exit.)
* Compile and install Asterisk:
    ```bash
    sudo make
    sudo make install
    sudo make config    ; Install init scripts
    sudo make samples   ; Install sample configuration files (important!)
    ```
* Set ownership and permissions:
    ```bash
    sudo chown -R asterisk:asterisk /var/{lib,log,run,spool}/asterisk /usr/lib/asterisk /etc/asterisk
    sudo chmod -R 750 /var/{lib,log,run,spool}/asterisk /usr/lib/asterisk /etc/asterisk
    ```
* Start Asterisk and connect to CLI:
    ```bash
    sudo systemctl start asterisk
    sudo systemctl enable asterisk
    sudo asterisk -rvvv
    ```

### 4. Asterisk Configuration (`/etc/asterisk/`)

#### 4.1. `sip.conf` (for chan_sip peers)

Ensure your `sip.conf` has the following structure. Remember to use `sudo nano /etc/asterisk/sip.conf` to edit.

```ini
[general]
insecure=port,invite
context=sufi              ; This must match your dialplan context
bindport=5060
bindaddr=0.0.0.0
; For NAT situations (uncomment and set if needed):
;externip=YOUR_PUBLIC_IP_IF_BEHIND_NAT
;localnet=192.168.100.0/24 
transport=udp
qualify=yes

; --- SIP Peer 100 (Mobile) ---
[100]
type=friend
context=sufi
CALLERID=Naseer_Mob
defaultuser=100
host=dynamic
dtmfmode=rfc2833
secret=your_secret_100    ; <--- CHANGE THIS!
disallow=all
allow=alaw
allow=ulaw
qualify=yes
callgroup=1               ; For call pickup
pickupgroup=1             ; For call pickup

; --- SIP Peer 101 (Laptop) ---
[101]
type=friend
context=sufi
CALLERID=Naseer_Laptop
defaultuser=101
host=dynamic
dtmfmode=rfc2833
secret=your_secret_101    ; <--- CHANGE THIS!
disallow=all
allow=alaw
allow=ulaw
qualify=yes
callgroup=1               ; For call pickup
pickupgroup=1             ; For call pickup

; --- SIP Peer 102 (Another Device, e.g., SamsungA13) ---
[102]
type=friend
context=sufi
CALLERID=Naseer_A13
defaultuser=102
host=dynamic
dtmfmode=rfc2833
secret=your_secret_102    ; <--- CHANGE THIS!
disallow=all
allow=alaw
allow=ulaw
qualify=yes
callgroup=1               ; For call pickup
pickupgroup=1             ; For call pickup

```
#### 4.2. `features.conf` (Call Transfer Codes)

This defines how calls are routed and enables the pickup feature. Ensure your extensions.conf has the following in the [sufi] context. Remember to use sudo nano /etc/asterisk/extensions.conf to edit.

```ini
[sufi]
; Dialing internal extensions (e.g., 100, 101, 102)
exten => _X.,1,NOOP(Call towards <span class="math-inline">\{EXTEN\}\)
same \=\> n,Dial\(SIP/</span>{EXTEN},20,tT) ; tT options enable transfer by caller/callee
same => n,Hangup()

; Call Pickup Feature Code
exten => *8,1,Pickup()     ; Picks up any ringing call in the caller's pickupgroup (group 1)
same => n,Hangup()

```
#### 4.3. features.conf (Call Transfer Codes)

This defines the DTMF codes for call transfer. Ensure these lines are uncommented or added under the [featuremap] section. Remember to use sudo nano /etc/asterisk/features.conf to edit.

```ini
[featuremap]
; Blind Transfer (default is '#')
blindxfer => # 
; Attended Transfer (default is '*2')
atxfer => *2

```

### 5. Reload Asterisk Configurations

After making changes to .conf files, reload them in the Asterisk CLI:

```ini
sip reload
dialplan reload
features reload ; Note: 'features reload' might not be recognized, 'features show' confirms active

```

### 6. Zoiper Softphone Configuration

Install Zoiper on your mobile and laptop. For each extension (100, 101, 102), configure Zoiper as follows:

- Account Type: SIP
- Hostname/Domain/Server: The IP address of your Ubuntu VM (e.g., 192.168.100.159)
- Username: The extension number (e.g., 100, 101, 102)
- Password: The secret defined in sip.conf for that extension.
- Transport: UDP
- DTMF Mode: RFC 2833 (important for feature codes)

Ensure all Zoiper clients register successfully. Verify in Asterisk CLI:
```ini
sip show peers

All peers should show `OK` status.

```
### Testing the Features

#### 1. Internal Calling

- From Zoiper 100, dial 101.
- Answer on 101. Verify two-way audio.
  
#### 2. Call Transfer (Blind Transfer using `#`)
- **Scenario:** 100 calls 101. 101 transfers the call to 102.
- From Zoiper 100, dial `101`.
- Answer on Zoiper 101.
- **On Zoiper 101 (during the active call):**
  - Open the dial pad in Zoiper.
  - Dial `#` (you should hear a dial tone from Asterisk).
  - Dial `102`.
  - Hang up Zoiper 101's call.
- **Expected:** Zoiper 100 should now be connected to Zoiper 102.

#### 3. Call Pickup (using `*8`)
- **Scenario:** 100 calls 102 (ringing). 101 picks up the call.
- From Zoiper 100, dial `102`.
- Observe Zoiper 102 ringing.
- **On Zoiper 101 (while 102 is ringing):**
  - Dial `*8`.
  - Press "Call" or "Send".
- **Expected:** Zoiper 102 stops ringing, and Zoiper 101 connects to Zoiper 100.

#### Troubleshooting Tips

* **Check** `sip show peers:` Always ensure all Zoiper clients are registered (OK status).
* **Asterisk CLI Debugging:** For detailed troubleshooting, use:

```
sudo asterisk -rvvv
core set verbose 5
sip set debug on
```

This will show all SIP traffic and Asterisk internal processing.

- **Firewall:** Ensure your Ubuntu VM's firewall (UFW) is not blocking SIP (UDP 5060) or RTP (UDP 10000-20000) traffic. Temporarily disable with `sudo ufw disable` for testing, then configure rules and re-enable.
- **Context Mismatch:** Double-check that the `context` in `sip.conf`'s `[general]` section and for each `[peer]` matches the context name in `extensions.conf` (e.g., `sufi`).
- **DTMF Mode:** Ensure Zoiper is sending DTMF as `RFC 2833`.
