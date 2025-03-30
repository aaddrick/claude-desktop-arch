# Maintainer: Your Name <your_email@domain.com> # Please update maintainer info

# --- Package Metadata ---
_pkgname=claude-desktop # Internal variable for the base package name
pkgname=$_pkgname       # The actual package name displayed to the user
pkgver=0.9.0            # Package version - Hardcoded, update manually for new releases
pkgrel=1                # Package release number, reset to 1 when pkgver changes
pkgdesc="Claude Desktop for Linux – an AI assistant from Anthropic" # Package description
arch=('x86_64')         # Supported architecture
url="https://github.com/aaddrick/claude-desktop-arch" # Project URL (this repo)
license=('Unlicense')   # Package license
depends=('nodejs' 'npm' 'electron' 'p7zip' 'icoutils' 'imagemagick') # Runtime dependencies
makedepends=('wget' 'p7zip') # Build-time dependencies (p7zip needed in build())

# --- Source Files ---
# Use a generic name for the installer in source array for easier referencing
_installer_filename="Claude-Setup-x64.exe"
# Define source files: installer from Google Storage, LICENSE from GitHub
source=("$_installer_filename::https://storage.googleapis.com/osprey-downloads-c02f6a0d-347c-492b-a752-3e0651722e97/nest-win-x64/Claude-Setup-x64.exe"
        "LICENSE::https://raw.githubusercontent.com/aaddrick/claude-desktop-arch/main/LICENSE")
# Define SHA256 checksums for source files. 'SKIP' is used because the installer checksum changes with each version.
sha256sums=('SKIP' # Installer checksum changes with version
            'SKIP') # Unlicense checksum (can be calculated if needed)

# --- Build Process ---
# This function prepares the application files from the downloaded sources.
build() {
  # Navigate to the source directory where makepkg downloaded files
  cd "$srcdir"
  # Define the path to the downloaded installer using the generic filename
  local _installer_file="$srcdir/$_installer_filename"

  # Create a build directory to keep extracted files organized
  mkdir -p build
  cd build

  # Extract the Windows installer using 7zip
  echo "Extracting Windows installer..."
  7z x -y "$_installer_file" || { echo "Extraction failed"; exit 1; } # Exit if extraction fails

  # The installer contains a .nupkg (NuGet package) file which holds the actual application files.
  echo "Extracting nupkg for version ${pkgver}..."
  # Construct the expected path to the nupkg file using the package version
  local _nupkg_path="./AnthropicClaude-${pkgver}-full.nupkg"
  # Check if the nupkg file exists
  if [ ! -f "$_nupkg_path" ]; then
      echo "❌ Could not find nupkg file for version ${pkgver}: $_nupkg_path"
      ls -la # List files for debugging
      exit 1 # Exit if nupkg is missing
  fi
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
  npx asar extract app.asar app.asar.contents || { echo "asar extract failed"; exit 1; }

  # Modify the application's JavaScript to enable the standard window frame and menu bar
  echo "Attempting to set frame:true and remove titleBarStyle/titleBarOverlay in index.js..."
  # Use sed to find and replace the relevant BrowserWindow options in the bundled JS.
  # This removes the custom title bar and enables the native one (frame:true).
  sed -i 's/height:e\.height,titleBarStyle:"default",titleBarOverlay:[^,]\+,/height:e.height,frame:true,/g' app.asar.contents/.vite/build/index.js || echo "Warning: sed command failed to modify index.js"


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

  # Copy necessary resources from the extracted nupkg into the asar contents
  echo "Copying tray and i18n resources..."
  # Ensure the resources directory exists within the asar contents
  mkdir -p app.asar.contents/resources
  # Copy tray icon files (ignore errors if they don't exist)
  cp ../lib/net45/resources/Tray* app.asar.contents/resources/ || echo "Warning: Failed to copy Tray icons"
  # Copy internationalization (i18n) JSON files
  mkdir -p app.asar.contents/resources/i18n
  cp ../lib/net45/resources/*-*.json app.asar.contents/resources/i18n/ || echo "Warning: Failed to copy i18n JSON files"

  # Repack the modified contents back into the app.asar archive
  echo "Repacking asar..."
  npx asar pack app.asar.contents app.asar || { echo "asar pack failed"; exit 1; }


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
  cp -r electron-app/app.asar.unpacked "$pkgdir/usr/share/lib/$_pkgname/" # Corrected path

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
  # This script handles detecting Wayland and passing appropriate flags to Electron.
  cat > "$pkgdir/usr/bin/$_pkgname" << EOF
#!/bin/bash
# Launcher script for Claude Desktop

# Detect if Wayland session is likely running by checking WAYLAND_DISPLAY environment variable
IS_WAYLAND=false
if [ ! -z "\$WAYLAND_DISPLAY" ]; then
  IS_WAYLAND=true
fi

# Base arguments for Electron: path to the app.asar archive
ELECTRON_ARGS=("/usr/lib/$_pkgname/app.asar")

# Add specific flags for Wayland if detected
if [ "\$IS_WAYLAND" = true ]; then
  echo "Wayland detected, adding Wayland flags to Electron..."
  # Flags to enable Ozone platform (Wayland backend) and Wayland window decorations
  ELECTRON_ARGS+=("--enable-features=UseOzonePlatform,WaylandWindowDecorations" "--ozone-platform=wayland")
fi

# Append any arguments passed to this launcher script (e.g., URLs) to the Electron arguments
ELECTRON_ARGS+=("\$@")

# Execute the 'electron' command (provided by the electron dependency) with the constructed arguments
electron "\${ELECTRON_ARGS[@]}"
EOF
  # Make the launcher script executable
  chmod +x "$pkgdir/usr/bin/$_pkgname"

  # Workaround for a potential hardcoded path issue in the application.
  # Some Electron apps might expect resources in a specific Electron version path.
  # This copies the en-US localization file to a path Electron might look for.
  # Note: This might need adjustment depending on the Electron version dependency.
  local extracted_en_us_json="$srcdir/build/lib/net45/resources/en-US.json"
  if [ -f "$extracted_en_us_json" ]; then
    echo "Applying workaround: Copying extracted en-US.json to /usr/lib/electron/resources/" # Adjusted path
    install -d "$pkgdir/usr/lib/electron/resources" # Adjusted path
    install -Dm644 "$extracted_en_us_json" "$pkgdir/usr/lib/electron/resources/en-US.json" # Adjusted path
  else
    echo "Warning: Could not find extracted en-US.json at $extracted_en_us_json to apply workaround."
  fi

  # Install the LICENSE file into the standard location
  install -Dm644 "$srcdir/LICENSE" "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
}
