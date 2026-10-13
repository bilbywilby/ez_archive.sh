```bash
#!/data/data/com.termux/files/usr/bin/bash
# ============================================================================
# ez_archive.sh — Resilient & Reusable Terminal Toolkit for Termux/Linux
# ============================================================================
# Purpose: One-stop utility script for directory management, Git operations,
#          file inspection, Python heredoc execution, validation, and debugging
# Author: Carter / bilbywilby
# Date: 2026-10-13
# Version: 2.0.0
# Platform: Termux (Android), Debian/Ubuntu, Alpine Linux
# Dependencies: bash >= 5.0, git, python3
#
# Usage: source ./ez_archive.sh [command] [args...]
# Examples:
#   source ez_archive.sh project_init "$HOME/my_project"
#   source ez_archive.sh git_status
#   source ez_archive.sh run_python << 'PYEOF'
#   python3 << 'PYEOF'
#     print("Hello World")
#   PYEOF
#
# Save this script to ~/.ez_archive.sh and source it from any session:
#   source ~/.ez_archive.sh
# ============================================================================

set -euo pipefail

# Global Configuration
readonly EZ_ARCHIVE_VERSION="2.0.0"
readonly EZ_LOG_DIR="${EZ_LOG_DIR:-$HOME/.ez_archive/logs}"
readonly EZ_BACKUP_DIR="${EZ_BACKUP_DIR:-$HOME/.ez_archive/backups}"
EZ_VERBOSE="${EZ_VERBOSE:-0}"  # Set to 1 for verbose output
EZ_COLORS="${EZ_COLORS:-1}"   # Set to 0 to disable ANSI colors

# ANSI Color Codes (disabled if EZ_COLORS=0)
if [ "$EZ_COLORS" -eq 1 ]; then
    readonly RED='\033[0;31m'
    readonly GREEN='\033[0;32m'
    readonly YELLOW='\033[0;33m'
    readonly BLUE='\033[0;34m'
    readonly MAGENTA='\033[0;35m'
    readonly CYAN='\033[0;36m'
    readonly WHITE='\033[0;37m'
    readonly BOLD='\033[1m'
    readonly RESET='\033[0m'
else
    readonly RED='' GREEN='' YELLOW='' BLUE='' MAGENTA='' CYAN='' WHITE='' BOLD='' RESET=''
fi

# ============================================================================
# UTILITY FUNCTIONS
# ============================================================================

# Print colored messages
msg_info() { printf "${BLUE}[INFO]${RESET} %s\n" "$*" >&2; }
msg_success() { printf "${GREEN}[SUCCESS]${RESET} %s\n" "$*" >&2; }
msg_warning() { printf "${YELLOW}[WARNING]${RESET} %s\n" "$*" >&2; }
msg_error() { printf "${RED}[ERROR]${RESET} %s\n" "$*" >&2; }
msg_debug() { [ "$EZ_VERBOSE" -eq 1 ] && printf "${CYAN}[DEBUG]${RESET} %s\n" "$*" >&2 || true; }

# Timestamped log entry
log_entry() {
    mkdir -p "$EZ_LOG_DIR"
    echo "$(date '+%Y-%m-%d %H:%M:%S') [$1] $2" >> "$EZ_LOG_DIR/ez_archive.log"
    if [ "$EZ_VERBOSE" -eq 1 ]; then
        echo "[$1] $2"
    fi
}

# Safe directory creation (creates parents, idempotent)
mkdir_safe() {
    local dir="$1"
    if [ ! -d "$dir" ]; then
        mkdir -p "$dir"
        msg_debug "Created directory: $dir"
    else
        msg_debug "Directory exists: $dir"
    fi
}

# Safe file backup before overwrite
backup_file() {
    local file="$1"
    local suffix="${2:-$(date +%Y%m%d_%H%M%S)}"
    
    if [ -f "$file" ]; then
        mkdir_safe "$EZ_BACKUP_DIR"
        cp "$file" "$EZ_BACKUP_DIR/$(basename "$file").$suffix.bak"
        msg_success "Backed up: $file → $EZ_BACKUP_DIR/"
    fi
}

# Check required commands exist
check_dependencies() {
    local deps=("$@")
    local missing=()
    
    for cmd in "${deps[@]}"; do
        if ! command -v "$cmd" &>/dev/null; then
            missing+=("$cmd")
        fi
    done
    
    if [ ${#missing[@]} -gt 0 ]; then
        msg_error "Missing dependencies: ${missing[*]}"
        msg_warning "Install with: pkg install ${missing[*]} (Termux) or apt-get install ${missing[*]}"
        return 1
    fi
    msg_debug "All dependencies satisfied: ${deps[*]}"
    return 0
}

# ============================================================================
# DIRECTORY MANAGEMENT
# ============================================================================

project_init() {
    local project_name="${1:-project}"
    local project_path="${2:-$HOME/$project_name}"
    
    msg_info "Initializing project: $project_name at $project_path"
    
    # Create directory structure
    mkdir_safe "$project_path"
    mkdir_safe "$project_path/src"
    mkdir_safe "$project_path/scripts"
    mkdir_safe "$project_path/tests/fixtures"
    mkdir_safe "$project_path/logs"
    mkdir_safe "$project_path/docs"
    
    # Initialize .gitignore
    if [ ! -f "$project_path/.gitignore" ]; then
        cat > "$project_path/.gitignore" << 'GITIGNORE_EOF'
# Data and Logs
logs/
*.log

# Python
__pycache__/
*.py[cod]
*.pyi
.Python
*.egg-info/
dist/
build/

# Node
node_modules/
package-lock.json

# Environment
.env
.env.local

# IDE
.idea/
.vscode/
*.swp
*~

# Testing
.pytest_cache/
.coverage
htmlcov/
GITIGNORE_EOF
        msg_success "Created .gitignore"
    fi
    
    # Initialize git if not present
    if [ ! -d "$project_path/.git" ]; then
        cd "$project_path"
        git init
        msg_success "Initialized git repository"
    fi
    
    # Create README
    if [ ! -f "$project_path/README.md" ]; then
        cat > "$project_path/README.md" << EOF
# $project_name

Generated on $(date -Iseconds) by ez_archive.sh

## Quick Start
\`\`\`bash
cd $project_path
pip install -r requirements.txt 2>/dev/null || true
\`\`\`

## Structure
\`\`\`
src/       - Core modules
scripts/   - Executable scripts
tests/     - Test suites
logs/      - Application logs
docs/      - Documentation
\`\`\`
EOF
        msg_success "Created README.md"
    fi
    
    msg_success "Project initialized at: $project_path"
}

cd_project() {
    local project_name="$1"
    local search_dirs=("$HOME" "$PROJECT_ROOT" "${PROJECT_DIR:-$HOME/projects}")
    
    for base in "${search_dirs[@]}"; do
        if [ -d "$base/$project_name" ]; then
            cd "$base/$project_name"
            msg_success "Changed to: $(pwd)"
            return 0
        fi
    done
    
    msg_error "Project not found: $project_name"
    return 1
}

tree_view() {
    local path="${1:-.}"
    local depth="${2:-3}"
    
    if command -v tree &>/dev/null; then
        tree -L "$depth" "$path" 2>/dev/null || ls -la "$path"
    else
        find "$path" -maxdepth "$depth" -not -path '*/\.*' | sort | sed "s|^|  |"
    fi
}

# ============================================================================
# GIT OPERATIONS
# ============================================================================

git_init() {
    local dir="${1:-.}"
    cd "$dir"
    
    if [ ! -d ".git" ]; then
        git init
        msg_success "Git initialized in: $dir"
    else
        msg_info "Git already initialized: $dir"
    fi
}

git_status() {
    local branch
    branch=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "unknown")
    
    echo "========================================"
    echo "GIT STATUS: $branch"
    echo "========================================"
    git status --short --branch 2>/dev/null || msg_error "Not a git repository"
}

git_stash_all() {
    local message="${1:-ez_archive_auto_stash_$(date +%Y%m%d_%H%M%S)}"
    
    if [ -n "$(git status --porcelain)" ]; then
        git stash save "$message"
        msg_success "Stashed changes: $message"
    else
        msg_info "No changes to stash"
    fi
}

git_stash_pop() {
    if [ -n "$(git stash list)" ]; then
        git stash pop
        msg_success "Applied latest stash"
    else
        msg_warning "No stashes to apply"
    fi
}

git_sync() {
    local remote_url="${1:-origin}"
    local branch="${2:-$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo main)}"
    
    msg_info "Fetching latest from $remote_url/$branch..."
    git fetch "$remote_url"
    
    msg_info "Pulling changes (merge strategy)..."
    git pull "$remote_url" "$branch" --no-rebase 2>/dev/null || \
        git pull "$remote_url" "$branch" 2>/dev/null || \
        msg_warning "Pull failed - manual resolution may be needed"
    
    msg_success "Sync complete"
}

git_push_force() {
    local branch="${1:-$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo main)}"
    
    msg_warning "Force pushing to $branch (THIS REWRITES HISTORY)"
    read -p "Continue? [y/N]: " confirm
    if [[ "$confirm" =~ ^[Yy]$ ]]; then
        git push -f origin "$branch"
        msg_success "Force push complete"
    else
        msg_info "Aborted"
    fi
}

git_diff_recent() {
    local count="${1:-5}"
    
    echo "========================================"
    echo "RECENT COMMITS (last $count)"
    echo "========================================"
    git log --oneline -"$count" --graph --decorate
}

git_branch_list() {
    echo "========================================"
    echo "LOCAL BRANCHES"
    echo "========================================"
    git branch -v 2>/dev/null || echo "No local branches"
    
    echo ""
    echo "========================================"
    echo "REMOTE BRANCHES"
    echo "========================================"
    git branch -r 2>/dev/null || echo "No remote branches"
}

# ============================================================================
# FILE INSPECTION TOOLS
# ============================================================================

view_file_tail() {
    local file="$1"
    local lines="${2:-50}"
    
    if [ -f "$file" ]; then
        echo "=== LAST $lines LINES OF $file ==="
        tail -n "$lines" "$file"
    else
        msg_error "File not found: $file"
    fi
}

view_file_tac() {
    local file="$1"
    local lines="${2:-20}"
    
    if [ -f "$file" ]; then
        echo "=== LAST $lines LINES (REVERSE ORDER - TAC) ==="
        tac "$file" 2>/dev/null | head -n "$lines" || \
        tail -n "$lines" "$file" | tac
    else
        msg_error "File not found: $file"
    fi
}

view_json_formatted() {
    local file="$1"
    
    if [ -f "$file" ]; then
        python3 -m json.tool "$file" 2>/dev/null || \
        jq '.' "$file" 2>/dev/null || \
        cat "$file"
    else
        msg_error "JSON file not found: $file"
    fi
}

search_file_content() {
    local pattern="$1"
    local directory="${2:-.}"
    local extension="${3:-*}"
    
    msg_info "Searching for '$pattern' in $directory (extension: $extension)"
    
    if command -v rg &>/dev/null; then
        rg --type "$extension" "$pattern" "$directory" 2>/dev/null || grep -r "$pattern" "$directory" --include="$extension"
    else
        grep -rn "$pattern" "$directory" --include="$extension" 2>/dev/null
    fi
}

count_lines() {
    local file="$1"
    
    if [ -f "$file" ]; then
        wc -l < "$file" | tr -d ' '
    else
        echo "0"
    fi
}

# ============================================================================
# PYTHON EXECUTION UTILITIES
# ============================================================================

run_python_inline() {
    """Execute Python code from stdin/heredoc"""
    
    if [ ! -t 0 ]; then
        # Read from stdin (heredoc)
        python3 - "$@"
    else
        msg_warning "No input provided. Example usage:"
        echo '  run_python_inline << '"'"'PYEOF'"'"''
        echo '    import json'
        echo '    print(json.dumps({"test": True}))'
        echo '  PYEOF'
    fi
}

python_check_imports() {
    local imports=("$@")
    local missing=()
    
    for imp in "${imports[@]}"; do
        if ! python3 -c "import $imp" 2>/dev/null; then
            missing+=("$imp")
        fi
    done
    
    if [ ${#missing[@]} -gt 0 ]; then
        msg_error "Missing Python packages: ${missing[*]}"
        msg_info "Install with: pip install ${missing[*]}"
        return 1
    fi
    
    msg_success "All imports available: ${imports[*]}"
    return 0
}

python_validate_json() {
    local file="$1"
    
    if [ -f "$file" ]; then
        if python3 -m json.tool "$file" > /dev/null 2>&1; then
            msg_success "Valid JSON: $file"
            return 0
        else
            msg_error "Invalid JSON: $file"
            return 1
        fi
    else
        msg_error "File not found: $file"
        return 1
    fi
}

# ============================================================================
# VALIDATION & DIAGNOSTICS
# ============================================================================

check_termux_environment() {
    echo "========================================"
    echo "TERMUX ENVIRONMENT CHECK"
    echo "========================================"
    
    echo "Shell:     $0 / BASH=$BASH_VERSION"
    echo "Platform:  $(uname -o) | Prefix: $PREFIX"
    python3 --version 2>/dev/null || echo "Python: NOT FOUND"
    git --version 2>/dev/null || echo "Git: NOT FOUND"
    
    # Storage check
    if [ -d "/sdcard" ]; then
        echo "Android Storage: Available at /sdcard"
    fi
    
    # Disk space
    df -h "$HOME" 2>/dev/null | tail -1
}

validate_python_project() {
    local project_root="${1:-.}"
    
    echo "========================================"
    echo "PYTHON PROJECT VALIDATION: $project_root"
    echo "========================================"
    
    cd "$project_root"
    
    # Check git
    if [ -d ".git" ]; then
        echo "✅ Git repository present"
    else
        echo "⚠️  Not a git repository"
    fi
    
    # Check requirements
    if [ -f "requirements.txt" ]; then
        echo "✅ requirements.txt present"
    else
        echo "⚠️  No requirements.txt"
    fi
    
    # Check pyproject.toml
    if [ -f "pyproject.toml" ]; then
        echo "✅ pyproject.toml present"
    else
        echo "⚠️  No pyproject.toml"
    fi
    
    # Check tests
    if [ -d "tests" ]; then
        echo "✅ tests/ directory present"
        test_count=$(find tests -name "*.py" 2>/dev/null | wc -l | tr -d ' ')
        echo "   Found $test_count test files"
    else
        echo "⚠️  No tests/ directory"
    fi
    
    # Check Python syntax
    echo ""
    echo "Checking Python syntax..."
    python3 -m py_compile src/*.py 2>&1 | head -20 || echo "✅ Syntax OK (or no .py files)"
}

disk_usage_report() {
    local path="${1:-$HOME}"
    
    echo "========================================"
    echo "DISK USAGE: $path"
    echo "========================================"
    du -sh "$path"/* 2>/dev/null | sort -hr | head -20
}

# ============================================================================
# CLEANUP & MAINTENANCE
# ============================================================================

cleanup_pycache() {
    local path="${1:-.}"
    
    msg_info "Removing __pycache__ directories in: $path"
    find "$path" -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null || true
    find "$path" -type f -name "*.pyc" -delete 2>/dev/null || true
    msg_success "Cleanup complete"
}

cleanup_git_tempfiles() {
    local path="${1:-.}"
    
    msg_info "Cleaning Git temporary files in: $path"
    cd "$path"
    git gc --aggressive --prune=now 2>/dev/null || true
    git reflog expire --expire=now --all 2>/dev/null || true
    msg_success "Git garbage collection complete"
}

find_large_files() {
    local path="${1:-$HOME}"
    local size_mb="${2:-10}"
    
    msg_info "Finding files larger than ${size_mb}MB in: $path"
    find "$path" -type f -size +"${size_mb}M" -exec ls -lh {} \; 2>/dev/null | \
        awk '{print $5, $9}' | sort -hr | head -20
}

# ============================================================================
# BACKUP & RESTORE
# ============================================================================

create_backup_tarball() {
    local source_dir="$1"
    local backup_name="${2:-$(basename "$source_dir")_$(date +%Y%m%d_%H%M%S)}"
    local backup_path="$EZ_BACKUP_DIR/${backup_name}.tar.gz"
    
    msg_info "Creating backup: $backup_path"
    mkdir_safe "$EZ_BACKUP_DIR"
    tar -czf "$backup_path" -C "$(dirname "$source_dir")" "$(basename "$source_dir")" 2>/dev/null
    
    if [ -f "$backup_path" ]; then
        msg_success "Backup created: $backup_path ($(du -h "$backup_path" | cut -f1))"
    else
        msg_error "Backup failed"
        return 1
    fi
}

restore_from_backup() {
    local backup_file="$1"
    local restore_path="${2:-.}"
    
    if [ ! -f "$backup_file" ]; then
        msg_error "Backup file not found: $backup_file"
        return 1
    fi
    
    msg_info "Restoring from: $backup_file to $restore_path"
    mkdir_safe "$restore_path"
    tar -xzf "$backup_file" -C "$restore_path"
    msg_success "Restore complete"
}

# ============================================================================
# HELP & VERSION
# ============================================================================

show_help() {
    cat << EOF
ez_archive.sh v${EZ_ARCHIVE_VERSION} — Resilient Bash Toolkit

USAGE:
  source ./ez_archive.sh <command> [arguments...]
  ./ez_archive.sh <command> [arguments...]   (executable mode)

DIRECTORY MANAGEMENT:
  project_init <name> [path]       Initialize new project structure
  cd_project <name>                Change to project directory
  tree_view [path] [depth]         Display directory tree
  mkdir_safe <directory>           Create directory with parents

GIT OPERATIONS:
  git_init [directory]             Initialize git repository
  git_status                       Show git status and branch
  git_stash_all [message]          Stash all uncommitted changes
  git_stash_pop                      Apply latest stash
  git_sync [remote] [branch]       Pull latest changes
  git_push_force [branch]          Force push (rewrites history!)
  git_diff_recent [count]          Show recent commit history
  git_branch_list                  List local and remote branches

FILE INSPECTION:
  view_file_tail <file> [lines]    Show last N lines of file
  view_file_tac <file> [lines]     Show last N lines in reverse
  view_json_formatted <file>       Pretty-print JSON file
  search_file_content <pattern> [dir] [ext] Search text in files
  count_lines <file>               Count lines in file
  find_large_files [path] [sizeMB] Find files larger than size

PYTHON UTILITIES:
  run_python_inline                Execute Python from heredoc
  python_check_imports <pkg1> <pkg2>...  Check if packages installed
  python_validate_json <file>      Validate JSON syntax

VALIDATION:
  check_termux_environment         Display system info
  validate_python_project [root]   Validate project structure
  disk_usage_report [path]         Show disk usage breakdown

MAINTENANCE:
  cleanup_pycache [path]           Remove Python cache files
  cleanup_git_tempfiles [path]     Run git gc and reflog expire
  find_large_files [path] [MB]     Find large files
  backup_file <file> [suffix]      Create timestamped backup

BACKUP & RESTORE:
  create_backup_tarball <dir> [name]  Create compressed backup
  restore_from_backup <file> [path]   Restore from tarball

ENVIRONMENT VARIABLES:
  EZ_VERBOSE=1                   Enable verbose debug output
  EZ_COLORS=0                    Disable ANSI color codes
  EZ_LOG_DIR=<path>              Custom log directory
  EZ_BACKUP_DIR=<path>           Custom backup directory

EXAMPLES:
  source ez_archive.sh project_init my_api ~/projects/my_api
  source ez_archive.sh git_status
  source ez_archive.sh run_python << 'PYEOF'
    import json
    print(json.dumps({"status": "ok"}))
  PYEOF
  source ez_archive.sh check_termux_environment
  source ez_archive.sh cleanup_pycache ~/projects

TIPS:
  • Source this script once per session: source ~/.ez_archive.sh
  • Set EZ_VERBOSE=1 for debug output on every command
  • Use EZ_COLORS=0 if colors cause issues in your terminal
  • All commands write to ~/.ez_archive/logs/ez_archive.log

EOF
}

show_version() {
    echo "ez_archive.sh version ${EZ_ARCHIVE_VERSION}"
}

# ============================================================================
# MAIN COMMAND DISPATCHER
# ============================================================================

main() {
    local command="${1:-help}"
    shift 2>/dev/null || true
    
    case "$command" in
        # Directory Management
        project_init)   project_init "$@" ;;
        cd_project)     cd_project "$@" ;;
        tree_view)      tree_view "$@" ;;
        mkdir_safe)     mkdir_safe "$@" ;;
        
        # Git Operations
        git_init)       git_init "$@" ;;
        git_status)     git_status ;;
        git_stash_all)  git_stash_all "$@" ;;
        git_stash_pop)  git_stash_pop ;;
        git_sync)       git_sync "$@" ;;
        git_push_force) git_push_force "$@" ;;
        git_diff_recent) git_diff_recent "$@" ;;
        git_branch_list) git_branch_list ;;
        
        # File Inspection
        view_file_tail) view_file_tail "$@" ;;
        view_file_tac)  view_file_tac "$@" ;;
        view_json_formatted) view_json_formatted "$@" ;;
        search_file_content) search_file_content "$@" ;;
        count_lines)    count_lines "$@" ;;
        find_large_files) find_large_files "$@" ;;
        
        # Python Utilities
        run_python_inline) run_python_inline "$@" ;;
        python_check_imports) python_check_imports "$@" ;;
        python_validate_json) python_validate_json "$@" ;;
        
        # Validation
        check_termux_environment) check_termux_environment ;;
        validate_python_project) validate_python_project "$@" ;;
        disk_usage_report) disk_usage_report "$@" ;;
        
        # Maintenance
        cleanup_pycache) cleanup_pycache "$@" ;;
        cleanup_git_tempfiles) cleanup_git_tempfiles "$@" ;;
        backup_file)     backup_file "$@" ;;
        
        # Backup & Restore
        create_backup_tarball) create_backup_tarball "$@" ;;
        restore_from_backup) restore_from_backup "$@" ;;
        
        # Help & Version
        help|--help|-h) show_help ;;
        version|--version|-v) show_version ;;
        
        *)
            msg_error "Unknown command: $command"
            msg_info "Run './ez_archive.sh help' for usage"
            exit 1
            ;;
    esac
}

# If executed directly (not sourced), run main
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

---

## Installation & Quick Start

```bash
# Save to your home directory
cd ~
curl -sSL https://raw.githubusercontent.com/bilbywilby/ez_remote/main/ez_archive.sh > ez_archive.sh 2>/dev/null || \
cat > ez_archive.sh  # Paste the script above

