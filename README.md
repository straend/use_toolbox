# 📦 Toolbox & Direnv Automation

This project provides a "Universal Environment" module for Fedora Atomic (Silverblue/Kinoite) and other Toolbox users.

It allows you to define your development environment entirely within your project folder.


## 🛠️ The Core Logic (Copy & Paste)

Place this function inside your ~/.config/direnv/direnvrc file. This is the engine that handles the automation.
``` Bash
# ~/.config/direnv/direnvrc

use_toolbox() {
  local container_name=$1
  local pkg_file=".toolbox-packages"
  local sentinel=".direnv/toolbox-timestamp"

  # 1. CREATE: Check if the container exists
  if ! toolbox list --containers | grep -q "\b${container_name}\b"; then
    echo "▶ Container '$container_name' not found. Creating..."
    toolbox create -c "$container_name" -y
  fi

  # 2. PROVISION: Sync packages if .toolbox-packages is newer than sentinel
  if [ -f "$pkg_file" ]; then
    if [ ! -f "$sentinel" ] || [ "$pkg_file" -nt "$sentinel" ]; then
      echo "▶ Syncing packages in '$container_name'..."
      local pkgs=$(grep -v '^#' "$pkg_file" | xargs)
      
      if [ -n "$pkgs" ]; then
        # This uses the container's internal DNF to install packages
        toolbox run -c "$container_name" sudo dnf install -y $pkgs
      fi

      mkdir -p .direnv && touch "$sentinel"
    fi
  fi

  # 3. ENTER: Swap host shell for toolbox shell
  if [ -z "$TOOLBOX_PATH" ]; then
    exec toolbox enter "$container_name"
  fi
}
```

# 📥 Installatio1. Host Setup

## 1. Install direnv on your host machine.
```Bash

# Fedora Atomic
rpm-ostree install direnv
## Reboot to apply
```
## 2. Shell Configuratio
Add the following to the end of your `~/.bashrc` to enable the automation:
```Bash

if command -v direnv &>/dev/null; then
    eval "$(direnv hook bash)"
fi
```
## 3.Register the Modulreate the directory and file if they don't exist, then paste the Core Logic into it:
```Bash

mkdir -p ~/.config/direnv
touch ~/.config/direnv/direnvrc
```

# 🚀 Usage (Per Project)
To use this in a new project, create the following two files in the project root:

`.toolbox-packages` List the DNF packages your project needs.
```Plaintext

# Compiler and tools
gcc
make
python3-devel
```
`.envrc` The configuration file that direnv reads.
```Bash

use_toolbox "my-project-name"
```

## Activate

The first time you cd into the directory, run:
```Bash

direnv allow
```

💡 How it Works

  Infrastructure as Code: Your project's system dependencies are tracked in Git via `.toolbox-packages.`

  Smart Provisioning: It uses a "sentinel" file in `.direnv/` to compare timestamps. It only runs `dnf install` if you actually change the package list.

  No Nested Shells: It uses `exec` to replace the host shell process, preventing a messy stack of "shells inside shells."

⚠️ Known Limitations

  Exiting: Since the script uses `exec`, typing `exit` will close the current terminal tab/window.

  Home Directory: Since Toolboxes share your `$HOME`, the `if command -v direnv` check in your `.bashrc` prevents errors inside the container where direnv might not be installed.
