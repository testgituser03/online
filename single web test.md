
## 1. Introduction
Apache is a powerful and widely used open-source web server. This guide walks you through the steps necessary to configure an Apache instance on Ubuntu for a single website, accessible within your LAN.

---

## 2. Prerequisites
- **Ubuntu server** (local or virtual/cloud instance).
- A **sudo-enabled user** to run administrative commands.
- Basic **Linux command-line knowledge**.
- A **static or reserved IP address** on your LAN for consistent accessibility (recommended).

---

## 3. Update and Install Apache

### 3.1 System Update
Keeping packages up to date reduces conflicts and security vulnerabilities:
```bash
sudo apt update && sudo apt upgrade -y
```

### 3.2 Install Apache
Install the Apache web server:
```bash
sudo apt install apache2 -y
```

### 3.3 Verify Apache Status
Check that Apache is active:
```bash
sudo systemctl status apache2
```
You should see an output indicating it’s “active (running).”

### 3.4 Test the Default Page
Open a browser on **the server** or any system with network access to the server, and go to:
```
http://<server-IP>
```
If you see the “Apache2 Ubuntu Default Page,” then Apache is serving requests successfully.

---

## 4. Firewall Configuration

### 4.1 Using UFW (Uncomplicated Firewall)

1. **Enable UFW** (if not already enabled):
   ```bash
   sudo ufw enable
   ```
2. **Allow HTTP (port 80)**:
   ```bash
   sudo ufw allow 80/tcp
   ```
   If you plan to enable HTTPS, allow that as well:
   ```bash
   sudo ufw allow 443/tcp
   ```
3. **Check status**:
   ```bash
   sudo ufw status
   ```

### 4.2 Using iptables (Alternative Option)
If you prefer iptables directly (without UFW):

1. **Allow HTTP (port 80) from anywhere**:
   ```bash
   sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
   ```
2. **Allow HTTPS (port 443) if needed**:
   ```bash
   sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
   ```
3. **Allow traffic from LAN only** (example: `192.168.1.x` range):
   ```bash
   sudo iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 80 -j ACCEPT
   ```
4. **Block all other inbound traffic** (as a default policy):
   ```bash
   sudo iptables -P INPUT DROP
   sudo iptables -P FORWARD DROP
   sudo iptables -P OUTPUT ACCEPT
   ```
5. **Save iptables rules** so they persist across reboots (on Ubuntu):
   ```bash
   sudo apt install iptables-persistent
   sudo netfilter-persistent save
   ```

> **Tip**: When tightening firewall rules, ensure you don’t lock yourself out. Use safe approaches, like a separate SSH session, so you can revert if needed.

---

## 5. IP Binding and Network Configuration

### 5.1 Bind Apache to a Specific IP Address
If your server has multiple interfaces or IPs (e.g., one public, one LAN), you can force Apache to listen only on a specific LAN IP (e.g., `192.168.1.10`):

1. **Edit the ports configuration**:
   ```bash
   sudo nano /etc/apache2/ports.conf
   ```
2. **Replace** the default `Listen 80` with:
   ```
   Listen 192.168.1.10:80
   ```
   And if you’re using HTTPS/SSL:
   ```
   Listen 192.168.1.10:443
   ```
3. **Save** and exit (`Ctrl+O`, `Ctrl+X`), then **restart** Apache:
   ```bash
   sudo systemctl restart apache2
   ```

> **Note**: The `192.168.1.10` IP should be your server’s static LAN IP address. Make sure you have configured your servers network settings so it always has this IP (via static config or DHCP reservation).

### 5.2 Verify Network Configuration
- **Check IP**: 
  ```bash
  ip addr show
  ```
  Ensure the interface (e.g., `eth0` or `ens160`) displays the LAN IP you intend to use.
