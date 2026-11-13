
# Overview

To run an rclone mount as a systemd user service, create a unit file in ```~/.config/systemd/user/``` using ```Type=notify``` and explicit paths. This lets you automatically mount cloud remotes without root privileges whenever you log in.

# 1. Create the Systemd User Service

You can create a reusable template file named ```~/.config/systemd/user/rclone@.service``` so you can mount any remote by name.
1. Ensure the directory exists by running ```mkdir -p ~/.config/systemd/user/```.
2. Create the service file ```(~/.config/systemd/user/rclone@.service)``` with the following configuration:

```bash
[Unit]
Description=Rclone FUSE mount for %i
Documentation=man:rclone(1)
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStartPre=-/usr/bin/mkdir -p %h/mnt/%i
ExecStart=/usr/bin/rclone mount \
  --config=%h/.config/rclone/rclone.conf \
  --cache-dir=%h/.cache/rclone \
  --vfs-cache-mode writes \
  --vfs-cache-max-size 1G \
  %i: %h/mnt/%i
ExecStop=/usr/bin/fusermount3 -u %h/mnt/%i
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

*Note on Paths and Flags*:
* Systemd environment variables. Systemd runs mount units without standard environment variables like HOME or PATH. Using systemd's %h specifier ensures ```--config``` and ```--cache-dir``` resolve to explicit absolute paths.
* FUSE binary. Depending on your Linux distribution, replace fusermount3 with fusermount in ExecStop if fusermount3 is not installed.
* Cache directory warnings. One point of agreement among users on the Rclone Forum is that systemd may still log a harmless warning about failing to find the user cache directory even when --cache-dir is explicitly set.
* Allow other users. If you add --allow-other to your rclone flags, you must first uncomment user_allow_other in /etc/fuse.conf (which requires root).

# 2. Enable and Manage the Mount

Replace <remote> with the exact name of the remote configured in your rclone.conf (for example, gdrive or dropbox).
* Reload systemd. Run ```systemctl --user daemon-reload``` so systemd recognizes the new unit file.
* Start and enable at login. Run ```systemctl --user enable --now rclone@<remote>.service``` to mount the folder immediately and on future logins.
* Check status and logs. Run ```systemctl --user status rclone@<remote>.service``` or ```journalctl --user -u rclone@<remote>.service -f``` to troubleshoot issues.
* Stop or restart. Use ```systemctl --user restart rclone@<remote>.service``` or ```systemctl --user stop rclone@<remote>.service``` to manage the active mount.


# 3. Key Configuration Details

* Template Naming (%i). Using @ in the filename lets you reuse this single file for multiple remotes. Any text you put after @ when starting the service replaces %i (for example, rclone@gdrive uses the gdrive remote and mounts it to ~/mnt/gdrive).
* Home Directory (%h). Systemd user units use %h to automatically resolve to your current user's home directory path.
* Service Type (Type=notify). As noted in the Rclone mount documentation, Type=notify tells systemd to wait until the mountpoint is actively ready before marking the service as started.
* No Daemon Flag. Users on the Rclone Forum agree that you should not include the --daemon flag inside a systemd unit file, as systemd manages background execution itself.


# 4. Enable and Start the Service

Replace ```remote_name``` with the exact name of your configured remote from rclone config, then run the following commands with the ```--user``` flag:

```bash
# Reload systemd to detect the new unit file
systemctl --user daemon-reload

# Enable on login and start immediately
systemctl --user enable --now rclone@remote_name.service

# Check status and logs
systemctl --user status rclone@remote_name.service
```

# 3. Keep Mounted After Logging Out (Optional)

By default, user services start when you log in and stop when your last session closes. If you are running this on a headless server or want the mount to persist across boot without an active login session, enable lingering for your user account:

```loginctl enable-linger $USER```