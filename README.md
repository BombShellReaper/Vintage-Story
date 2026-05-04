# Vintage-Story-Server

**Overview**

This is a step-by-step guide on how to set up and run a Ubuntu Vintage Story dedicated server.

**Prerequisites**

- Ubuntu server (20.04 or higher recommended)
- Basic knowledge of terminal commands
- A user with sudo privileges

> [!Caution]
> Directory structures may differ based on your specific setup.

# Step 1: Update and Upgrade Your System

    sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y

---

# Step 2: Install Required Dependencies

**Install .NET Runtime**

Vintage Story requires the .NET runtime to run on Linux.

    sudo apt install -y dotnet-runtime-8.0

> [!NOTE]
> On Ubuntu 20.04, .NET may not be available in the default repos. If the above fails, add Microsoft's package feed first:
>
>     wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O /tmp/ms-prod.deb
>     sudo dpkg -i /tmp/ms-prod.deb
>     sudo apt update
>     sudo apt install -y dotnet-runtime-8.0

**Install Screen (Session Manager)**

    sudo apt install screen -y

**Install OpenSSH Server**

This enables secure remote access to your server.

    sudo apt install openssh-server -y

**Install Runtime Libraries**

Required by the Vintage Story server engine on Linux.

    sudo apt install -y libsqlite3-0 libsdl2-2.0-0

**Install UFW (Uncomplicated Firewall)**

    sudo apt install ufw -y

---

# Step 3: Configure UFW (Uncomplicated Firewall)

Allow connections on the default Vintage Story game port:

    sudo ufw allow 42420/tcp comment "Vintage Story Game Port TCP"
    sudo ufw allow 42420/udp comment "Vintage Story Game Port UDP"

> [!TIP]
> For added security, change "any" to a specific IP address or range.

**Allow SSH Connections Through UFW** (Optional)

    sudo ufw allow from any to any port 22 comment "SSH"

> [!TIP]
> For added security, change "any" to a specific IP address or range.

Set the default rule to deny incoming traffic (Optional)

    sudo ufw default deny incoming

**Enable UFW** (UFW will enable on reboot)

    sudo ufw enable

Check the UFW status after enabling it:

    sudo ufw status

---

# Step 4: Create a Non-Sudo User

Replace *your_username* with the desired username.

    sudo adduser your_username

> [!NOTE]
> This will prompt you through the setup.

**Reboot the system**

    sudo reboot

---

# Step 5: Download and Install the Vintage Story Server

Log in as your new user, then download the latest server release from the official site.
Check https://vintagestory.at/downloads for the current version number and replace `x.x.x` below.

    mkdir -p /home/your_username/vintagestory
    cd /home/your_username/vintagestory
    wget https://cdn.vintagestory.at/gamefiles/stable/vs_server_linux-x64_x.x.x.tar.gz
    tar -xzf vs_server_linux-x64_x.x.x.tar.gz
    rm vs_server_linux-x64_x.x.x.tar.gz

> [!NOTE]
> Replace `your_username` and `x.x.x` with your actual Linux user and the current server version.

---

# Step 6: Configure the Server

Vintage Story generates `serverconfig.json` automatically on first launch. Run the server briefly to create the file, then stop it:

    cd /home/your_username/vintagestory
    ./start_server.sh --dataPath data

Once you see the server has started in the terminal, press `Ctrl+C` to stop it. The `data/` folder and config file will now exist.

Edit the config file:

    nano data/serverconfig.json

Key settings to update in the JSON:

- **ServerName**: Your server's public name.
- **Password**: Set a password for private play (leave empty for public).
- **Port**: Default is `42420`. Change if needed.
- **MaxClients**: Maximum number of simultaneous players.
- **SaveFileLocation**: Path where world saves are stored.

Example snippet:

    {
      "ServerName": "My Vintage Story Server",
      "Password": "",
      "Port": 42420,
      "MaxClients": 16,
      "SaveFileLocation": "Saves/default.vcdbs"
    }

---

# Step 7: Start the Server

To keep the server running after you close your terminal, use Screen:

    screen -S vintagestory
    cd /home/your_username/vintagestory
    ./start_server.sh --dataPath data

Detach from the screen session without stopping the server:

    Ctrl+A then D

