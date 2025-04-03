# Maintainer: aaddrick <aaddrick@gmail.com> 

# --- Package Metadata ---
_pkgname=claude-desktop # Internal variable for the base package name
pkgname=$_pkgname       # The actual package name displayed to the user
# pkgver is now determined dynamically by the pkgver() function
pkgrel=1                # Package release number, reset to 1 when pkgver changes
pkgdesc="Claude Desktop for Linux – an AI assistant from Anthropic" # Package description
arch=('x86_64' 'aarch64') # Supported architectures
url="https://github.com/aaddrick/claude-desktop-arch" # Project URL (this repo)
license=('Unlicense')   # Package license
# electron is removed as it's now packaged locally
depends=('nodejs' 'npm' 'p7zip' 'icoutils' 'imagemagick') # Runtime dependencies
# npm is added for local electron/asar install
makedepends=('wget' 'p7zip' 'npm') # Build-time dependencies
install=$_pkgname.install # Script for post-install actions (sandbox permissions)
# --- Source Files & Arch Detection ---
# Define architecture-specific installer details
if [[ "$CARCH" == "x86_64" ]]; then
  _installer_filename="Claude-Setup-x64.exe"
  _installer_url="https://storage.googleapis.com/osprey-downloads-c02f6a0d-347c-492b-a752-3e0651722e97/nest-win-x64/Claude-Setup-x64.exe"
  _nupkg_suffix="-full" # Suffix for the nupkg file on x86_64
elif [[ "$CARCH" == "aarch64" ]]; then
  _installer_filename="Claude-Setup-arm64.exe"
  _installer_url="https://storage.googleapis.com/osprey-downloads-c02f6a0d-347c-492b-a752-3e0651722e97/nest-win-arm64/Claude-Setup-arm64.exe"
  _nupkg_suffix="-arm64-full" # Suffix for the nupkg file on aarch64
else
  error "Unsupported architecture: $CARCH"
  exit 1
fi

# Define source files using the architecture-specific variables
source=("$_installer_filename::$_installer_url"
        "LICENSE::https://raw.githubusercontent.com/aaddrick/claude-desktop-arch/main/LICENSE"
        "$_pkgname.install")
# --- Version Detection ---
# This function automatically determines the latest version from the downloaded files.
pkgver() {
  # Use the architecture-specific URL and filename defined earlier
  local _download_url="$_installer_url"
  local _temp_dir=$(mktemp -d)
  local _installer_file="$_temp_dir/$_installer_filename" # Use arch-specific filename

  # Download the installer to a temporary location
  echo "Checking latest version by downloading installer..." >&2 # Output to stderr
  if ! wget -q -O "$_installer_file" "$_download_url"; then
    echo "ERROR: Failed to download installer for version check." >&2
    rm -rf "$_temp_dir"
    return 1
  fi

  # Extract the installer in the temporary directory
  if ! 7z x -y -o"$_temp_dir" "$_installer_file" &> /dev/null; then
    echo "ERROR: Failed to extract installer for version check." >&2
    rm -rf "$_temp_dir"
    return 1
  fi

  # Find the nupkg file
  local _nupkg_path_relative=$(find "$_temp_dir" -maxdepth 1 -name "AnthropicClaude-*.nupkg" | head -1)
  if [ -z "$_nupkg_path_relative" ]; then
    echo "ERROR: Could not find nupkg file for version check in $_temp_dir" >&2
    ls -la "$_temp_dir" >&2 # List files for debugging
    rm -rf "$_temp_dir"
    return 1
  fi

  # Extract version from the nupkg filename (using LC_ALL=C for locale compatibility)
  # Use the architecture-specific nupkg suffix defined earlier in the regex
  local _version=$(basename "$_nupkg_path_relative" | LC_ALL=C grep -oP "AnthropicClaude-\\K[0-9]+\\.[0-9]+\\.[0-9]+(?=${_nupkg_suffix}\\.nupkg)")

  # Clean up temporary directory
  rm -rf "$_temp_dir"

  if [ -z "$_version" ]; then
    echo "ERROR: Could not extract version from nupkg filename: $(basename "$_nupkg_path_relative")" >&2
    return 1
  fi

  # Output the detected version
  echo "$_version"
}

