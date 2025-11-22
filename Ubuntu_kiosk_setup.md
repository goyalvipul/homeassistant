Below is the fully formatted .md (Markdown) file, ready to save directly as README.md in your GitHub repository.

# Ubuntu Home Assistant Kiosk Setup
A complete guide to creating a stable, auto-restarting, fullscreen Home Assistant dashboard on Ubuntu.

---

## Phase 1: Operating System Preparation

### **Goal:**  
Ensure the system never sleeps, hides the mouse, and allows external control.

---

### 1. Switch to Xorg (Critical)

Ubuntu defaults to **Wayland**, which blocks screen-control scripts. Switch to **Xorg (X11)**:

1. Log out of your current session.  
2. Click your username. Before entering your password, click the **gear icon**.  
3. Select **Ubuntu on Xorg**.  
4. Log in.

**Permanent configuration change:**

```bash
sudo nano /etc/gdm3/custom.conf


Find:

#WaylandEnable=false


Uncomment it:

WaylandEnable=false


Save (Ctrl+O), exit (Ctrl+X).

2. Disable Power Saving & Install Utilities

Run the following to install browser utilities, enable SSH, and disable system sleep:

# Update and install Chromium, SSH Server, Unclutter, Xdotool
sudo apt update
sudo apt install chromium-browser openssh-server unclutter xdotool -y

# Prevent all suspend / sleep states
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target

3. Hide the Mouse Cursor

Open Startup Applications.

Add the following entry:

Name: Hide Cursor

Command: unclutter -idle 0.5 -root

Phase 2: Kiosk Browser Setup
Goal:

Launch Home Assistant automatically in fullscreen, restart on crash, and allow audio autoplay.

1. Create the Kiosk Service

Create a systemd service:

sudo nano /etc/systemd/system/kiosk.service


Paste and update YOUR_USER and HA URL:

[Unit]
Description=Chromium Kiosk
Wants=graphical.target
After=graphical.target

Environment=DISPLAY=:0.0
Environment=XAUTHORITY=/home/YOUR_USER/.Xauthority
Type=simple
ExecStart=/usr/bin/chromium-browser \
  --kiosk \
  --incognito \
  --noerrdialogs \
  --disable-infobars \
  --autoplay-policy=no-user-gesture-required \
  --check-for-update-interval=31536000 \
  http://192.168.1.XXX:8123
Restart=always
User=YOUR_USER
Group=YOUR_USER

[Install]
WantedBy=graphical.target

Important Flags

--kiosk – fullscreen mode

--incognito – prevents session restore prompts

--autoplay-policy=no-user-gesture-required – critical for camera & doorbell audio

2. Enable the Service
sudo systemctl daemon-reload
sudo systemctl enable kiosk.service


(Restart computer later to start the service.)

Phase 3: Home Assistant Integration
Goal:

Pair the device with HA, hide UI elements, and customize per-device display.

1. Install Browser Mod (HACS)

Go to HACS → Integrations → Explore and install Browser Mod.

Restart Home Assistant.

Go to Settings → Devices → Add Integration → Browser Mod.

On the Ubuntu kiosk, open the sidebar and register the device.

Example name: wall_dashboard

2. Install Kiosk Mode (HACS)

Search and install Kiosk Mode via HACS.

Add to Dashboard YAML (Edit Dashboard → 3 dots → Raw Config Editor):

kiosk_mode:
  entity_settings:
    - entity:
        browser_id: wall_dashboard
      hide_header: true
      hide_sidebar: true

Phase 4: Automations & Hardware Control
Goal:

Wake the screen on motion, control display power, and show doorbell popups.

1. Setup SSH Keys (For Screen Wake)

On Home Assistant Terminal / SSH add-on:

ssh-keygen -t rsa
ssh-copy-id your_ubuntu_user@192.168.1.XXX


Test:

ssh your_ubuntu_user@192.168.1.XXX "export DISPLAY=:0; xset dpms force on"

2. Create the Screen Control Switch

Add to configuration.yaml:

command_line:
  - switch:
      name: Wall Panel Screen
      command_on: >
        ssh -i /config/.ssh/id_rsa -o StrictHostKeyChecking=no ubuntu_user@192.168.1.XXX "export DISPLAY=:0; xset dpms force on"
      command_off: >
        ssh -i /config/.ssh/id_rsa -o StrictHostKeyChecking=no ubuntu_user@192.168.1.XXX "export DISPLAY=:0; xset dpms force off"

3. Automation: Wake on Motion

Assumes binary_sensor.hallway_motion.

alias: "Wall Panel: Wake on Motion"
trigger:
  - platform: state
    entity_id: binary_sensor.hallway_motion
    to: "on"
action:
  - service: switch.turn_on
    target:
      entity_id: switch.wall_panel_screen

4. Automation: Doorbell Popup

Assumes camera.front_door and binary_sensor.doorbell_pressed.

alias: "Wall Panel: Doorbell Popup"
trigger:
  - platform: state
    entity_id: binary_sensor.doorbell_pressed
    to: "on"
action:
  - service: switch.turn_on
    target:
      entity_id: switch.wall_panel_screen

  - service: browser_mod.popup
    data:
      browser_id: wall_dashboard
      title: Front Door
      timeout: 60000
      content:
        type: picture-entity
        entity: camera.front_door
        camera_view: live

Decision Guide & Limitations
Limitations

Touch input: Ubuntu on-screen keyboards are less smooth than Android.

Power recovery: Chromebooks may lack a “Power On After Power Loss” setting.

Motion detection: Requires external motion sensors (unless using CPU-heavy webcam detection).

Choose This Setup If…

You want a large, stable, always-on display

Prefer Ethernet and fast camera feed performance

Want zero battery management

Choose a Tablet If…

You need camera-based screen wake

You type often on the device

You want a lightweight interface


---

If you want, I can also:

✅ Package this into a downloadable `README.md` file  
✅ Create a full GitHub repo structure (`/services`, `/ha_config`, `/scripts`)  
✅ Add screenshots or architecture diagrams  