- **Set a Static IP** (if necessary):
  - On Ubuntu, typically you will configure this in `/etc/netplan/<config>.yaml`.
  - A sample Netplan configuration snippet might look like:
    ```yaml
    network:
      version: 2
      renderer: networkd
      ethernets:
        ens160:
          addresses:
            - 192.168.1.10/24
          gateway4: 192.168.1.1
          nameservers:
            addresses: [8.8.8.8, 8.8.4.4]
    ```
  - **Apply changes**:
    ```bash
    sudo netplan apply
    ```

---

## 6. Configure the Single Website

For a single-site setup, you can use Apache’s default document root (`/var/www/html`) instead of creating a new directory structure.

### 6.1 Place Your Website Files in `/var/www/html`
Remove or rename the default index file and replace it with your own:
```bash
sudo rm /var/www/html/index.html
echo "<html><body><h1>Hello LAN!</h1></body></html>" | sudo tee /var/www/html/index.html
```
> This ensures a flat structure. You can place additional HTML/PHP files directly under `/var/www/html`.

### 6.2 Adjust Permissions
To keep it simple:
```bash
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html
```
- `www-data` is the default Apache user and group on Ubuntu.

### 6.3 Update the Default Apache Configuration (Optional)
If you want to explicitly set your server name (helpful for logs), edit the default site:
```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```
Within `<VirtualHost *:80>`:
```apache
ServerName example.com
DocumentRoot /var/www/html
```
(You can still access the server by its IP from the LAN, even if you set `ServerName` to a domain.)

Save, exit, and reload:
```bash
sudo systemctl reload apache2
```

---

## 7. Testing from Other Devices on the LAN

### 7.1 Basic Connectivity Tests

1. **Ping the Server** from another LAN device:
   ```bash
   ping 192.168.1.10
   ```
   Replace `192.168.1.10` with your server’s LAN IP. If the ping returns replies, basic network connectivity is present.

2. **Check Firewall**:
   - If you can ping but **cannot** reach the website, ensure your firewall allows port 80 (and 443 if using SSL).
   - Confirm you have set up port forwarding or firewall allowances if your server is behind another router/firewall device.

### 7.2 Website Access Tests

1. **Web Browser**:
   - On a LAN computer, open a browser and go to:
     ```
     http://192.168.1.10
     ```
   - You should see the “Hello LAN!” page or whichever content you placed in `/var/www/html/index.html`.

2. **curl Command** (alternative CLI test):
   ```bash
   curl -I http://192.168.1.10
   ```
   This should return an HTTP 200 OK status along with server headers.

### 7.3 Common Troubleshooting Tips

- **No Response**: Check if Apache is running:
  ```bash
  sudo systemctl status apache2
  ```
- **Connection Refused**: Double-check your firewall rules, ensure port 80 is open.
- **Not Found / 404**: Confirm the file paths in `/var/www/html` match your requested URL.

---


## 8. Common Errors and Solutions

1. **Apache Not Starting / Failed to Start**
   - **Potential Causes**: Configuration syntax error, port conflict.
   - **Solution**:
     ```bash
     sudo apache2ctl configtest
     ```
     Resolve reported errors, then:
     ```bash
     sudo systemctl restart apache2
     ```

2. **403 Forbidden**
   - **Potential Causes**: Incorrect file permissions or directory ownership.
   - **Solution**:
     ```bash
     sudo chown -R www-data:www-data /var/www/html
     sudo chmod -R 755 /var/www/html
     ```

3. **404 Not Found**
   - **Potential Causes**: File doesn’t exist in `/var/www/html`, or you typed the wrong path/filename.
   - **Solution**:
     - Confirm the exact file name in `/var/www/html`.
     - Check your request URL.

4. **Firewall Blocking Traffic**
   - **Potential Causes**: Ports 80 or 443 not open, or only certain IP ranges allowed.
   - **Solution**: Re-check your ufw or iptables rules.

---

## 10. Conclusion
By following this guide, you have:

1. Installed and secured Apache on Ubuntu.
2. Configured the firewall to allow only necessary traffic.
3. (Optionally) bound Apache to a specific LAN IP.
4. Ensured a flat directory structure at `/var/www/html`.
5. Verified accessibility across your LAN with `ping`, `curl`, and a standard web browser.