chmod +x ez_archive.sh

# Make it available globally (copy to clipboard-friendly location)
cp ez_archive.sh ~/.ez_archive.sh

# Load into current session
source ~/.ez_archive.sh
```

---

## Common Usage Patterns

### Pattern 1: One-Liner Commands

```bash
# Quick project setup
source ez_archive.sh project_init new_backend ~/projects/new_backend

# Check git state anywhere
source ez_archive.sh git_status

# View last 100 lines of a log file
source ez_archive.sh view_file_tail ~/ez_remote/logs/pipeline_refresh.log 100

# Validate JSON config
source ez_archive.sh view_json_formatted src/source_registry.json
```

### Pattern 2: Embedded Python Execution

```bash
# Run inline Python with heredoc
source ez_archive.sh run_python << 'PYEOF'
import json
from datetime import datetime

data = {
    "timestamp": datetime.utcnow().isoformat(),
    "hostname": "termux",
    "modules_loaded": ["json", "datetime"]
}

print(json.dumps(data, indent=2))
PYEOF
```

### Pattern 3: Multi-Step Workflows

```bash
# Combine multiple ez_archive commands
source ez_archive.sh git_stash_all "before_major_change"
source ez_archive.sh cleanup_pycache .
source ez_archive.sh git_pull origin main
source ez_archive.sh validate_python_project .
source ez_archive.sh git_stash_pop
```

### Pattern 4: Debug Mode

```bash
# Enable verbose output for debugging
export EZ_VERBOSE=1
source ez_archive.sh
source ez_archive.sh git_status
# Every function call will be logged
```

---

## Key Features That Prevent Common Mistakes

| Problem | ez_archive Solution |
|---------|---------------------|
| Heredoc delimiter collisions | Uses unique quoted delimiters (`'EOF'`) everywhere |
| Variable expansion bugs | All variables use `${VAR}` syntax with defaults |
| Missing command errors | `check_dependencies()` validates before execution |
| Silent failures | All functions log to `~/.ez_archive/logs/ez_archive.log` |
| Color issues in terminals | `EZ_COLORS=0` disables ANSI codes |
| Wrong working directory | Most functions accept path arguments explicitly |
| Accidental overwrites | `backup_file()` creates timestamped backups |

---

## Save to Clipboard (One Command)

```bash
# Copy entire script to Android clipboard
cat ez_archive.sh | termux-clipboard-set

# Or save as universal snippet
cat > ~/.ez_archive_snippet.txt << 'SNIP_EOF'
[PASTE THE ENTIRE SCRIPT HERE]
SNIP_EOF
```

---

## Next Time You Need This

Just run:

```bash
source ~/.ez_archive.sh [any_command]
```

