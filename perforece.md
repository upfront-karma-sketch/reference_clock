# Perforce (Helix Core) Setup Guide for Cadence Virtuoso IC Design

## Table of Contents
1. [Server Installation](#1-server-installation)
2. [Server Configuration](#2-server-configuration)
3. [Client Setup](#3-client-setup)
4. [Workspace Configuration](#4-workspace-configuration)
5. [Directory Structure](#5-directory-structure)
6. [File Ignore Rules](#6-file-ignore-rules)
7. [File Type Mappings](#7-file-type-mappings)
8. [What to Version](#8-what-to-version)
9. [Daily Workflow](#9-daily-workflow)
10. [Useful Commands](#10-useful-commands)
11. [Bash Aliases](#11-bash-aliases)
12. [Team Collaboration](#12-team-collaboration)
13. [Branching for Tape-out](#13-branching-for-tape-out)
14. [Backup](#14-backup)

---

## 1. Server Installation

```bash
# Add Perforce repository
wget -qO - https://package.perforce.com/perforce.pubkey | sudo apt-key add -
echo "deb http://package.perforce.com/apt/ubuntu focal release" | sudo tee /etc/apt/sources.list.d/perforce.list

# Update and install
sudo apt update
sudo apt install helix-p4d helix-cli
```

---

## 2. Server Configuration

```bash
# Create data directory
sudo mkdir -p /opt/perforce/data
sudo chown -R $USER:$USER /opt/perforce/data

# Set environment
export P4ROOT=/opt/perforce/data
export P4PORT=1666

# Start server
p4d -r $P4ROOT -p $P4PORT -d

# Verify running
ps aux | grep p4d
```

---

## 3. Client Setup

Add to `~/.bashrc`:

```bash
export P4PORT=localhost:1666
export P4USER=UserName
export P4CLIENT=UserName-ProjectName
```

Then:

```bash
source ~/.bashrc

# Create user
p4 user -f UserName

# Set password
p4 passwd

# Verify connection
p4 info
```

---

## 4. Workspace Configuration

```bash
p4 client
```

Configure as:

```
Client: UserName-ProjectName
Owner:  UserName
Root:   /home/UserName/work/ProjectName
Options: allwrite clobber nocompress unlocked modtime normdir
SubmitOptions: submitunchanged
LineEnd: local

View:
    //depot/ProjectName/cds/...       //UserName-ProjectName/cds/...
    //depot/ProjectName/lib/...       //UserName-ProjectName/lib/...
    //depot/ProjectName/scripts/...   //UserName-ProjectName/scripts/...
    //depot/ProjectName/doc/...       //UserName-ProjectName/doc/...
    //depot/ProjectName/release/...   //UserName-ProjectName/release/...
    //depot/ProjectName/sim/*/tb_*/netlist/... //UserName-ProjectName/sim/*/tb_*/netlist/...
    //depot/ProjectName/sim/*/tb_*/run/*.sp    //UserName-ProjectName/sim/*/tb_*/run/*.sp
```

Key options:
- `allwrite` - Files always writable (no need for `p4 edit`)
- `clobber` - Allow `p4 sync` to overwrite modified files

---

## 5. Directory Structure

```
/home/UserName/work/ProjectName/
├── cds/                    # Cadence config (versioned)
│   ├── cds.lib
│   ├── .cdsinit
│   ├── display.drf
│   └── data.reg
├── lib/                    # Design libraries (versioned)
│   ├── ANALOG_LIB/
│   ├── DIGITAL_LIB/
│   ├── IO_LIB/
│   └── TOP_LIB/
├── tech/                   # PDK reference
│   └── tsmc28/
├── sim/                    # Simulation (partially versioned)
│   ├── hsim/
│   ├── spectre/
│   └── ams/
├── scripts/                # Scripts (versioned)
│   ├── skill/
│   ├── ocean/
│   └── shell/
├── doc/                    # Documentation (versioned)
└── release/                # Tape-out deliverables (versioned)
    ├── gds/
    ├── cdl/
    └── lef/
```

---

## 6. File Ignore Rules

Create `.p4ignore`:

```bash
cat > ~/work/ProjectName/.p4ignore << 'EOF'
# ===== Cadence Generated =====
*.log
*.log.*
*.LOGFILE
.nfs*
.cadence/
.pcell_cache/
*.cdslck*
*.pyc

# ===== OpenAccess Locks =====
*.cdslck
*.lck

# ===== Simulation Results (Large) =====
*.raw
*.fsdb
*.vcd
*.shm/
*.trn/
*.wdb
*.vpd
*.psf/
psf/

# ===== HSim/Spectre Outputs =====
*.mt*
*.st*
*.ac*
*.tr*
*.sw*
*.ic*
*.pa*
*.meas*
*.lis
simOutput*/
rundir*/

# ===== AMS Results =====
*.dat
*.csv
*.png
*.eps

# ===== Calibre/PVS =====
svdb/
*.rep
*.sum
*.gds.gz

# ===== Large Binaries =====
*.gds
*.oasis

# ===== Temporary =====
*.tmp
*.bak
*.swp
*~
core.*
EOF

p4 set P4IGNORE=.p4ignore
```

---

## 7. File Type Mappings

```bash
p4 typemap
```

Add:

```
TypeMap:
    binary+l //depot/....oa
    binary+l //depot/....gds
    binary+l //depot/....oasis
    binary   //depot/....png
    binary   //depot/....pdf
    text     //depot/....cdl
    text     //depot/....sp
    text     //depot/....spi
    text     //depot/....v
    text     //depot/....sv
    text     //depot/....vams
```

Note: `+l` means exclusive lock (only one person can edit at a time).

---

## 8. What to Version

| Version (YES) | Don't Version (NO) |
|---------------|-------------------|
| `.oa` cell views | `.raw` waveforms |
| `cds.lib` | `.fsdb` dumps |
| `.cdsinit` | `.log` files |
| `.sp` netlists | `psf/` directories |
| `.cdl` exports | `.cdslck` locks |
| SKILL scripts | `.pcell_cache/` |
| OCEAN scripts | `svdb/` |
| Testbench setup | Calibre run dirs |
| Final `.gds` (release only) | Intermediate `.gds` |
| Documentation | Core dumps |

---

## 9. Daily Workflow

### Morning (get latest)

```bash
cd ~/work/ProjectName
p4 sync
```

### During work

```bash
# With allwrite mode, just edit freely in Virtuoso
# No p4 commands needed
```

### End of day (commit)

```bash
cd ~/work/ProjectName

# Detect changes
p4 reconcile lib/... cds/... scripts/...

# Review
p4 opened
p4 diff

# Submit
p4 submit -d "LIB_A: Added bandgap cell, updated bias"
```

---

## 10. Useful Commands

| Task | Command |
|------|---------|
| Check connection | `p4 info` |
| Get latest | `p4 sync` |
| Detect changes | `p4 reconcile ...` |
| See pending | `p4 opened` |
| Submit | `p4 submit -d "message"` |
| View history | `p4 filelog <file>` |
| Compare versions | `p4 diff2 <file>#1 <file>#2` |
| See all changes | `p4 changes //depot/ProjectName/...` |
| Who has files open | `p4 opened -a` |
| Revert changes | `p4 revert <file>` |
| Lock file | `p4 lock <file>` |
| Add new file | `p4 add <file>` |
| Delete file | `p4 delete <file>` |

---

## 11. Bash Aliases

Add to `~/.bashrc`:

```bash
# P4 shortcuts
alias p4s='p4 sync'
alias p4r='p4 reconcile lib/... cds/... scripts/...'
alias p4o='p4 opened'
alias p4d='p4 diff'
alias p4c='p4 submit'
alias p4log='p4 changes -l //depot/ProjectName/...'
alias p4who='p4 opened -a'

# Quick commit function
p4commit() {
    p4 reconcile lib/... cds/... scripts/...
    p4 opened
    read -p "Submit? (y/n) " yn
    if [ "$yn" = "y" ]; then
        p4 submit -d "$1"
    fi
}
```

Usage:

```bash
source ~/.bashrc
p4commit "Updated bandgap schematic"
```

---

## 12. Team Collaboration

### See who is editing

```bash
p4 opened -a //depot/ProjectName/lib/...
```

### Lock critical cells before tape-out

```bash
p4 lock //depot/ProjectName/lib/TOP_LIB/chip_top/...
```

### Resolve conflicts

```bash
p4 sync
p4 resolve
```

---

## 13. Branching for Tape-out

### Create release branch

```bash
p4 integrate //depot/ProjectName/lib/... //depot/ProjectName_release/v1.0/lib/...
p4 submit -d "Release branch v1.0 for tape-out"
```

### Cherry-pick bug fixes

```bash
p4 integrate //depot/ProjectName/lib/ANALOG_LIB/bandgap/... //depot/ProjectName_release/v1.0/lib/ANALOG_LIB/bandgap/...
p4 submit -d "Backport bandgap fix to v1.0"
```

---

## 14. Backup

### Backup script

```bash
#!/bin/bash
# /opt/perforce/backup.sh
DATE=$(date +%Y%m%d)
p4 admin checkpoint
cp /opt/perforce/data/checkpoint.* /backup/p4/checkpoint_$DATE
```

### Add to cron

```bash
crontab -e
```

Add:

```
0 2 * * * /opt/perforce/backup.sh
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────┐
│  P4 Quick Reference for Virtuoso                    │
├─────────────────────────────────────────────────────┤
│  Morning:    p4 sync                                │
│  End of day: p4 reconcile lib/... → p4 submit      │
│  Check:      p4 opened / p4 info                    │
│  History:    p4 filelog <file>                      │
│  Team:       p4 opened -a (who has what open?)      │
└─────────────────────────────────────────────────────┘
```

---

## Virtuoso Integration (Optional)

For automatic checkout on edit, set environment:

```bash
export CDS_DM_SYSTEM=p4
```

Or use `allwrite` mode (recommended) and reconcile manually.

---

## Troubleshooting

### Server not running

```bash
# Check if running
ps aux | grep p4d

# Start server
export P4ROOT=/opt/perforce/data
export P4PORT=1666
p4d -r $P4ROOT -p $P4PORT -d
```

### Connection refused

```bash
# Verify P4PORT
echo $P4PORT

# Should be: localhost:1666
export P4PORT=localhost:1666
```

### Files not detected by reconcile

```bash
# Check .p4ignore
p4 set P4IGNORE

# Force add
p4 add -f <file>
```

### Wrong depot path

```bash
# Delete and recreate client
p4 client -d <clientname>
p4 client
```

---

## Author

Generated from hands-on P4 setup session for IC design workflow.

Date: January 2023
