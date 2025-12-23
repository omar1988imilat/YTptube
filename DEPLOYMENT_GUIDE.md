# Deployment Guide: Running YTPTube on Another Machine

This guide explains how to copy and set up YTPTube on a different machine.

## Files to Copy

### Essential Files & Folders
You need to copy the following from your current installation:

```
ytdlp/
├── app/                    # Python application code (REQUIRED)
├── ui/                     # Frontend code (REQUIRED, but exclude node_modules)
├── pyproject.toml          # Python dependencies (REQUIRED)
├── uv.lock                 # Python lock file (REQUIRED)
├── start.sh                # Start script (REQUIRED)
├── entrypoint.sh           # Entry point script (OPTIONAL)
├── README.md               # Documentation
├── API.md                  # API documentation
├── FAQ.md                  # FAQ documentation
└── var/                    # Data directory (OPTIONAL - see below)
```

### Files & Folders to EXCLUDE (Do NOT copy)
```
❌ venv/                    # Virtual environment (will be recreated)
❌ ui/node_modules/         # Node modules (will be reinstalled)
❌ ui/.nuxt/                # Nuxt build cache (will be regenerated)
❌ ui/dist/                 # Built frontend (will be regenerated)
❌ .git/                    # Git history (optional, can skip to save space)
❌ __pycache__/             # Python cache (will be regenerated)
❌ .pytest_cache/           # Test cache
❌ build/                   # Build artifacts
❌ dist/                    # Distribution files
```

### Optional Data Files
The `var/` directory contains your application data:
- `var/config/` - Configuration files
- `var/downloads/` - Downloaded videos
- `var/tmp/` - Temporary files

**Decision**: 
- Copy `var/` if you want to migrate your settings and downloads
- Skip `var/` for a fresh installation on the new machine

---

## Quick Copy Commands

### Option 1: Using rsync (Recommended)
```bash
# From your current machine, copy to remote machine
rsync -av --progress \
  --exclude='venv/' \
  --exclude='ui/node_modules/' \
  --exclude='ui/.nuxt/' \
  --exclude='ui/dist/' \
  --exclude='.git/' \
  --exclude='__pycache__/' \
  --exclude='.pytest_cache/' \
  --exclude='build/' \
  --exclude='dist/' \
  /home/oga/ytdlp/ username@remote-machine:/path/to/destination/
```

### Option 2: Create a tar archive
```bash
# Create archive (excluding unnecessary files)
cd /home/oga
tar -czf ytdlp-deployment.tar.gz \
  --exclude='ytdlp/venv' \
  --exclude='ytdlp/ui/node_modules' \
  --exclude='ytdlp/ui/.nuxt' \
  --exclude='ytdlp/ui/dist' \
  --exclude='ytdlp/.git' \
  --exclude='ytdlp/__pycache__' \
  --exclude='ytdlp/.pytest_cache' \
  --exclude='ytdlp/build' \
  --exclude='ytdlp/dist' \
  ytdlp/

# This creates: ytdlp-deployment.tar.gz
# Transfer this file to the new machine and extract it:
# tar -xzf ytdlp-deployment.tar.gz
```

### Option 3: Using zip
```bash
cd /home/oga
zip -r ytdlp-deployment.zip ytdlp/ \
  -x "ytdlp/venv/*" \
  -x "ytdlp/ui/node_modules/*" \
  -x "ytdlp/ui/.nuxt/*" \
  -x "ytdlp/ui/dist/*" \
  -x "ytdlp/.git/*" \
  -x "ytdlp/__pycache__/*" \
  -x "ytdlp/.pytest_cache/*" \
  -x "ytdlp/build/*" \
  -x "ytdlp/dist/*"

# Transfer ytdlp-deployment.zip to the new machine and unzip it:
# unzip ytdlp-deployment.zip
```

---

## Setup on New Machine

### Prerequisites
The new machine needs the following installed:

1. **Python 3.13 or higher**
   ```bash
   python3 --version  # Check version
   ```

2. **Node.js and npm** (for the UI)
   ```bash
   node --version     # Should be v18 or higher
   npm --version
   ```

3. **UV Package Manager** (Python package manager)
   ```bash
   # Install UV if not present
   curl -LsSf https://astral.sh/uv/install.sh | sh
   # OR
   pip install uv
   ```

4. **ffmpeg** (for video processing)
   ```bash
   # Ubuntu/Debian
   sudo apt install ffmpeg

   # Fedora
   sudo dnf install ffmpeg

   # macOS
   brew install ffmpeg
   ```

### Installation Steps

1. **Extract the files** (if you created an archive)
   ```bash
   cd /path/to/destination
   tar -xzf ytdlp-deployment.tar.gz
   # OR
   unzip ytdlp-deployment.zip
   ```

2. **Navigate to the directory**
   ```bash
   cd ytdlp
   ```

3. **Create Python virtual environment and install dependencies**
   ```bash
   # Create virtual environment
   python3 -m venv venv
   
   # Activate virtual environment
   source venv/bin/activate
   
   # Install Python dependencies using UV
   uv pip install -e .
   ```

4. **Install UI dependencies**
   ```bash
   cd ui
   npm install
   
   # Build the production UI
   npm run build
   
   cd ..
   ```

5. **Create necessary directories** (if not copying var/)
   ```bash
   mkdir -p var/config var/downloads var/tmp
   ```

6. **Make start script executable**
   ```bash
   chmod +x start.sh
   ```

7. **Run the application**
   ```bash
   ./start.sh
   ```

The application should now be running! Access it at `http://localhost:8081` (or whatever port is configured).

---

## Troubleshooting

### Issue: Python version mismatch
- The app requires Python 3.13+
- Install the correct version or modify `pyproject.toml` if needed

### Issue: Missing ffmpeg
```bash
# Verify ffmpeg is installed
ffmpeg -version
```

### Issue: Permission errors
```bash
# Make sure the start script is executable
chmod +x start.sh entrypoint.sh
```

### Issue: Port already in use
- Check if port 8081 is already in use
- Modify the port in your configuration files if needed

### Issue: UI not building
```bash
cd ui
rm -rf node_modules .nuxt
npm install
npm run build
```

---

## Environment Variables & Configuration

If you have custom environment variables or configuration:

1. Check for `.env` files in the root directory
2. Copy them to the new machine
3. Update any machine-specific paths or settings

---

## Production Deployment Options

### Using Docker (Alternative)
If the new machine has Docker:
```bash
# The project includes a Dockerfile
docker build -t ytptube .
docker run -p 8081:8081 -v $(pwd)/var:/app/var ytptube
```

### Using systemd (Linux service)
Create a systemd service file to run YTPTube as a background service:
```bash
sudo nano /etc/systemd/system/ytptube.service
```

Add:
```ini
[Unit]
Description=YTPTube Web Service
After=network.target

[Service]
Type=simple
User=your-username
WorkingDirectory=/path/to/ytdlp
ExecStart=/path/to/ytdlp/start.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

Then:
```bash
sudo systemctl daemon-reload
sudo systemctl enable ytptube
sudo systemctl start ytptube
```

---

## Notes

- **Data persistence**: The `var/` directory contains all your data. Back it up regularly.
- **Updates**: Keep the `pyproject.toml` and `ui/package.json` files to track dependencies.
- **Security**: Do not expose this application directly to the internet without proper security measures (firewall, reverse proxy, authentication).
