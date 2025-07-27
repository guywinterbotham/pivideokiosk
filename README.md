# pivideokiosk
Raspberry Pi Video Kiosk Configuration Manual
This document outlines the steps to configure a Raspberry Pi to function as an automated video kiosk, syncing content from Google Drive and playing it on a continuous loop.

1. Prepare Raspberry Pi OS
Ensure your Raspberry Pi OS (formerly Raspbian) is installed and updated.

Install Raspberry Pi OS (64-bit Lite recommended for kiosks):
Use Raspberry Pi Imager to flash the OS to your SD card. For a headless kiosk, the "Lite" version is sufficient and more efficient.

Enable SSH:
For headless setup, enable SSH via Raspberry Pi Imager settings or by creating an ssh file in the boot partition of the SD card.

Connect and Update:
Connect your Raspberry Pi to the network and SSH into it.

Bash

sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove -y
Install Necessary Software:

Bash

sudo apt install rclone mpv
2. Configure Rclone for Google Drive Shared Drive
rclone will handle syncing your videos from Google Drive.

Start Rclone Configuration:

Bash

rclone config
Follow the interactive prompts:

n) for New remote.

Enter a name for your remote (e.g., promos).

Select drive (Google Drive) from the list.

Leave client_id and client_secret blank (press Enter).

For scope, select 1 (Full access to all files, including Shared Drives).

Leave root_folder_id and service_account_file blank.

For Edit advanced config?, type n.

For Use auto config?, type n (since you're likely SSH'd in).

You will be given a URL. Copy this URL and paste it into a web browser on your local computer.

Log in with your Google account. Grant rclone permission.

You will receive a verification code. Copy this code and paste it back into your SSH terminal.

For Configure this as a team drive?, type y.

rclone will then display a list of Shared Drives your account has access to. Select the correct one by entering its corresponding number.

For Keep this "your_remote_name" remote?, type y (or press Enter).

Finally, type q to quit the rclone config utility.

Test Rclone Connectivity:
Replace promos with your chosen remote name and Promos with your video folder name.

Bash

rclone lsd promos:
rclone ls promos:Promos/
You should see your Promos folder and then your MP4 files listed.

3. Setup RAM Disk for Temporary Video Storage
A RAM disk (tmpfs) protects your SD card from wear and offers faster video access.

Create Mount Point:

Bash

sudo mkdir /mnt/kiosk_videos
sudo chown display:display /mnt/kiosk_videos
(Replace display with your Raspberry Pi username if different).

Add tmpfs to /etc/fstab:

Bash

sudo nano /etc/fstab
Add the following line to the end of the file. Adjust size=1G if your videos require more or less space (e.g., 512M, 2G).

tmpfs /mnt/kiosk_videos tmpfs defaults,noatime,nosuid,nodev,noexec,size=1G 0 0
Save and exit (Ctrl+X, Y, Enter).

Mount the RAM Disk and Reload Systemd:

Bash

sudo systemctl daemon-reload
sudo mount -a
Verify RAM Disk:

Bash

df -h /mnt/kiosk_videos
You should see tmpfs mounted with the specified size.

4. Create Video Playback Script
This script syncs videos and plays them in a loop.

Create the Script File:

Bash
nano ~/play_kiosk_videos.sh

Ensure GDRIVE_REMOTE_NAME and GDRIVE_VIDEO_FOLDER match your setup.

chmod +x ~/play_kiosk_videos.sh

5. Schedule and Autostart with Systemd
This ensures the kiosk starts automatically on boot and runs reliably.

Create Systemd Service File:
Bash
sudo nano /etc/systemd/system/kiosk-video.service
Paste the following content:

Ini, TOML

[Unit]
Description=Kiosk Video Player Service
After=network.target graphical.target

[Service]
Type=simple
User=display
Group=display
Environment="DISPLAY=:0"
ExecStart=/home/display/play_kiosk_videos.sh
Restart=always
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=kiosk-video

[Install]
WantedBy=graphical.target
Save and exit (Ctrl+X, Y, Enter).

Reload Systemd and Enable Service:

Bash

sudo systemctl daemon-reload
sudo systemctl enable kiosk-video.service
Start the Service:

Bash

sudo systemctl start kiosk-video.service
Check Service Status and Logs:

Bash

sudo systemctl status kiosk-video.service
journalctl -u kiosk-video.service -f
6. Power Management for LabWC (Wayland)
Since you are using labwc (a Wayland compositor), xset commands are not applicable. We'll use wlr-randr for display control.

Install wlr-randr:

Bash

sudo apt update
sudo apt install wlr-randr
Identify Your Display Output Name:
You found this using the "Screen Configuration" menu option on the unit. This will be something like HDMI-A-1.

Create a Helper Script for Wayland Commands:
This script will be called by cron to manage the display.

Bash

sudo nano /usr/local/bin/wayland-display-control.sh
Replace <YOUR_DISPLAY_OUTPUT_NAME> with the exact name you found.

Bash

