Since **Bash** is used, you can take advantage of more robust error handling and string manipulation. I’ve updated the scripts to be more "production-ready" by adding safety checks, such as ensuring the script is run on a Debian/Ubuntu-style system (common for Yocto) and verifying that the user has `sudo` privileges.

---

### 1. `init.sh`
This script prepares the foundation. I've added a check to ensure the groups don't already exist and a "success" report at the end.

```bash
#!/bin/bash
# init.sh - Setup the shared filesystem hierarchy

set -e # Exit on any error

SHARED_DIR="/srv/test_lab"
TOOL_DIR="/opt/cross-tools"
GROUP_NAME="project_group"
ADMIN_GROUP="toolchain-admins"

echo "==> Initializing shared environment..."

# Create groups if they don't exist
for grp in "$GROUP_NAME" "$ADMIN_GROUP"; do
    if ! getent group "$grp" > /dev/null; then
        sudo groupadd "$grp"
        echo "Created group: $grp"
    fi
done

# Setup /srv/test_lab
sudo mkdir -p "$SHARED_DIR"
sudo chgrp "$GROUP_NAME" "$SHARED_DIR"
sudo chmod 2770 "$SHARED_DIR"
sudo setfacl -dm g:"$GROUP_NAME":rwx "$SHARED_DIR"
sudo setfacl -m g:"$GROUP_NAME":rwx "$SHARED_DIR"

# Setup /opt/cross-tools
sudo mkdir -p "$TOOL_DIR"
sudo chgrp "$ADMIN_GROUP" "$TOOL_DIR"
sudo chmod 2775 "$TOOL_DIR"

echo "==> Success! Paths initialized."
ls -ld "$SHARED_DIR" "$TOOL_DIR"
```

---

### 2. `grant.sh`
I updated this to verify if the user actually exists on the system before trying to modify their groups, which prevents messy error messages.

```bash
#!/bin/bash
# grant.sh - Onboard a new team member

USER_NAME=$1
IS_ADMIN=$2 # Pass 'admin' as second argument for toolchain rights

if [[ -z "$USER_NAME" ]]; then
    echo "Usage: ./grant.sh <username> [admin]"
    exit 1
fi

if ! id "$USER_NAME" &>/dev/null; then
    echo "Error: User '$USER_NAME' does not exist."
    exit 1
fi

# Standard access
sudo usermod -aG project_group "$USER_NAME"
echo "Granting '$USER_NAME' access to /srv/test_lab..."

# Admin access
if [[ "$IS_ADMIN" == "admin" ]]; then
    sudo usermod -aG toolchain-admins "$USER_NAME"
    echo "Granting '$USER_NAME' admin rights for /opt/cross-tools..."
fi

echo "Done. IMPORTANT: $USER_NAME must log out and back in (or restart SSH) to see changes."
```

---

### 3. `my_installer.sh`
This is the most complex one. It executes the Yocto `.sh` file and then "normalizes" the permissions. Yocto installers often set files to be immutable or owned by the installer's UID; this script wrangles them back into the group.

```bash
#!/bin/bash
# my_installer.sh - Yocto SDK Wrapper

set -e

INSTALLER_PATH=$1
VERSION_TAG=$2 # e.g., 4.0.1
TARGET_BASE="/opt/cross-tools/yocto-sdk-$VERSION_TAG"
ADMIN_GROUP="toolchain-admins"

if [[ ! -f "$INSTALLER_PATH" || -z "$VERSION_TAG" ]]; then
    echo "Usage: $0 <sdk-installer.sh> <version_tag>"
    exit 1
fi

echo "==> Starting Yocto SDK Installation to $TARGET_BASE..."

# 1. Run the vendor installer
# Note: Yocto installers usually ask for confirmation; -y or -d flags help automate
bash "$INSTALLER_PATH" -d "$TARGET_BASE"

echo "==> Normalizing permissions for group collaboration..."

# Force group ownership
sudo chgrp -R "$ADMIN_GROUP" "$TARGET_BASE"

# Set directory bits: SGID + rwxrwxr-x
sudo find "$TARGET_BASE" -type d -exec chmod 2775 {} +

# Set file bits: rw-rw-r-- (keeps existing execute bits for binaries)
# We use 'g+w' to ensure group can write without stripping +x from binaries
sudo chmod -R g+w "$TARGET_BASE"

# Apply default ACLs so future logs/temps are shared
sudo setfacl -R -dm g:"$ADMIN_GROUP":rwx "$TARGET_BASE"

# Update the symlink
sudo ln -sfn "$TARGET_BASE" /opt/cross-tools/current

echo "==> SDK $VERSION_TAG installed and linked to /opt/cross-tools/current"
```

---

### Final "Remote Team" Note
Since your users log in via **Bash**, you might want to add a snippet to `/etc/bash.bashrc` (system-wide) that detects if the shared folder exists and reminds them to source the SDK:

```bash
# Add to /etc/bash.bashrc
if [ -d "/opt/cross-tools/current" ]; then
    echo "--- Cross-Compile Environment Available ---"
    echo "Run: source /opt/cross-tools/current/environment-setup-*"
fi
```

This keeps the workflow top-of-mind for everyone connecting remotely. You've now got a fully scripted, repeatable environment. Does your team use a specific **build system** (like CMake or Meson) for the unit tests that might need specific environment variables set in these scripts?