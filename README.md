# SSH Chat via tmux (Multi-User, Password Protected)

This project creates a **shared SSH-based chat room** using `tmux`.

- Multiple users can log in **at the same time**
- Each user has their **own username and password**
- Everyone joins the **same persistent chat session**
- If one user disconnects, **others are not affected**
- Users are **auto-generated** with random passwords
- No SSH keys required (password-only login)

Works great on fresh Ubuntu/Debian VPSes (Linode, DigitalOcean, etc.).

---

## How it works (high level)

- Ansible creates **N Linux users** (`chat01`, `chat02`, etc.)
- All users belong to a shared group (`sshchat`)
- SSH is configured so that users in that group:
    - are forced into a `tmux` session
    - cannot get a shell
    - authenticate **only via password**
- The tmux session hosts a simple shared chat UI
- Messages are logged and tagged with the username

---

## Requirements (on your local machine)

- Ansible 2.14+
- SSH access to the server as `root` (or a sudo user)
- Linux/macOS recommended (WSL works)

---

## Requirements (on the server)

- Ubuntu / Debian (fresh install recommended)
- OpenSSH (usually already installed)

The playbook installs everything else automatically.

---

## Files

```
ssh_chat.yml# The Ansible playbook
sshchat_credentials.txt# Auto-generated after run (local file)

```

---

## Usage

### 1) Clone or copy the playbook

```bash
gitclone <repo> ssh_chat
cd ssh_chat

```

Or just copy `ssh_chat.yml` into an empty directory.

---

### 2) Run the playbook

```bash
ansible-playbook -i"SERVER_IP," -u root ssh_chat.yml

```

You will be prompted for:

```
How many chat userstocreate?  →  e.g.5
Username prefix (e.g. chat)     →  press Enterfordefault

```

Example output users:

```
chat01
chat02
chat03
chat04
chat05

```

---

### 3) Get credentials

After the run finishes:

- Credentials are printed in the terminal
- They are also saved locally to:

```
./sshchat_credentials.txt

```

Example:

```
SSH Chat Credentials
====================
Server: 203.0.113.10

chat01 : X7mK9P3sFq2LQeA1
chat02 : JF8xN5A2KzQm9E6P

```

⚠️ **Treat this file like a password vault.**

It is created with permissions `0600`.

---

## How users join the chat

Each user connects using SSH:

```bash
ssh chat01@SERVER_IP
ssh chat02@SERVER_IP

```

They will be:

- asked for their password
- automatically attached to the shared tmux chat

---

## Chat behavior

- Everyone sees the same chat
- Messages are tagged with the username
- Disconnecting (`Ctrl+C`, network drop, etc.) **does not affect others**
- The tmux session stays alive as long as the server is running

---

## Security notes

- Password authentication is **enabled only** for chat users
- No SSH keys are allowed for chat users
- Chat users:
    - cannot port-forward
    - cannot run commands
    - cannot escape tmux
- Root/admin SSH behavior is untouched

```python
ssh-copy-id [root@](mailto:root@172.105.135.117)Server_IP
```

```python
ssh [root@](mailto:root@172.105.135.117)Server_IP
```

```python
sudo apt update
```

---