Reattach later with:

    screen -r vintagestory

---

# Step 8: Create a Startup Script (Optional)

Return to the user's home directory:

    cd

Create a directory for your scripts. Change *name* to your desired directory name:

    mkdir scripts
    cd scripts

Create the script. Change *name.sh* to your desired script name:

    nano start_vintagestory.sh

Copy and edit the following script:

    #!/bin/bash
    
    # --- Configuration ---
    LOGFILE="/home/your_username/vintagestory/server_log.txt"  # Update your_username
    DIRPATH="/home/your_username/vintagestory"                  # Path to Vintage Story folder
    DATAPATH="$DIRPATH/data"                                    # Path to server data directory
    
    # Create log directory/file if needed
    mkdir -p $(dirname "$LOGFILE")
    touch "$LOGFILE"
    
    log() {
        echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOGFILE"
    }
    
    {
        # Start the Vintage Story server in a detached screen session
        log "Starting Vintage Story server..."
        if /usr/bin/screen -dmS VintageStoryServer "$DIRPATH/start_server.sh" --dataPath "$DATAPATH"; then
            log "Vintage Story server started in screen session 'VintageStoryServer'."
        else
            log "Failed to start Vintage Story server."
            exit 1
        fi
    } 2>&1 | tee -a "$LOGFILE"

Make the script executable:

    chmod +x start_vintagestory.sh

---

# Step 9: Create a Systemd Service (Optional)

Switch to your sudo user. Create the service file:

    sudo nano /etc/systemd/system/VintageStory.service

Add the following configuration:
(Replace `your_username` and the path to your script accordingly)

    [Unit]
    Description=Vintage Story Dedicated Server
    After=network.target
    
    [Service]
    Type=forking
    User=your_username
    ExecStart=/home/your_username/scripts/start_vintagestory.sh
    RemainAfterExit=yes
    Restart=on-failure
    RestartSec=10
    StartLimitIntervalSec=60
    StartLimitBurst=3
    
    [Install]
    WantedBy=multi-user.target

Enable and start the service:

    sudo systemctl daemon-reload
    sudo systemctl enable VintageStory.service
    sudo systemctl start VintageStory.service

---

# Step 10: Hardening (Optional)

Log in with the sudo user and edit the sshd_config file:

    sudo nano /etc/ssh/sshd_config

Locate the following lines and uncomment them, making the specified edits:

**#LoginGraceTime 2m**

    LoginGraceTime 1m

**#PermitRootLogin prohibit-password**

    PermitRootLogin no

**#MaxSessions 10**

    MaxSessions 4

Reload systemctl & restart sshd:

    sudo systemctl daemon-reload
    sudo systemctl restart ssh.service

These are some steps you can take to enhance the security of your SSH service.

**Install Fail2Ban (Recommended)**

Fail2Ban automatically bans IPs after repeated failed SSH login attempts — a set-and-forget layer of protection for any public-facing server.

    sudo apt install fail2ban -y
    sudo systemctl enable fail2ban
    sudo systemctl start fail2ban

# Change Who Can Use the Switch User (su) Command

Make a new group for the su command. Replace *group_name* with your desired name:

    sudo groupadd group_name

> **Example:** *sudo groupadd restrictedsu*

**Edit who can use the *su* command**

    sudo nano /etc/pam.d/su

Edit the following line to restrict su. Replace *group_name* with the one you made earlier:

    auth       required   pam_wheel.so group=group_name

> **Example:** *auth       required   pam_wheel.so group=restrictedsu*

---

# Step 11: Create the Maintenance Script

Create a new script file (as your sudo user):

    sudo nano /usr/local/bin/vintagestory-maintenance.sh

