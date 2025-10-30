# Local Mail System with aerc, mbsync, notmuch, and goimapnotify

## Overview

This setup provides a fast, local-first email system that syncs with Gmail. Instead of aerc connecting directly to IMAP (slow), mail is synced locally and indexed for instant access.

## Architecture

```
Gmail IMAP Server
      ↕
   mbsync (sync tool)
      ↕
~/Mail/gmail/ (local Maildir)
      ↕
   notmuch (search index)
      ↕
   aerc (mail client)
```

**Push notifications:**
```
Gmail IMAP IDLE → goimapnotify → triggers mbsync → notmuch indexes → aerc sees updates
```

**Bidirectional sync:**
- **Gmail → aerc**: goimapnotify detects changes → mbsync syncs → notmuch indexes
- **aerc → Gmail**: Flag changes trigger hook → mbsync pushes to Gmail

## Components

### 1. mbsync (isync)
**Purpose**: Syncs mail between Gmail IMAP and local Maildir storage

**Config**: `~/.mbsyncrc`
```ini
IMAPAccount gmail
Host imap.gmail.com
User terakael@gmail.com
PassCmd "secret-tool lookup aerc password"
TLSType IMAPS
TLSVersions +1.2
CertificateFile /etc/ssl/certs/ca-certificates.crt

IMAPStore gmail-remote
Account gmail

MaildirStore gmail-local
Path ~/Mail/gmail/
Inbox ~/Mail/gmail/INBOX
SubFolders Verbatim

Channel gmail-inbox
Far :gmail-remote:INBOX
Near :gmail-local:INBOX
Create Both
Expunge Both
SyncState *

Channel gmail-all
Far :gmail-remote:
Near :gmail-local:
Patterns * !"[Gmail]/All Mail" !"[Gmail]/Important" !"[Gmail]/Starred"
Create Both
Expunge Both
SyncState *

Group gmail
Channel gmail-inbox
Channel gmail-all
```

**Manual sync**: `mbsync -a` (syncs all accounts/folders)

### 2. notmuch
**Purpose**: Indexes mail for fast search and provides virtual folders via tags

**Config**: `~/.notmuch-config`
```ini
[database]
path=/home/dan/Mail

[user]
name=Dan
primary_email=terakael@gmail.com

[new]
tags=unread;inbox
ignore=

[search]
exclude_tags=deleted;spam

[maildir]
synchronize_flags=true

[crypto]
gpg_path=gpg
```

**Post-new hook**: `~/Mail/.notmuch/hooks/post-new` (executable)
```bash
#!/bin/bash

# Tag messages based on their folder location

# Remove inbox tag from all messages first (we'll re-add it selectively)
notmuch tag -inbox -- tag:new

# Tag based on folders
notmuch tag +personal -inbox -- tag:new and folder:gmail/Personal
notmuch tag +work -inbox -- tag:new and folder:gmail/Work
notmuch tag +receipts -inbox -- tag:new and folder:gmail/Receipts
notmuch tag +archive -inbox -- tag:new and folder:gmail/Archive
notmuch tag +sent -inbox -- tag:new and folder:gmail/Sent
notmuch tag +sent -inbox -- tag:new and folder:"gmail/[Gmail]/Sent Mail"
notmuch tag +trash -inbox -- tag:new and folder:gmail/Trash
notmuch tag +deleted -inbox -- tag:new and folder:"gmail/[Gmail]/Bin"
notmuch tag +spam -inbox -- tag:new and folder:"gmail/[Gmail]/Spam"
notmuch tag +draft -inbox -- tag:new and folder:gmail/Drafts
notmuch tag +draft -inbox -- tag:new and folder:"gmail/[Gmail]/Drafts"

# Re-add inbox tag only to actual INBOX messages
notmuch tag +inbox -- tag:new and folder:gmail/INBOX

# Remove the new tag
notmuch tag -new -- tag:new
```

**Manual index**: `notmuch new` (indexes new/changed mail)

### 3. goimapnotify
**Purpose**: Watches Gmail via IMAP IDLE and triggers sync on changes

**Config**: `~/.config/goimapnotify/gmail.json`
```json
{
  "host": "imap.gmail.com",
  "port": 993,
  "tls": true,
  "tlsOptions": {
    "rejectUnauthorized": true
  },
  "username": "terakael@gmail.com",
  "passwordCmd": "secret-tool lookup aerc password",
  "onNewMail": "mbsync gmail-inbox && notmuch new",
  "onNewMailPost": "",
  "boxes": [
    "INBOX"
  ]
}
```

**Systemd service**: `~/.config/systemd/user/goimapnotify@.service`
```ini
[Unit]
Description=IMAP IDLE notifications for %i
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/goimapnotify -conf %h/.config/goimapnotify/%i.json
Restart=always
RestartSec=30

[Install]
WantedBy=default.target
```

**Enable**: `systemctl --user enable --now goimapnotify@gmail.service`

### 4. aerc
**Purpose**: Mail client that reads from notmuch database

**Config**: `~/.config/aerc/accounts.conf`
```ini
[gmail]
source        = notmuch:///home/dan/Mail
outgoing      = smtp://terakael%40gmail.com@smtp.gmail.com:587
outgoing-cred-cmd = secret-tool lookup aerc password
default       = INBOX
from          = Dan <terakael@gmail.com>
query-map     = /home/dan/.config/aerc/gmail-query-map
```