```yaml
---
- name: SSH chat via tmux (multi-user, password protected, auto-generated creds)
  hosts: all
  become: true
  gather_facts: true

  vars_prompt:
    - name: chat_user_count
      prompt: "How many chat users to create?"
      private: false
    - name: chat_user_prefix
      prompt: "Username prefix (e.g. chat)"
      private: false
      default: "chat"

  vars:
    chat_group: sshchat
    chat_session: sshchat
    chat_script: /usr/local/bin/ssh-chat

    chat_dir: /var/lib/sshchat
    chat_log: /var/lib/sshchat/chat.log
    chat_err: /var/lib/sshchat/chat.err

    sshd_dropin_dir: /etc/ssh/sshd_config.d
    sshd_dropin_file: /etc/ssh/sshd_config.d/99-sshchat.conf

    # Password generation settings
    chat_password_length: 16
    chat_password_chars: "ascii_letters,digits"

    # Where to save creds on your Ansible machine
    creds_outfile: "./sshchat_credentials.txt"

  pre_tasks:
    - name: Validate user count
      ansible.builtin.assert:
        that:
          - (chat_user_count | int) >= 1
          - (chat_user_count | int) <= 200
        fail_msg: "chat_user_count must be between 1 and 200"

    - name: Build user list (prefix01..prefixNN)
      ansible.builtin.set_fact:
        chat_users: >-
          {{
            query('sequence', 'start=1 end=' ~ (chat_user_count | int) ~ ' format=%02d')
            | map('regex_replace', '^', chat_user_prefix)
            | list
          }}

    - name: Generate passwords for each user (on controller)
      ansible.builtin.set_fact:
        chat_passwords: "{{ chat_passwords | default({}) | combine({ item: lookup('password', '/dev/null length=' ~ chat_password_length ~ ' chars=' ~ chat_password_chars) }) }}"
      loop: "{{ chat_users }}"
      no_log: true

  tasks:
    - name: Install tmux, openssh-server, openssl
      ansible.builtin.package:
        name:
          - tmux
          - openssh-server
          - openssl
        state: present

    - name: Ensure sshd drop-in directory exists
      ansible.builtin.file:
        path: "{{ sshd_dropin_dir }}"
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Ensure chat group exists
      ansible.builtin.group:
        name: "{{ chat_group }}"
        state: present

    - name: Create chat users and add to group
      ansible.builtin.user:
        name: "{{ item }}"
        shell: /bin/bash
        create_home: true
        groups: "{{ chat_group }}"
        append: true
      loop: "{{ chat_users }}"

    - name: Ensure chat accounts are usable (unlock + not expired)
      ansible.builtin.shell: |
        set -e
        passwd -u {{ item }} 2>/dev/null || true
        chage -E -1 -I -1 -m 0 -M 99999 {{ item }} 2>/dev/null || true
      changed_when: false
      failed_when: false
      loop: "{{ chat_users }}"

    - name: Set passwords (SHA-512 hash, robust)
      ansible.builtin.shell: |
        set -euo pipefail
        HASH="$(openssl passwd -6 '{{ chat_passwords[item] }}')"
        usermod --password "$HASH" '{{ item }}'
      args:
        executable: /bin/bash
      loop: "{{ chat_users }}"
      no_log: true

    - name: Ensure chat directory exists (shared, group-writable, setgid)
      ansible.builtin.file:
        path: "{{ chat_dir }}"
        state: directory
        owner: root
        group: "{{ chat_group }}"
        mode: "2775"

    - name: Ensure chat log files exist (shared)
      ansible.builtin.file:
        path: "{{ item }}"
        state: touch
        owner: root
        group: "{{ chat_group }}"
        mode: "0664"
      loop:
        - "{{ chat_log }}"
        - "{{ chat_err }}"

    - name: Install forced chat command
      ansible.builtin.copy:
        dest: "{{ chat_script }}"
        owner: root
        group: root
        mode: "0755"
        content: |
          #!/usr/bin/env bash
          set -e

          SESSION="{{ chat_session }}"
          LOG="{{ chat_log }}"
          ERR="{{ chat_err }}"
          CHATDIR="{{ chat_dir }}"
          export TERM="${TERM:-xterm-256color}"

          if ! command -v tmux >/dev/null 2>&1; then
            echo "tmux not installed."
            exit 1
          fi

          mkdir -p "$CHATDIR"
          touch "$LOG" "$ERR" || true

          if ! tmux has-session -t "$SESSION" 2>/dev/null; then
            tmux new-session -d -s "$SESSION" 2>>"$ERR" || {
              echo "Failed to start tmux. See $ERR"
              exit 1
            }
            tmux send-keys -t "$SESSION":0.0 "tail -n 200 -f '$LOG'" C-m
            tmux split-window -v -t "$SESSION":0 2>>"$ERR"
            tmux resize-pane -t "$SESSION":0.1 -y 6 2>>"$ERR" || true
            tmux send-keys -t "$SESSION":0.1 \
              "bash -lc 'while true; do printf \"> \"; IFS= read -r msg || exit; ts=\$(date \"+%Y-%m-%d %H:%M:%S\"); printf \"%s [%s] %s\n\" \"\$ts\" \"\$USER\" \"\$msg\" >> \"{{ chat_log }}\"; done'" \
              C-m
          fi

          exec tmux attach-session -t "$SESSION"

    - name: Configure sshd for chat group (password only, no keys)
      ansible.builtin.copy:
        dest: "{{ sshd_dropin_file }}"
        owner: root
        group: root
        mode: "0644"
        content: |
          Match Group {{ chat_group }}
              ForceCommand {{ chat_script }}
              PermitTTY yes

              PasswordAuthentication yes
              KbdInteractiveAuthentication no
              PubkeyAuthentication no

              X11Forwarding no
              AllowTcpForwarding no
              AllowAgentForwarding no
              PermitTunnel no
              GatewayPorts no
      register: chat_dropin

    - name: Validate sshd config
      ansible.builtin.command: sshd -t
      changed_when: false
      when: chat_dropin.changed

    - name: Restart SSH service
      ansible.builtin.service:
        name: "{{ 'ssh' if ansible_facts.os_family == 'Debian' else 'sshd' }}"
        state: restarted
      when: chat_dropin.changed

  post_tasks:
    - name: Build credentials text (for display + saving locally)
      ansible.builtin.set_fact:
        creds_text: |
          SSH Chat Credentials
          ====================
          Server: {{ inventory_hostname }}

          {% for u in chat_users %}
          {{ u }} : {{ chat_passwords[u] }}
          {% endfor %}
      no_log: true

    - name: Save credentials to local file (controller)
      ansible.builtin.copy:
        dest: "{{ creds_outfile }}"
        content: "{{ creds_text }}"
        mode: "0600"
      delegate_to: localhost
      run_once: true
      no_log: true

    - name: Show credentials (appears in terminal output)
      ansible.builtin.debug:
        msg: "{{ creds_text.split('\n') }}"
      no_log: false

```

### How to use

```bash
ansible-playbook -i "SERVER_IP," -u root ssh_chat.yml

```

Then users can join with:

```bash
ssh chat01@SERVER_IP
ssh chat02@SERVER_IP
...

```
