# Git Setup Guide for SPICE Simulation Environment

## Integrated with P4 Virtuoso Design Flow

---

## Table of Contents
1. [Overview](#1-overview)
2. [Git Installation](#2-git-installation)
3. [Initial Setup](#3-initial-setup)
4. [Directory Structure](#4-directory-structure)
5. [Git Ignore Rules](#5-git-ignore-rules)
6. [Symlinks to P4 Files](#6-symlinks-to-p4-files)
7. [What to Version](#7-what-to-version)
8. [Daily Workflow](#8-daily-workflow)
9. [Useful Commands](#9-useful-commands)
10. [Bash Aliases](#10-bash-aliases)
11. [Branching Strategy](#11-branching-strategy)
12. [Remote Repository](#12-remote-repository)
13. [Team Collaboration](#13-team-collaboration)
14. [Integration with P4](#14-integration-with-p4)
15. [Troubleshooting](#15-troubleshooting)

---

## 1. Overview

### Why Git for Simulation?

| Virtuoso (P4) | Simulation (Git) |
|---------------|------------------|
| Binary `.oa` files | Text `.sp` files |
| Exclusive locks needed | Merge-friendly |
| Large files | Small files |
| Commercial tool | Free & ubiquitous |

### Tool Separation

```
/home/UserName/work/ProjectName/
├── lib/          ← P4 (Virtuoso .oa binary)
├── cds/          ← P4 (Cadence config)
├── scripts/      ← P4 (SKILL)
└── sim/          ← Git (SPICE text files)
```

---

## 2. Git Installation

```bash
# Install Git
sudo apt update
sudo apt install git

# Verify
git --version
```

---

## 3. Initial Setup

### Configure identity

```bash
git config --global user.name "UserName"
git config --global user.email "UserName@example.com"
```

### Configure defaults

```bash
# Default branch name
git config --global init.defaultBranch main

# Default editor
git config --global core.editor vim

# Enable symlinks (important!)
git config --global core.symlinks true

# Line endings (for WSL)
git config --global core.autocrlf input

# Useful colors
git config --global color.ui auto
```

### Initialize repository

```bash
cd ~/work/ProjectName/sim
git init
```

---

## 4. Directory Structure

```
~/work/ProjectName/sim/
├── .git/                   # Git repository
├── .gitignore              # Ignore rules
├── README.md               # Documentation
├── Makefile                # Top-level make targets
├── common/                 # Shared files
│   ├── corners.inc
│   ├── supplies.inc
│   └── measurements.inc
├── hsim/                   # HSim simulations
│   ├── tb_bandgap/
│   │   ├── tb_bandgap.sp
│   │   ├── runscript.sh
│   │   ├── Makefile
│   │   └── netlist/        # Symlinks to P4
│   └── tb_ldo/
│       ├── tb_ldo.sp
│       ├── runscript.sh
│       └── netlist/
├── spectre/                # Spectre simulations
│   └── tb_opamp/
│       ├── tb_opamp.scs
│       └── runscript.sh
├── primesim/               # PrimeSim simulations
│   └── tb_pll/
│       ├── tb_pll.sp
│       └── runscript.sh
└── cosim/                  # Co-simulation
    └── tb_top_ams/
        ├── tb_top.sv
        ├── tb_analog.sp
        └── runscript.sh
```

---

## 5. Git Ignore Rules

Create `.gitignore`:

```bash
cat > ~/work/ProjectName/sim/.gitignore << 'EOF'
# ===== Simulation Results (Large, Regenerate) =====
*.raw
*.raw.gz
*.fsdb
*.vcd
*.vpd
*.wdb
*.shm/
*.trn/
psf/
*.psf

# ===== HSPICE/HSim Outputs =====
*.tr0
*.ac0
*.sw0
*.mt0
*.mt*
*.st0
*.st*
*.ic0
*.ic*
*.pa0
*.pa*
*.ma0
*.ms0
*.lis
*.log
*.out

# ===== Spectre Outputs =====
*.raw/
*.ahdlSimDB/
*.ahdlcmi/

# ===== PrimeSim Outputs =====
*.fsdb.gz
*.rc

# ===== Measurement Results =====
*.meas
*.measure
*.csv
*.dat

# ===== Run Directories =====
rundir*/
simout*/
output*/
results*/
work/

# ===== Waveform Viewer =====
*.session
*.window
*.virsim

# ===== Temporary Files =====
*.tmp
*.bak
*.swp
*~
\#*\#
.DS_Store

# ===== Core Dumps =====
core.*
vfastLog/

# ===== License Logs =====
*.lic_log

# ===== Don't ignore symlinks or netlists =====
# (symlinks to P4 are tracked)
!netlist/
EOF
```

Add to repository:

```bash
cd ~/work/ProjectName/sim
git add .gitignore
git commit -m "Initial commit: Added .gitignore"
```

---

## 6. Symlinks to P4 Files

### Why symlinks?

- Simulation needs netlists from Virtuoso (P4)
- Don't duplicate files
- Always use latest P4 version
- Git tracks the link, not the content

### Create symlinks

```bash
cd ~/work/ProjectName/sim/hsim/tb_bandgap

# Create netlist directory
mkdir -p netlist

# Link to P4-versioned CDL
ln -s ../../../../lib/ANALOG_LIB/bandgap/cdl/bandgap.cdl netlist/bandgap.cdl

# Link to extracted PEX netlist
ln -s ../../../../lib/ANALOG_LIB/bandgap/pex/bandgap.spf netlist/bandgap.spf

# Link to tech models (shared)
ln -s ../../../../tech/tsmc28/models models
```

### Verify symlinks

```bash
ls -la netlist/
# bandgap.cdl -> ../../../../lib/ANALOG_LIB/bandgap/cdl/bandgap.cdl

# Test link works
cat netlist/bandgap.cdl
```

### Add symlinks to Git

```bash
git add netlist/bandgap.cdl    # Adds symlink, not file content
git commit -m "Added netlist symlink to P4"
```

### Symlink setup script

Create `setup_links.sh`:

```bash
#!/bin/bash
# setup_links.sh - Create symlinks to P4 files
# Usage: ./setup_links.sh <testbench_dir> <cell_name> <lib_name>

TB_DIR=$1
CELL=$2
LIB=${3:-ANALOG_LIB}

P4_ROOT=~/work/ProjectName
LIB_ROOT=$P4_ROOT/lib
TECH_ROOT=$P4_ROOT/tech/tsmc28

# Create directories
mkdir -p $TB_DIR/netlist

# Link tech models
ln -sf $TECH_ROOT/models $TB_DIR/models

# Link CDL netlist
if [ -f "$LIB_ROOT/$LIB/$CELL/cdl/$CELL.cdl" ]; then
    ln -sf $LIB_ROOT/$LIB/$CELL/cdl/$CELL.cdl $TB_DIR/netlist/
    echo "Linked: $CELL.cdl"
fi

# Link PEX netlist
if [ -f "$LIB_ROOT/$LIB/$CELL/pex/$CELL.spf" ]; then
    ln -sf $LIB_ROOT/$LIB/$CELL/pex/$CELL.spf $TB_DIR/netlist/
    echo "Linked: $CELL.spf"
fi

echo "Setup complete for $TB_DIR"
```

Usage:

```bash
chmod +x setup_links.sh
./setup_links.sh hsim/tb_bandgap bandgap ANALOG_LIB
```

---

## 7. What to Version

| Version (YES) | Don't Version (NO) |
|---------------|-------------------|
| `.sp` testbench | `.raw` waveforms |
| `.scs` spectre | `.fsdb` dumps |
| `.spi` includes | `.tr0` results |
| `.spf` symlinks | `.log` files |
| `runscript.sh` | `.lis` outputs |
| `Makefile` | `psf/` directory |
| `.tcl` scripts | `rundir*/` |
| `corners.inc` | `*.mt0` measurements |
| `.ocean` scripts | `.csv` exports |
| `README.md` | |

---

## 8. Daily Workflow

### Morning (get latest)

```bash
# First sync P4 (netlists may have changed)
cd ~/work/ProjectName
p4 sync lib/... 

# Then pull Git
cd ~/work/ProjectName/sim
git pull
```

### During work

```bash
# Edit files freely
vim hsim/tb_bandgap/tb_bandgap.sp

# Check status anytime
git status
```

### End of day (commit)

```bash
cd ~/work/ProjectName/sim

# See what changed
git status

# Add changes
git add -A                    # Add all changes
# or
git add hsim/tb_bandgap/      # Add specific directory

# Commit
git commit -m "tb_bandgap: Added PVT corner sweep"

# Push to remote (if configured)
git push
```

### Quick commit

```bash
git add -A && git commit -m "Updated testbenches"
```

---

## 9. Useful Commands

### Basic commands

| Task | Command |
|------|---------|
| Check status | `git status` |
| Add files | `git add <file>` |
| Add all | `git add -A` |
| Commit | `git commit -m "message"` |
| View history | `git log` |
| View one-line history | `git log --oneline` |
| Show changes | `git diff` |
| Show staged changes | `git diff --staged` |

### File operations

| Task | Command |
|------|---------|
| Remove file | `git rm <file>` |
| Move/rename | `git mv <old> <new>` |
| Discard changes | `git checkout -- <file>` |
| Unstage file | `git reset HEAD <file>` |

### History & comparison

| Task | Command |
|------|---------|
| Show file history | `git log -p <file>` |
| Show who changed what | `git blame <file>` |
| Compare commits | `git diff <commit1> <commit2>` |
| Show specific commit | `git show <commit>` |

### Branching

| Task | Command |
|------|---------|
| List branches | `git branch` |
| Create branch | `git branch <name>` |
| Switch branch | `git checkout <name>` |
| Create & switch | `git checkout -b <name>` |
| Merge branch | `git merge <name>` |
| Delete branch | `git branch -d <name>` |

### Remote operations

| Task | Command |
|------|---------|
| Clone repo | `git clone <url>` |
| Pull changes | `git pull` |
| Push changes | `git push` |
| Show remotes | `git remote -v` |
| Add remote | `git remote add origin <url>` |

---

## 10. Bash Aliases

Add to `~/.bashrc`:

```bash
# ===== Git Shortcuts =====
alias gs='git status'
alias ga='git add'
alias gaa='git add -A'
alias gc='git commit -m'
alias gp='git push'
alias gl='git pull'
alias gd='git diff'
alias glog='git log --oneline -20'
alias gbr='git branch'
alias gco='git checkout'

# ===== Combined P4 + Git Workflow =====

# Morning sync everything
sync_all() {
    echo "=== Syncing P4 ==="
    cd ~/work/ProjectName
    p4 sync
    
    echo "=== Pulling Git ==="
    cd ~/work/ProjectName/sim
    git pull
    
    echo "=== Done ==="
}

# End of day commit everything
commit_all() {
    echo "=== P4 Status ==="
    cd ~/work/ProjectName
    p4 reconcile lib/... cds/... scripts/...
    p4 opened
    
    echo "=== Git Status ==="
    cd ~/work/ProjectName/sim
    git status
    
    read -p "Commit message: " msg
    
    read -p "Submit P4? (y/n) " p4yn
    if [ "$p4yn" = "y" ]; then
        cd ~/work/ProjectName
        p4 submit -d "$msg"
    fi
    
    read -p "Commit Git? (y/n) " gityn
    if [ "$gityn" = "y" ]; then
        cd ~/work/ProjectName/sim
        git add -A
        git commit -m "$msg"
        git push
    fi
}

# Quick simulation commit
simcommit() {
    cd ~/work/ProjectName/sim
    git add -A
    git status
    read -p "Commit? (y/n) " yn
    if [ "$yn" = "y" ]; then
        git commit -m "$1"
        git push
    fi
}

alias syncall='sync_all'
alias commitall='commit_all'
```

Usage:

```bash
source ~/.bashrc
syncall                              # Morning
simcommit "Added noise simulation"   # Quick commit
commitall                            # End of day (P4 + Git)
```

---

## 11. Branching Strategy

### Branch naming

```
main              # Stable, tape-out ready
├── dev           # Development integration
├── feature/xxx   # New features
├── corner/xxx    # Corner experiments  
└── debug/xxx     # Debug investigations
```

### Create feature branch

```bash
git checkout -b feature/bandgap-temp-sweep
# ... make changes ...
git add -A
git commit -m "Added temperature sweep -40 to 125C"
git push -u origin feature/bandgap-temp-sweep
```

### Merge to main

```bash
git checkout main
git merge feature/bandgap-temp-sweep
git push
git branch -d feature/bandgap-temp-sweep
```

### Branch for tape-out

```bash
git checkout -b release/v1.0
git tag -a v1.0 -m "Tape-out v1.0"
git push origin release/v1.0
git push origin v1.0
```

---

## 12. Remote Repository

### Option 1: GitHub/GitLab

```bash
# Create repo on GitHub/GitLab first, then:
cd ~/work/ProjectName/sim
git remote add origin git@github.com:UserName/ProjectName-sim.git
git push -u origin main
```

### Option 2: Bare repository on server

```bash
# On server
mkdir -p /data/git/ProjectName-sim.git
cd /data/git/ProjectName-sim.git
git init --bare

# On local
cd ~/work/ProjectName/sim
git remote add origin user@server:/data/git/ProjectName-sim.git
git push -u origin main
```

### Option 3: Local bare repository

```bash
# Create bare repo
mkdir -p ~/git-repos/ProjectName-sim.git
cd ~/git-repos/ProjectName-sim.git
git init --bare

# Add as remote
cd ~/work/ProjectName/sim
git remote add origin ~/git-repos/ProjectName-sim.git
git push -u origin main
```

---

## 13. Team Collaboration

### Clone repository

```bash
git clone git@github.com:UserName/ProjectName-sim.git
cd ProjectName-sim
./setup_links.sh hsim/tb_bandgap bandgap    # Recreate symlinks
```

### Pull before push

```bash
git pull --rebase
git push
```

### Resolve conflicts

```bash
git pull
# If conflict:
vim <conflicted_file>    # Edit to resolve
git add <conflicted_file>
git commit -m "Resolved merge conflict"
git push
```

### See who changed what

```bash
git blame hsim/tb_bandgap/tb_bandgap.sp
```

### Review teammate's changes

```bash
git log --oneline --author="teammate"
git show <commit_hash>
```

---

## 14. Integration with P4

### Directory separation

```
~/work/ProjectName/
├── lib/          ← P4 only (never git init here!)
├── cds/          ← P4 only
├── scripts/      ← P4 only  
└── sim/          ← Git only (never p4 add here!)
    └── .git/
```

### Symlink workflow

```
P4 (source)                     Git (consumer)
────────────                    ──────────────
lib/ANALOG/bandgap/cdl/   →     sim/hsim/tb_bandgap/netlist/
     bandgap.cdl          ←──symlink──  bandgap.cdl

1. Designer updates schematic in Virtuoso
2. Export CDL to lib/ANALOG/bandgap/cdl/
3. p4 submit
4. Simulation engineer: p4 sync
5. Symlink automatically points to new version
6. Run simulation
```

### Combined daily workflow

```
┌─────────────────────────────────────────────────────────────┐
│  Morning                                                     │
├─────────────────────────────────────────────────────────────┤
│  cd ~/work/ProjectName && p4 sync      # Get latest Virtuoso      │
│  cd sim && git pull              # Get latest simulations   │
├─────────────────────────────────────────────────────────────┤
│  During Day                                                  │
├─────────────────────────────────────────────────────────────┤
│  Edit Virtuoso    → lib/ files change   (P4 tracked)        │
│  Edit testbenches → sim/ files change   (Git tracked)       │
│  Symlinks connect them automatically                        │
├─────────────────────────────────────────────────────────────┤
│  End of Day                                                  │
├─────────────────────────────────────────────────────────────┤
│  cd ~/work/ProjectName                                            │
│  p4 reconcile lib/... && p4 submit -d "msg"                 │
│  cd sim && git add -A && git commit -m "msg" && git push    │
└─────────────────────────────────────────────────────────────┘
```

---

## 15. Troubleshooting

### Symlink broken

```bash
# Check if P4 synced
cd ~/work/ProjectName
p4 sync lib/...

# Verify link target exists
ls -la sim/hsim/tb_bandgap/netlist/

# Recreate if needed
rm sim/hsim/tb_bandgap/netlist/bandgap.cdl
ln -s ../../../../lib/ANALOG_LIB/bandgap/cdl/bandgap.cdl sim/hsim/tb_bandgap/netlist/
```

### Git not tracking symlinks

```bash
# Check git config
git config core.symlinks

# Should be true
git config core.symlinks true
```

### Accidentally added results

```bash
# Remove from git but keep file
git rm --cached *.raw
git commit -m "Removed results from tracking"

# Add to .gitignore
echo "*.raw" >> .gitignore
git add .gitignore
git commit -m "Updated gitignore"
```

### Undo last commit

```bash
# Keep changes, undo commit
git reset --soft HEAD~1

# Discard changes and commit
git reset --hard HEAD~1
```

### Discard all local changes

```bash
git checkout -- .
git clean -fd
```

### Large repository

```bash
# Check size
du -sh .git

# Clean up
git gc --aggressive --prune=now
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│  Git Quick Reference for Simulation                         │
├─────────────────────────────────────────────────────────────┤
│  Status:     git status / git diff                          │
│  Add:        git add -A                                     │
│  Commit:     git commit -m "message"                        │
│  Push:       git push                                       │
│  Pull:       git pull                                       │
│  History:    git log --oneline                              │
│  Branches:   git branch / git checkout -b <name>            │
├─────────────────────────────────────────────────────────────┤
│  Symlinks:   ln -s <p4_path> <git_path>                     │
│  P4 first:   p4 sync before running sim                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Example Testbench Template

### File: `hsim/tb_template/tb_template.sp`

```spice
* Testbench: tb_template
* Author: UserName
* Date: January 2026
* Git: ProjectName-sim/hsim/tb_template

.option ingold=2 probe post
.temp 25

* Include models (symlink to P4)
.include './models/corners.inc'

* Include DUT netlist (symlink to P4)
.include './netlist/dut.cdl'

* Supplies
.param vdd=0.8
v_vdd vdd 0 dc vdd
v_gnd gnd 0 dc 0

* Testbench circuit
* ... your testbench here ...

* Analysis
.tran 1n 10u

* Measurements
.meas tran t_rise trig v(out) val='vdd/2' rise=1 targ v(out) val='vdd*0.9' rise=1

* Control block
.control
run
wrdata results/sim.dat v(out) v(in)
.endc

.end
```

### File: `hsim/tb_template/Makefile`

```makefile
# Testbench Makefile
CELL = template
TB = tb_$(CELL)
SIM = hsim

# Paths
RESULT_DIR = results
NETLIST_DIR = netlist

# Simulator
HSIM = hsim64
HSIM_OPTS = -i $(TB).sp -o $(RESULT_DIR)/$(TB)

.PHONY: all sim clean view

all: sim

sim: $(RESULT_DIR)
	$(HSIM) $(HSIM_OPTS)

$(RESULT_DIR):
	mkdir -p $@

clean:
	rm -rf $(RESULT_DIR)

view:
	gwave $(RESULT_DIR)/$(TB).raw &
```

---

## Author

Generated from hands-on Git + P4 integration session for IC simulation workflow.

Date: January 2023