# Define SHA256 checksums. Both installers are 'SKIP' as they change with version.
sha256sums=('SKIP' # Installer checksum changes with version
            'SKIP' # Unlicense checksum
            'SKIP') # .install script checksum (run updpkgsums)

# --- Build Process ---
# This function prepares the application files from the downloaded sources.
build() {
  # Navigate to the source directory where makepkg downloaded files
  cd "$srcdir"
  # Define the path to the downloaded installer using the generic filename
  # Use the architecture-specific installer filename defined earlier
  local _installer_file="$srcdir/$_installer_filename" # Use arch-specific filename

  # Create a build directory to keep extracted files organized
  mkdir -p build
  cd build

  # Install electron and asar locally
  echo "Installing local electron and asar..."
  # Create dummy package.json if none exists
  if [ ! -f "package.json" ]; then
      echo '{"name":"claude-desktop-build","version":"0.0.1","private":true}' > package.json
  fi
  npm install --no-save electron asar || { echo "npm install failed"; exit 1; }
  local _asar_exec="$(realpath ./node_modules/.bin/asar)"
  echo "✓ Using local asar: $_asar_exec"

  # Extract the Windows installer using 7zip
  echo "Extracting Windows installer..."
  7z x -y "$_installer_file" || { echo "Extraction failed"; exit 1; } # Exit if extraction fails

  # The installer contains a .nupkg (NuGet package) file which holds the actual application files.
  echo "Finding and extracting nupkg..."
  # Find the nupkg file dynamically (makepkg ensures sources are downloaded)
  # Use the architecture-specific nupkg suffix defined earlier
  local _nupkg_path_relative=$(find . -maxdepth 1 -name "AnthropicClaude-${pkgver}${_nupkg_suffix}.nupkg" | head -1) # Use arch-specific suffix
  if [ -z "$_nupkg_path_relative" ]; then
      echo "❌ Could not find AnthropicClaude nupkg file in $(pwd)"
      ls -la # List files for debugging
      exit 1 # Exit if nupkg is missing
  fi
  local _nupkg_path="$(realpath "$_nupkg_path_relative")" # Store full path
  echo "✓ Using nupkg: $_nupkg_path"
  # Extract the nupkg file
  7z x -y "$_nupkg_path" || { echo "nupkg extraction failed"; exit 1; } # Exit if extraction fails


  # Process application icons
  echo "Processing icons..."
  # Extract the icon group (type 14) from the main executable using wrestool (from icoutils)
  wrestool -x -t 14 "lib/net45/claude.exe" -o claude.ico || { echo "wrestool failed"; exit 1; }
  # Extract individual PNG images from the .ico file using icotool (from icoutils)
  icotool -x claude.ico || { echo "icotool failed"; exit 1; }

  # Prepare the Electron application files
  echo "Preparing Electron app..."
  # Create a directory for the Electron app components
  mkdir -p electron-app
  # Copy the main application archive (app.asar)
  cp "lib/net45/resources/app.asar" electron-app/
  # Copy any unpacked resources needed by the asar archive
  cp -r "lib/net45/resources/app.asar.unpacked" electron-app/

  # Navigate into the electron-app directory
  cd electron-app

  # Extract the asar archive to modify its contents
  echo "Extracting asar package..."
  "$_asar_exec" extract app.asar app.asar.contents || { echo "asar extract failed"; exit 1; }


  # Create a stub for the native Node.js module used by the Windows version.
  # This module handles Windows-specific features (like window effects, notifications)
  # that are not needed or don't work directly on Linux.
  echo "Creating stub native module..."
  mkdir -p app.asar.contents/node_modules/claude-native # Create the module directory
  # Create the index.js file for the stub module with no-op functions
  cat > app.asar.contents/node_modules/claude-native/index.js << 'EOF'
  // Stub implementation of claude-native for Linux compatibility
  // Provides dummy functions for Windows-specific APIs.
  // KeyboardKey enum values are kept for potential compatibility if the app uses them directly.
  const KeyboardKey = {
    Backspace: 43,
    Tab: 280,
    Enter: 261,
    Shift: 272,
    Control: 61,
    Alt: 40,
    CapsLock: 56,
    Escape: 85,
    Space: 276,
    PageUp: 251,
    PageDown: 250,
    End: 83,
    Home: 154,
    LeftArrow: 175,
    UpArrow: 282,
    RightArrow: 262,
    DownArrow: 81,
    Delete: 79,
    Meta: 187
  };
  Object.freeze(KeyboardKey); // Make the enum immutable
  // Export dummy functions matching the expected native module API
  module.exports = {
    getWindowsVersion: () => "10.0.0", // Return a plausible Windows version
    setWindowEffect: () => {},        // No-op for setting window effects
    removeWindowEffect: () => {},     // No-op for removing window effects
    getIsMaximized: () => false,      // Assume window is not maximized
    flashFrame: () => {},             // No-op for flashing window frame
    clearFlashFrame: () => {},        // No-op for clearing frame flash
    showNotification: () => {},       // No-op for showing notifications (Linux uses different mechanisms)
    setProgressBar: () => {},         // No-op for setting taskbar progress
    clearProgressBar: () => {},       // No-op for clearing taskbar progress
    setOverlayIcon: () => {},         // No-op for setting taskbar overlay icon
    clearOverlayIcon: () => {},       // No-op for clearing taskbar overlay icon
    KeyboardKey                       // Export the KeyboardKey enum
  };
EOF

  # Also create the stub in the unpacked directory
  echo "Creating stub native module in unpacked directory..."
  mkdir -p app.asar.unpacked/node_modules/claude-native
  cp app.asar.contents/node_modules/claude-native/index.js app.asar.unpacked/node_modules/claude-native/index.js

  # Copy necessary resources from the extracted nupkg into the asar contents
  echo "Copying tray and i18n resources..."
  # Ensure the resources directory exists within the asar contents
  mkdir -p app.asar.contents/resources
  # Copy tray icon files (ignore errors if they don't exist)
  cp ../lib/net45/resources/Tray* app.asar.contents/resources/ || echo "Warning: Failed to copy Tray icons"
  # Copy internationalization (i18n) JSON files
  mkdir -p app.asar.contents/resources/i18n
  cp ../lib/net45/resources/*-*.json app.asar.contents/resources/i18n/ || echo "Warning: Failed to copy i18n JSON files"

  echo "Downloading Main Window Fix Assets"
  cd app.asar.contents
  wget -O- https://github.com/emsi/claude-desktop/raw/refs/heads/main/assets/main_window.tgz | tar -zxvf -
  cd ..

  # Repack the modified contents back into the app.asar archive
  echo "Repacking asar..."
  "$_asar_exec" pack app.asar.contents app.asar || { echo "asar pack failed"; exit 1; }


  # Return to the main build directory ($srcdir/build) before finishing the build function
  cd "$srcdir/build"
}

# --- Packaging Process ---
# This function installs the built files into the final package structure ($pkgdir).
package() {
  # Navigate to the build directory where processed files are located
  cd "$srcdir/build"

  echo "Installing application files..."
  # Create necessary directories within the package staging directory ($pkgdir)
  install -d "$pkgdir/usr/lib/$_pkgname"           # Main application library directory
  install -d "$pkgdir/usr/share/applications"     # Directory for .desktop files
  install -d "$pkgdir/usr/share/icons/hicolor"    # Base directory for themed icons
  install -d "$pkgdir/usr/bin"                    # Directory for executable scripts

  # Install the main Electron application archive and its unpacked resources
  cp electron-app/app.asar "$pkgdir/usr/lib/$_pkgname/"
  cp -a electron-app/app.asar.unpacked "$pkgdir/usr/lib/$_pkgname/" # Corrected path, use -a

  # Install the locally built electron
  echo "Installing local electron..."
  cp -a "$srcdir/build/node_modules" "$pkgdir/usr/lib/$_pkgname/"

  # Install application icons into the standard hicolor theme directories
  echo "Installing icons..."
  # Loop through desired icon sizes
  for size in 16 24 32 48 64 256; do
    local icon_file="" # Variable to hold the specific icon filename
    # Map size to the corresponding filename extracted by icotool
    case "$size" in
      16) icon_file="claude_13_16x16x32.png" ;;
      24) icon_file="claude_11_24x24x32.png" ;;
      32) icon_file="claude_10_32x32x32.png" ;;
      48) icon_file="claude_8_48x48x32.png" ;;
      64) icon_file="claude_7_64x64x32.png" ;;
      256) icon_file="claude_6_256x256x32.png" ;;
    esac
    # Check if the icon file exists in the build directory
    if [ -f "$srcdir/build/$icon_file" ]; then
      # Install the icon file into the appropriate size directory within the hicolor theme
      install -Dm644 "$srcdir/build/$icon_file" "$pkgdir/usr/share/icons/hicolor/${size}x${size}/apps/${_pkgname}.png"
    else
      # Warn if a specific icon size is missing
      echo "Warning: Missing ${size}x${size} icon ($icon_file)"
    fi
  done

  # Create the .desktop file for application launchers (e.g., in GNOME, KDE)
  echo "Creating desktop entry..."
  cat > "$pkgdir/usr/share/applications/${_pkgname}.desktop" << EOF
[Desktop Entry]
Name=Claude                     # Application name displayed in menus
Exec=$_pkgname %u               # Command to execute (the launcher script), %u handles URL arguments
Icon=$_pkgname                  # Icon name (refers to the installed icons)
Type=Application                # Entry type
Terminal=false                  # Does not require a terminal window
Categories=Office;Utility;      # Application categories
MimeType=x-scheme-handler/claude; # Associates the app with claude:// URLs
StartupWMClass=Claude           # Helps window managers associate windows with this entry
EOF

  # Create a launcher script in /usr/bin
  echo "Creating launcher script..."
  # This script handles detecting Wayland, using the packaged Electron, and passing appropriate flags.
  cat > "$pkgdir/usr/bin/$_pkgname" << EOF
#!/bin/bash
# Launcher script for Claude Desktop

APP_DIR="/usr/lib/$_pkgname"
APP_ASAR="\$APP_DIR/app.asar"
ELECTRON_EXEC="\$APP_DIR/node_modules/.bin/electron" # Use packaged electron

# Check if packaged electron exists
if [ ! -f "\$ELECTRON_EXEC" ]; then
    echo "Error: Packaged Electron executable not found at \$ELECTRON_EXEC"
    # Optionally notify the user graphically
    if command -v zenity &> /dev/null; then
        zenity --error --text="Claude Desktop cannot start because the packaged Electron framework is missing. Please reinstall the package."
    elif command -v kdialog &> /dev/null; then
        kdialog --error "Claude Desktop cannot start because the packaged Electron framework is missing. Please reinstall the package."
    fi
    exit 1
fi

# Detect if Wayland session is likely running
IS_WAYLAND=false
if [ ! -z "\$WAYLAND_DISPLAY" ]; then
  IS_WAYLAND=true
fi

# Base arguments for Electron: path to the app.asar archive
ELECTRON_ARGS=("\$APP_ASAR")

# Add specific flags for Wayland if detected
if [ "\$IS_WAYLAND" = true ]; then
  echo "Wayland detected, adding Wayland flags to Electron..."
  ELECTRON_ARGS+=("--enable-features=UseOzonePlatform,WaylandWindowDecorations" "--ozone-platform=wayland")
fi

# Append any arguments passed to this launcher script (e.g., URLs)
ELECTRON_ARGS+=("\$@")

# Change to the application directory before execution
cd "\$APP_DIR" || { echo "Error: Failed to change directory to \$APP_DIR"; exit 1; }

# Execute the packaged electron command with the constructed arguments
"\$ELECTRON_EXEC" "\${ELECTRON_ARGS[@]}"
EOF
  # Make the launcher script executable
  chmod +x "$pkgdir/usr/bin/$_pkgname"

  # Install the LICENSE file into the standard location
  install -Dm644 "$srcdir/LICENSE" "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
}
