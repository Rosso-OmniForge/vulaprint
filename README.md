# Vula! Print Label Printer

## Desktop Printing Application

A modern, branded desktop application for printing product labels directly from the Vula! Print inventory system.

### Features

- 🖨️ **Auto-Discovery**: Automatically scans for USB label printers
- 🎯 **Calibration**: Built-in printer calibration with test labels
- 🔄 **Auto-Sync**: Automatically fetches print requests from the server every 30 seconds
- 🚀 **Auto-Start + Auto-Connect**: Starts at login and reconnects to saved API URL
- ✅ **Modern UI**: Clean, branded PyQt6 interface
- 🔒 **Secure**: API key authentication
- 📊 **Real-time Progress**: Live printing progress with status updates

### Installation

#### System Requirements

- **OS**: Debian 13 Trixie (or compatible Linux distro)
- **Printer**: USB-connected TSC/TSPL label printer (40mm x 30mm labels)
- **Python**: 3.10 or higher
- **Network**: Connection to Vula! Print backend server

#### Quick Install

```bash
cd /path/to/printer
chmod +x install_printer_app.sh
./install_printer_app.sh
```

This will:
1. Install system dependencies (Python, PyQt6, USB libraries)
2. Create a Python virtual environment
3. Install all required Python packages
4. Make the application executable
5. Create desktop autostart entry (`~/.config/autostart/vula-print.desktop`)

### Running the Application

#### Option 1: Using the Launcher (Recommended)

```bash
./launch_printer.sh
```

#### Option 2: Manual Launch

```bash
source venv/bin/activate
python3 vula_print_app.py
```

### Startup on Boot (Desktop Login)

The installer creates this autostart file:

```bash
~/.config/autostart/vula-print.desktop
```

To disable autostart:

```bash
rm ~/.config/autostart/vula-print.desktop
```

### Configuration

1. **API Server URL**: Enter the backend server URL (default: `http://localhost:8000`)
   - URL is saved automatically and restored on next launch
   - App attempts automatic API connection on startup
2. **API Key**: Configured in the application (default: `VULA-PRINTER-2026-SECURE-KEY`)
   - For production, update the API key in both:
     - `vula_print_app.py` (line ~21)
     - Backend: `backend/app/routes/label_printing.py` (line ~307)

### Usage Guide

#### First-Time Setup

1. **Launch the Application**
   ```bash
   ./launch_printer.sh
   ```

2. **Connect Printer**
   - Connect your USB label printer
   - Click "Scan for Printers"
   - Select your printer from the dropdown

3. **Configure API**
   - Enter your backend server URL
   - Click "Test API Connection"
   - Verify connection is successful (green indicator)

4. **Calibrate Printer**
   - Click "Calibrate & Test Print"
   - Verify the test label prints correctly
   - Confirm calibration when prompted

#### Printing Labels

1. **Refresh Queue**
   - Click "🔄 Refresh Queue" or wait for auto-refresh (30s)
   - Pending print requests will appear in the table

2. **Review Request**
   - Click on a request to see details
   - Review items, quantities, and source

3. **Print**
   - Click the "🖨️ Print" button for the desired request
   - Monitor progress in the progress bar
   - Request will be marked as completed automatically

### UI Design

The application features a modern dark theme with Vula! Print branding:

- **Color Scheme**: Dark background (#1a1a1a) with orange accents (#ff6b35)
- **Header**: Displays the Primary Text Logo scaled to 60px height with real-time status indicators
  - Connection Status: Green (●) when connected, Red (●) when disconnected
  - Printer Status: Shows current printer state
- **Styled Components**:
  - Orange gradient buttons with hover effects
  - Dark-themed tables with orange selection highlights
  - Bordered panels with orange accents
  - Monospace details display with orange text
- **Professional Appearance**: Consistent branding throughout the interface matching Vula! Print's visual identity

### Auto-Generated Print Requests

Print requests are automatically created when:

#### From Procurement

- **Price Unchanged**: When stock is added, prints labels for only the new items
- **Price Changed**: When stock is added AND price changes, prints labels for ALL items (existing + new)

#### Manual Requests

- Staff can manually create print requests via the admin panel
- Useful for replacing damaged labels

### Troublesoting

#### Printer Not Found

```bash
# Check USB connection
ls -la /dev/usb/lp*

# Check USB devices
lsusb

# Ensure permissions
sudo usermod -a -G lp $USER
```

#### Connection Failed

- Verify backend server is running
- Check firewall settings
- Confirm API key matches backend configuration
- Test with: `curl -H "X-API-Key: YOUR_KEY" http://your-server:8000/admin/api/label-printing/pending`

#### Label Alignment Issues

1. Verify label size is exactly 40mm x 30mm
2. Adjust gap setting in printer (should be 2mm)
3. Run calibration again
4. Check `horizontal_shift_dots` in code if needed (line ~273 in `vula_print_app.py`)

#### PyQt6 Installation Issues

```bash
# On Debian 13, you may need:
sudo apt-get install python3-pyqt6 python3-pyqt6.qtcore python3-pyqt6.qtwidgets

# If pip install fails, use system packages:
pip install --no-deps PyQt6
```

### API Endpoints Used

- `GET /admin/api/label-printing/pending` - Fetch pending requests
- `GET /admin/api/label-printing/request/{id}` - Get request details
- `POST /admin/api/label-printing/complete` - Mark request as completed

### Security Notes

⚠️ **Important**: Change the default API key before deploying to production!

1. Generate a secure random key:
   ```bash
   openssl rand -base64 32
   ```

2. Update in both:
   - Desktop app: `vula_print_app.py`
   - Backend: `backend/app/routes/label_printing.py`

3. Consider using environment variables for the API key

### Development

#### Running in Development Mode

```bash
# Activate virtual environment
source venv/bin/activate

# Run with debug output
python3 vula_print_app.py

# Or with verbose logging
python3 -v vula_print_app.py
```

#### Customizing Labels

Edit the `_generate_label_tspl()` method in `vula_print_app.py` to customize:
- Font sizes and positions
- Barcode settings (type, size, position)
- Label layout and content

### Support

For issues or questions, contact the OmniForge.

### License

Proprietary - OmniForge © 2026
# vulaprint