Paste the following code:

    #!/bin/bash
    
    # --- Configuration ---
    LOGFILE="/var/log/vintagestory_maintenance.log"
    SERVICE_NAME="VintageStory.service"
    SCREEN_NAME="VintageStoryServer"  # Must match the session name in your startup script
    WEBHOOK_URL="" # Paste your Discord Webhook URL here
    
    send_discord_message() {
        local message="$1"
        if [[ -n "$WEBHOOK_URL" ]]; then
            curl -H "Content-Type: application/json" -X POST -d "{\"content\": \"$message\"}" "$WEBHOOK_URL" > /dev/null 2>&1
        fi
    }
    
    # Create log file if it doesn't exist
    if [ ! -f "$LOGFILE" ]; then
        touch "$LOGFILE"
    fi
    
    echo "$(date +'%Y-%m-%d %H:%M:%S') --- STARTING MAINTENANCE ---" | tee -a "$LOGFILE"
    send_discord_message "🚀 **VS Maintenance:** Saving world and shutting down safely..."
    
    # 1. GRACEFUL SHUTDOWN VIA SCREEN
    # Send commands directly into the screen session.
    # The 'stuff' command simulates typing. '\015' is the Enter key.
    if screen -list | grep -q "$SCREEN_NAME"; then
        echo "$(date +'%Y-%m-%d %H:%M:%S') Sending save and shutdown commands to Screen..." | tee -a "$LOGFILE"
        
        # Notify players in-game
        su - your_username -c "screen -S $SCREEN_NAME -p 0 -X stuff \"/announce Server maintenance starting in 15 seconds. Saving world...\015\""
        sleep 15
        
        # Force a world save
        su - your_username -c "screen -S $SCREEN_NAME -p 0 -X stuff \"/server.autosavenow\015\""
        sleep 5
        
        # Graceful shutdown command
        su - your_username -c "screen -S $SCREEN_NAME -p 0 -X stuff \"/stop\015\""
    else
        echo "$(date +'%Y-%m-%d %H:%M:%S') Screen session not found. Stopping service directly." | tee -a "$LOGFILE"
    fi
    
    # 2. VERIFY SERVICE STATUS (Wait up to 60s for VS to finish writing to disk)
    MAX_WAIT=30
    COUNT=0
    while [ "$(systemctl is-active $SERVICE_NAME)" == "active" ] && [ $COUNT -lt $MAX_WAIT ]; do
        echo "Waiting for process to exit... ($((COUNT*2))s)" | tee -a "$LOGFILE"
        sleep 2
        ((COUNT++))
    done
    
    # If still running after timeout, stop via systemctl
    if [ "$(systemctl is-active $SERVICE_NAME)" == "active" ]; then
        echo "$(date +'%Y-%m-%d %H:%M:%S') Service still active. Stopping via systemctl..." | tee -a "$LOGFILE"
        systemctl stop "$SERVICE_NAME"
    fi
    
    # 3. CHECK FOR UPDATES
    apt-get update > /dev/null
    UPGRADES=$(apt list --upgradable 2>/dev/null | grep -v "Listing...")
    
    if [ -n "$UPGRADES" ]; then
        send_discord_message "✅ **Updates Found.** Applying system patches..."
        echo "$(date +'%Y-%m-%d %H:%M:%S') Applying upgrades..." | tee -a "$LOGFILE"
        DEBIAN_FRONTEND=noninteractive apt-get -o Dpkg::Options::="--force-confold" -y full-upgrade >> "$LOGFILE" 2>&1
        apt-get autoremove -y >> "$LOGFILE" 2>&1
    else
        send_discord_message "ℹ️ No updates found. System is clean."
    fi
    
    # 4. REBOOT
    send_discord_message "🔄 **Maintenance Complete:** Rebooting server. Vintage Story will auto-restart."
    echo "$(date +'%Y-%m-%d %H:%M:%S') Maintenance finished. Rebooting." | tee -a "$LOGFILE"
    sleep 2
    reboot

Make the script executable:

    sudo chmod +x /usr/local/bin/vintagestory-maintenance.sh

Schedule it with cron (Optional). Edit the root crontab:

    sudo crontab -e

Add a line to run maintenance nightly at 4:00 AM:

    0 4 * * * /usr/local/bin/vintagestory-maintenance.sh

**Monitor the Maintenance Log**

If the server doesn't come back up after a scheduled window, check the log to diagnose what happened:

    tail -f /var/log/vintagestory_maintenance.log

> [!TIP]
> To prevent the log from growing indefinitely, add a logrotate config:
>
>     sudo nano /etc/logrotate.d/vintagestory
>
> Paste the following:
>
>     /var/log/vintagestory_maintenance.log {
>         weekly
>         rotate 4
>         compress
>         missingok
>         notifempty
>     }