**Query map**: `~/.config/aerc/gmail-query-map`
```
# Maps folder names to notmuch queries
INBOX=tag:inbox
Personal=tag:personal
Work=tag:work
Receipts=tag:receipts
Archive=tag:archive
Sent=tag:sent
Drafts=tag:draft
Trash=tag:trash
[Gmail]/Bin=tag:deleted
[Gmail]/Spam=tag:spam
[Gmail]/Sent Mail=tag:sent
[Gmail]/Drafts=tag:draft
[Gmail]/All Mail=*
```

**Flag sync hook** in `~/.config/aerc/aerc.conf` under `[hooks]`:
```ini
flag-changed=sh -c 'mbsync gmail-inbox >/dev/null 2>&1 &'
```

This triggers a background sync when you mark emails as read/unread in aerc.

## Setup Steps

### 1. Install packages
```bash
# Arch Linux
sudo pacman -S isync notmuch goimapnotify

# Other distros: check package names
```

### 2. Set up Gmail app password
Store in system keyring:
```bash
secret-tool store --label="Aerc Gmail Password" aerc password
# Enter password when prompted
```

### 3. Create directory structure
```bash
mkdir -p ~/Mail/gmail
mkdir -p ~/Mail/.notmuch/hooks
mkdir -p ~/.config/aerc
mkdir -p ~/.config/goimapnotify
mkdir -p ~/.config/systemd/user
```

### 4. Create configuration files
- Copy `~/.mbsyncrc` (see config above)
- Copy `~/.notmuch-config` (see config above)
- Copy `~/Mail/.notmuch/hooks/post-new` (see script above, make executable)
- Copy `~/.config/goimapnotify/gmail.json` (see config above)
- Copy `~/.config/systemd/user/goimapnotify@.service` (see config above)
- Update `~/.config/aerc/accounts.conf` (see config above)
- Create `~/.config/aerc/gmail-query-map` (see config above)
- Add flag-changed hook to `~/.config/aerc/aerc.conf` (see config above)

### 5. Initial sync
```bash
# Download all mail from Gmail (takes a while)
mbsync -a

# Build search index
notmuch new
```

### 6. Enable push notifications
```bash
systemctl --user daemon-reload
systemctl --user enable --now goimapnotify@gmail.service
```

### 7. Test
```bash
# Check service is running
systemctl --user status goimapnotify@gmail.service

# Start aerc
aerc
```

## How It Works

### Initial Startup
1. `mbsync -a` downloads all mail to `~/Mail/gmail/`
2. `notmuch new` indexes all messages and applies folder-based tags
3. aerc reads from notmuch database instantly (no IMAP connection)

### New Mail Arrives
1. Gmail receives new mail
2. goimapnotify (watching INBOX via IMAP IDLE) detects it within seconds
3. goimapnotify triggers: `mbsync gmail-inbox && notmuch new`
4. mbsync downloads new mail
5. notmuch indexes it (post-new hook applies tags)
6. aerc sees new mail immediately

### Flag Changes from aerc
1. User marks email as read in aerc
2. aerc updates notmuch database
3. notmuch writes flag to maildir file (`synchronize_flags=true`)
4. aerc's flag-changed hook triggers: `mbsync gmail-inbox &`
5. mbsync syncs maildir flags back to Gmail
6. Gmail shows email as read

### Flag Changes from Gmail Web
1. User marks email as read in Gmail
2. IMAP IDLE notification sent to goimapnotify
3. goimapnotify triggers: `mbsync gmail-inbox && notmuch new`
4. mbsync syncs flag changes to local maildir
5. notmuch reads updated flags from maildir
6. aerc reflects the change

## Unread Count Script

`~/.local/bin/gmail_unread_count`:
```bash
#!/bin/bash

set -e

# Query notmuch for unread messages in inbox
count=$(notmuch count tag:inbox and tag:unread 2>/dev/null || echo "0")

echo "$count"
```

This script is now instant (queries local notmuch database instead of IMAP).

## Troubleshooting

### Check if goimapnotify is working
```bash
journalctl --user -u goimapnotify@gmail.service -f
```

### Manual sync
```bash
mbsync -a && notmuch new
```

### Verify notmuch tags
```bash
notmuch search tag:inbox
notmuch count tag:unread
```

### Check aerc connection
```bash
aerc
# Should start instantly, no IMAP connection delay
```

### Re-tag all messages
If folder tags are wrong, re-run tagging:
```bash
notmuch tag +personal -inbox -- folder:gmail/Personal
notmuch tag +work -inbox -- folder:gmail/Work
# etc...
```

## Performance

- **aerc startup**: Instant (reads local database)
- **New mail notification**: 1-3 seconds (IMAP IDLE)
- **Flag sync aerc → Gmail**: 1-2 seconds (background)
- **Flag sync Gmail → aerc**: Next time new mail arrives
- **Unread count script**: Instant (local query)

## Storage

- Mail storage: `~/Mail/gmail/` (~3.4GB for 13,000 messages in this setup)
- Notmuch database: `~/Mail/.notmuch/` (~100MB)
- Total: ~3.5GB

Future syncs only download new/changed messages, so growth is incremental.

## Security

- Password stored in system keyring (secret-tool)
- Gmail app password (not main password)
- Mail files: 0600 permissions (user-only readable)
- Configuration files should have restrictive permissions

## Customization

### Add more folders to sync
Edit `~/.mbsyncrc` Patterns line

### Add more folders to aerc
1. Ensure they're synced (mbsync)
2. Add tagging rule to `post-new` hook
3. Add query map entry in `gmail-query-map`

### Change sync frequency
goimapnotify syncs immediately on new mail. No timer needed.

### Sync multiple accounts
1. Create separate sections in `~/.mbsyncrc`
2. Create separate notmuch query maps
3. Create separate goimapnotify configs
4. Enable multiple goimapnotify services: `goimapnotify@account2.service`
