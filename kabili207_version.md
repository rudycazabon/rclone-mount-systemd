[GIST LINK](https://gist.github.com/kabili207/2cd2d637e5c7617411a666d8d7e97101)

# rclone systemd service

## Preparation

This service will use the same remote name you specified when using [`rclone config create`](https://rclone.org/commands/rclone_config_create/). If you haven't done that yet, do so now.

Next, create the mountpoint for your remote. The service uses the location `~/mnt/<remote>` by default.
```bash
mkdir ~/mnt/dropbox
```

The `--allow-other` option is required in order to work in many desktop environments. This flag must be enabled by adding `user_allow_other` to `/etc/fuse.conf`. If you aren't using a desktop environment, such as on a server, this option can be omitted.

## Adding the service

Save the `rclone@.service` file in `~/.config/systemd/user/`
Make sure you include the `@`. This is required to work.

As your normal user, run:
```bash
systemctl --user daemon-reload
```

## Using the service

You can now start/enable each remote by using `rclone@<remote>`

```bash
systemctl --user enable --now rclone@dropbox

# User service for Rclone mounting
#
# Place in ~/.config/systemd/user/
# File must include the '@' (ex rclone@.service)
# As your normal user, run 
#   systemctl --user daemon-reload
# You can now start/enable each remote by using rclone@<remote>
#   systemctl --user enable --now rclone@dropbox

[Unit]
Description=rclone: Remote FUSE filesystem for cloud storage config %i
Documentation=man:rclone(1)
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStartPre=-/usr/bin/mkdir -p %h/mnt/%i
ExecStart= \
  /usr/bin/rclone mount \
    --config=%h/.config/rclone/rclone.conf \
    --vfs-cache-mode writes \
    --vfs-cache-max-size 100M \
    --log-level INFO \
    --log-file /tmp/rclone-%i.log \
    --umask 022 \
    --allow-other \
    %i: %h/mnt/%i
ExecStop=/bin/fusermount -u %h/mnt/%i

[Install]
WantedBy=default.target
```