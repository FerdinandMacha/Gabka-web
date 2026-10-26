Isolating services into dedicated, unprivileged user accounts is an excellent security practice. It adheres to the principle of least privilege, ensuring that if the container or the application is compromised, the attacker gains no access to your personal files, sudo privileges, or the broader system.

In Fedora, running rootless Podman containers under a dedicated service account is fully supported. Below is the best-practice methodology for creating and configuring this isolated environment.

### 1. Create the Dedicated Service Account
We will create a standard user (to ensure proper UID/GID mapping for rootless Podman) but restrict it from interactive logins and sudo access.

```bash
# Create the user with a home directory, but set the shell to nologin
sudo useradd -m -s /sbin/nologin -c "Uptime Kuma Service Account" uptimekuma
```

### 2. Verify Rootless Podman UID Mappings
For rootless Podman to map the container's internal root user to an unprivileged user on the host, the service account must have entries in `/etc/subuid` and `/etc/subgid`. Fedora usually configures this automatically upon user creation, but it is best practice to verify it.

```bash
grep uptimekuma /etc/subuid
grep uptimekuma /etc/subgid
```
*If these commands return no output, you must manually assign the sub-UIDs and sub-GIDs:*
```bash
sudo usermod --add-subuids 200000-265535 --add-subgids 200000-265535 uptimekuma
```

### 3. Enable Lingering for the Service Account
Because the account has `/sbin/nologin` as its shell, it will never "log in" in the traditional sense. To ensure its systemd user instance starts at boot and keeps the Podman container running, you must enable lingering.

```bash
sudo loginctl enable-linger uptimekuma
```

### 4. Deploy the Quadlet Configuration
Since you cannot log in as the `uptimekuma` user directly, you will execute the setup commands inside a machinectl shell. 

```bash
sudo machinectl shell uptimekuma@ /bin/bash
```

First, create the necessary directory structure for Quadlet:
```bash
mkdir -p /home/uptimekuma/.config/containers/systemd/
```

Next, create the container definition file. We will use `tee` to write the file directly as the `uptimekuma` user:
```bash
tee /home/uptimekuma/.config/containers/systemd/uptime-kuma.container > /dev/null <<EOF
[Unit]
Description=Uptime Kuma Monitoring Service
After=network-online.target
Wants=network-online.target

[Container]
Image=docker.io/louislam/uptime-kuma:release
ContainerName=uptime-kuma
PublishPort=3001:3001

# Using a named volume ensures Podman handles SELinux contexts automatically
Volume=uptime-kuma-data:/app/data

AutoUpdate=registry

[Service]
Restart=always

[Install]
WantedBy=default.target
EOF
```

### 5. Start and Enable the Service
Now, instruct the `uptimekuma` user's systemd instance to reload its configuration and start the container.

Run the following commands inside your `machinectl` shell:

```bash
# Reload the daemon for the service account
systemctl --user daemon-reload

# Enable and start the container
systemctl --user start uptime-kuma.service
```

### 6. Verify and Manage the Service
You can verify that the container is running and check its logs without switching users:

```bash
# Check the status of the service
systemctl --user status uptime-kuma.service

# View the container logs
journalctl --user -u uptime-kuma.service -f

# Verify the Podman container is active
podman ps
```

### 7. Accessing the Web Interface
Because the container is bound to port `3001` on the host's network namespace, you can access the Uptime Kuma web interface from your primary user account by simply navigating to `http://localhost:3001` in your browser. The `uptimekuma` user does not need any special permissions to serve this traffic, and your primary user does not need any special permissions to access it locally.

### Security and SELinux Considerations on Fedora
*   **SELinux:** By using a named Podman volume (`Volume=uptime-kuma-data:/app/data`) rather than a bind mount to a specific host directory, Podman automatically manages the SELinux contexts. You will not encounter "Permission denied" errors related to SELinux.
*   **Firewall:** If you only access the dashboard via `localhost`, no `firewalld` rules are required. If you need to access it from another device on your local network, you must open the port globally (as `firewalld` operates independently of the user running the service):
    ```bash
    sudo firewall-cmd --permanent --add-port=3001/tcp
    sudo firewall-cmd --reload
    ```
*   **Privilege Escalation:** The `uptimekuma` user has no `sudo` privileges. Even if an attacker exploits a vulnerability in Uptime Kuma, they are confined strictly to the container's environment and the `uptimekuma` user's home directory.

This architecture provides a highly secure, robust, and modern monitoring deployment perfectly aligned with Fedora's security models.



### Recovery when the password is genuinely lost

Uptime Kuma documents a **CLI password reset routine**. The project ships a small helper script (historically `extra/reset-password.js`) intended to be executed inside the running container. With your Podman deployment, the equivalent invocation is:

```bash
sudo machinectl shell uptimekuma@ /bin/bash
podman exec -it uptime-kuma sh
```

Inside the container, inspect the image to confirm the current path and entry point before running it, since v2 reorganised parts of the tree:

```sh
pwd
ls
cat package.json
ls extra/ 2>/dev/null
```

Look for the reset script or an `npm run` target named for password reset, then execute it. It will assign a **new** password; it cannot print the old one, because only the hash is stored.

If you had enabled TOTP, the same recovery flow must also clear the 2FA enrolment, otherwise you will be reset to a password challenge you still cannot complete. Check the project's *Reset Password via CLI* wiki page for the v2-specific instructions before relying on the older v1 command.

**Do not delete the `uptime-kuma-data` volume to "reset" the login.** That destroys your monitors, status pages, and notification configuration along with the password.

---

### 4. The pragmatic recovery plan

For a single-user, SSH-tunnel-only deployment, this is the setup I would actually recommend:

1. Generate a random password (`openssl rand -base64 32`) and set it in the UI.
2. Store it in **GNOME Keyring** via `secret-tool` — no extra manager.
3. Export a volume backup once configuration is complete:
   ```bash
   sudo -u uptimekuma podman volume export uptime-kuma-data > ~/uptime-kuma-backup.tar
   ```
   Store that backup somewhere other than this machine.
4. If 2FA is enabled, record the TOTP seed in the same keyring entry or securely printed.
5. If everything is lost: restore or reuse the volume, run the **CLI password reset**, log in, and store the new secret again.

That gives you both paths covered: **on-demand retrieval** through the OS keyring, and **guaranteed recovery** through the CLI, without introducing a password manager or weakening your rootless, firewall-closed deployment.