# Electron.js Dark Mode

This example demonstrates an Electron application that derives its theme colors from the `nativeTheme` API. It provides theme toggle and reset controls using IPC channels.

## Overview

### Automatically Update Native Interfaces

"Native interfaces" include the file picker, window border, dialogs, context menus, and more—anything where the UI comes from your operating system and not from your app. The default behavior is to opt into this automatic theming from the OS.

### Automatically Update Your Own Interfaces

If your app has its own dark mode, you should toggle it on and off in sync with the system's dark mode setting. You can do this by using the `prefers-color-scheme` CSS media query.

### Manually Update Your Own Interfaces

If you want to manually switch between light and dark modes, you can set the desired mode in the `themeSource` property of the `nativeTheme` module. This property's value will be propagated to your renderer process. Any CSS rules related to `prefers-color-scheme` will be updated accordingly.

## macOS Settings

### macOS 10.14 Mojave and Later

In macOS 10.14 Mojave, Apple introduced a new system-wide dark mode for all macOS computers. If your Electron app has a dark mode, you can make it follow the system-wide dark mode setting using the `nativeTheme` API.

### macOS 10.15 Catalina

In macOS 10.15 Catalina, Apple introduced a new "automatic" dark mode option. To ensure `nativeTheme.shouldUseDarkColors` and Tray APIs work correctly in this mode on Catalina, you need:

- Electron >=7.0.0, or
- Set `NSRequiresAquaSystemAppearance` to `false` in your `Info.plist` file (for older versions)

Both Electron Packager and Electron Forge have a `darwinDarkModeSupport` option to automate the `Info.plist` changes during app build time.

### Opting Out (Electron > 8.0.0)

If you wish to opt-out while using Electron > 8.0.0, set the `NSRequiresAquaSystemAppearance` key in the `Info.plist` file to `true`. Note that Electron 8.0.0 and above will not let you opt-out of this theming, due to the use of the macOS 10.14 SDK.

## Project Structure

- `main.js` - Main process
- `preload.js` - Preload script
- `index.html` - HTML template
- `renderer.js` - Renderer process
- `style.css` - Stylesheet

## Implementation

### main.js

```javascript
const { app, BrowserWindow, ipcMain, nativeTheme } = require('electron/main')
const path = require('node:path')

function createWindow () {
  const win = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js')
    }
  })

  win.loadFile('index.html')
}

ipcMain.handle('dark-mode:toggle', () => {
  if (nativeTheme.shouldUseDarkColors) {
    nativeTheme.themeSource = 'light'
  } else {
    nativeTheme.themeSource = 'dark'
  }
  return nativeTheme.shouldUseDarkColors
})

ipcMain.handle('dark-mode:system', () => {
  nativeTheme.themeSource = 'system'
})

app.whenReady().then(() => {
  createWindow()

  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) {
      createWindow()
    }
  })
})

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit()
  }
})
```

### index.html

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Hello World!</title>
    <meta http-equiv="Content-Security-Policy" content="script-src 'self' 'unsafe-inline';" />
    <link rel="stylesheet" type="text/css" href="./styles.css">
</head>
<body>
    <h1>Hello World!</h1>
    <p>Current theme source: <strong id="theme-source">System</strong></p>

    <button id="toggle-dark-mode">Toggle Dark Mode</button>
    <button id="reset-to-system">Reset to System Theme</button>

    <script src="renderer.js"></script>
</body>
</html>
```

### style.css

```css
@media (prefers-color-scheme: dark) {
  body { background: #333; color: white; }
}

@media (prefers-color-scheme: light) {
  body { background: #ddd; color: black; }
}
```

The example renders an HTML page with a few key elements:
- The `<strong id="theme-source">` element displays which theme is currently selected
- The two `<button>` elements provide theme controls
- The CSS file uses the `prefers-color-scheme` media query to set the `<body>` element's background and text colors

### preload.js

```javascript
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('darkMode', {
  toggle: () => ipcRenderer.invoke('dark-mode:toggle'),
  system: () => ipcRenderer.invoke('dark-mode:system')
})
```

The `preload.js` script adds a new API to the `window` object called `darkMode`. This API exposes two IPC channels to the renderer process:
- `dark-mode:toggle`
- `dark-mode:system`

It also assigns two methods (`toggle` and `system`) which pass messages from the renderer to the main process. This allows the renderer process to communicate with the main process securely and perform the necessary mutations to the `nativeTheme` object.

### renderer.js

```javascript
document.getElementById('toggle-dark-mode').addEventListener('click', async () => {
  const isDarkMode = await window.darkMode.toggle()
  document.getElementById('theme-source').innerHTML = isDarkMode ? 'Dark' : 'Light'
})

document.getElementById('reset-to-system').addEventListener('click', async () => {
  await window.darkMode.system()
  document.getElementById('theme-source').innerHTML = 'System'
})
```

The `renderer.js` file is responsible for controlling the button functionality. Using `addEventListener`, it adds 'click' event listeners to each button element. Each event listener handler makes calls to the respective `window.darkMode` API methods.

## How It Works

1. **IPC Communication**: The `ipcMain.handle` methods are how the main process responds to click events from the buttons on the HTML page.

2. **Toggle Handler**: The `dark-mode:toggle` IPC channel handler checks the `shouldUseDarkColors` boolean property, sets the corresponding `themeSource`, and returns the current `shouldUseDarkColors` value. This return value is used to update the theme source text in the UI.

3. **System Handler**: The `dark-mode:system` IPC channel handler assigns the string `'system'` to `themeSource`, allowing the app to follow the OS theme settings.

## Running the Example

Use Electron Fiddle to run the example, or execute it with:

```bash
npm start
```

Click the "Toggle Dark Mode" button to switch between light and dark themes. The app will alternate the background color accordingly based on the current theme selection.