#!/bin/bash
LOG_FILE="/home/display/wayland_display_control.log"
USER="display"
DISPLAY_OUTPUT="<YOUR_DISPLAY_OUTPUT_NAME>" # E.g., HDMI-A-1

ACTION="$1" # 'on' or 'off'

echo "$(date): Attempting to turn display $DISPLAY_OUTPUT $ACTION." >> "$LOG_FILE"

USER_UID=$(id -u "$USER")
RUNTIME_DIR="/run/user/$USER_UID"

if [ -d "$RUNTIME_DIR" ] && [ -S "$RUNTIME_DIR/wayland-0" ]; then
    su - "$USER" -c "export XDG_RUNTIME_DIR='$RUNTIME_DIR'; export WAYLAND_DISPLAY='wayland-0'; /usr/bin/wlr-randr --output '$DISPLAY_OUTPUT' --$ACTION" >> "$LOG_FILE" 2>&1
    if [ $? -eq 0 ]; then
        echo "$(date): Successfully set display $DISPLAY_OUTPUT $ACTION." >> "$LOG_FILE"
    else
        echo "$(date): Failed to set display $DISPLAY_OUTPUT $ACTION. Check logs for wlr-randr output." >> "$LOG_FILE"
    fi
else
    echo "$(date): Wayland session not found for user $USER or Wayland socket not available. Cannot control display." >> "$LOG_FILE"
fi
Save and exit (Ctrl+X, Y, Enter).

Make Helper Script Executable:

Bash

sudo chmod +x /usr/local/bin/wayland-display-control.sh
Add Cron Jobs for Display On/Off:
Edit the root crontab:

Bash

sudo crontab -e
Add the following lines. Adjust 0 22 (10 PM) and 0 6 (6 AM) to your desired times.

Code snippet

# Turn off display at 10:00 PM every day using Wayland
0 22 * * * /usr/local/bin/wayland-display-control.sh off

# Turn on display at 6:00 AM every day using Wayland
0 6 * * * /usr/local/bin/wayland-display-control.sh on
Save and exit (Ctrl+X, Y, Enter).

Maintenance and Updating
To ensure your Raspberry Pi video kiosk runs smoothly and reliably long-term, follow these best practices.

1. Regular System Updates
Periodically update your Raspberry Pi OS to get the latest security patches, bug fixes, and software improvements.

Frequency: At least once every few months, or more frequently if critical updates are released.

Procedure:

Bash

sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove -y
It's recommended to reboot after a full-upgrade.

2. SD Card Longevity
While the RAM disk significantly reduces wear, the OS still writes to the SD card.

High-Quality SD Card: Always use a reputable brand (e.g., SanDisk, Samsung) with a good speed rating (A1 or A2 for applications).

Monitor SD Card Health (Advanced): Tools like smartmontools can sometimes provide basic health info for SD cards, but their effectiveness varies.

Consider SSD Upgrade: For maximum longevity and performance, consider booting your Raspberry Pi from a USB-connected Solid State Drive (SSD). This is a more involved setup but offers superior durability and speed.

3. Physical Security
If the kiosk is in a public or accessible location, protect the hardware.

Robust Enclosure: Use a durable case for the Raspberry Pi to protect it from dust, accidental damage, and tampering.

Secure Mounting: Mount the enclosure securely to a wall or surface to prevent theft.

Cable Management: Keep power, HDMI, and network cables tidy and concealed to prevent accidental disconnections or damage.

4. Cooling
Running 24/7, especially in an enclosed space, can generate heat.

Heatsinks/Fan: Ensure your Raspberry Pi has adequate cooling. Many cases come with passive heatsinks or small fans.

Monitor Temperature: Periodically check the CPU temperature:

Bash

vcgencmd measure_temp
If temperatures consistently exceed 60-70°C, consider improving ventilation or adding active cooling.

5. Basic Troubleshooting Tips
If the kiosk stops displaying videos or syncing, use these commands via SSH for diagnosis.

Check Kiosk Service Status:

Bash

sudo systemctl status kiosk-video.service
Look for Active: active (running). If not, try sudo systemctl start kiosk-video.service.

View Kiosk Service Logs (real-time):

Bash

journalctl -u kiosk-video.service -f
This shows the script's output, including sync progress and playback messages. Press Ctrl+C to exit.

Check Rclone Configuration:

Bash

rclone config show promos:
(Replace promos with your remote name). This confirms rclone can still access your Google Drive configuration.

Test Internet Connectivity:

Bash

ping google.com
No internet means rclone cannot sync.

Check Display Power Control Logs:

Bash

cat /home/display/wayland_display_control.log
This log will show if the scheduled display on/off commands are executing successfully.

Check RAM Disk Usage:

Bash

df -h /mnt/kiosk_videos
Ensure the RAM disk isn't full, though rclone sync should manage space effectively.

6. SD Card Backup
Regularly back up your SD card. This is the fastest way to recover your entire kiosk setup in case of SD card corruption or failure.

Method: Use the Raspberry Pi Imager tool on a separate computer. Select the "Backup OS" option and choose your Raspberry Pi's SD card.

Frequency: After initial setup, and after any significant configuration changes.
