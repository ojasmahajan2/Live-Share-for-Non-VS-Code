# 🔧 Fix VS Live Share on Antigravity IDE

## Copy-Paste Prompt for AI Assistants

> **Prompt:**
>
> Fix VS Live Share (ms-vsliveshare.vsliveshare) to work on Antigravity IDE. The extension installs but commands show "not found" because of three issues:
>
> 1. **Finalized API proposal**: Remove `notebookCellExecutionState` from both the extension's `package.json` (`enabledApiProposals` array) and Antigravity's `product.json` (`extensionEnabledApiProposals` → `ms-vsliveshare.vsliveshare` entry). This proposal was finalized and no longer exists, causing extension load failures.
>
> 2. **checkHostApp.js** (obfuscated webpack chunk in the extension root): This file contains an obfuscated host validation check that only allows VS Code. Replace the entire file contents with:
>    ```js
>    'use strict';exports['id']=0x218,exports['ids']=[0x218],exports['modules']={0x8aca:(_m,_exports,_req)=>{Object.defineProperty(_exports,'__esModule',{value:true});_exports.checkHostApp=function checkHostApp(){return true;};}}; 
>    ```
>
> 3. **checkHostAppName() in extension.js**: Find the function `checkHostAppName(){return["Visual Studio Code","Visual Studio Code - Insiders","Visual Studio Code - Exploration"].includes(r.env.appName)}` and replace it with `checkHostAppName(){return true}`.
>
> 4. **GitHub Authentication Client ID**: Antigravity IDE uses its own GitHub OAuth Client ID (`Ov23liqK7zUW1cAPleVE`) but Live Share's backend expects the official VS Code Client ID. Find `C:\Users\ojasmahajan\AppData\Local\Programs\Antigravity IDE\resources\app\extensions\github-authentication\dist\extension.js`, search for `Ov23liqK7zUW1cAPleVE` and replace it with `01ab8ac9400c4e429b23`.
>
> After patching, fully restart Antigravity IDE, then sign out of GitHub in the IDE and sign back in to refresh the token.

---

## Manual Instructions (No AI Needed)

### File Locations
- **Extension dir**: `%USERPROFILE%\.antigravity-ide\extensions\ms-vsliveshare.vsliveshare-<VERSION>\`
- **Product.json**: `%LOCALAPPDATA%\Programs\Antigravity IDE\resources\app\product.json`

### Patch 1: Remove Finalized API Proposal

**In the extension's `package.json`**, find `enabledApiProposals` and remove `"notebookCellExecutionState"` from the array.

**In Antigravity's `product.json`**, find `extensionEnabledApiProposals` → `ms-vsliveshare.vsliveshare` and remove `"notebookCellExecutionState"` from its array.

### Patch 2: Replace `checkHostApp.js`

Replace the **entire contents** of `checkHostApp.js` (in the extension root) with:

```js
'use strict';exports['id']=0x218,exports['ids']=[0x218],exports['modules']={0x8aca:(_m,_exports,_req)=>{Object.defineProperty(_exports,'__esModule',{value:true});_exports.checkHostApp=function checkHostApp(){return true;};}}; 
```

### Patch 3: Patch `extension.js`

Open `extension.js` in the extension root. Search for:

```
checkHostAppName(){return["Visual Studio Code","Visual Studio Code - Insiders","Visual Studio Code - Exploration"].includes(r.env.appName)}
```

Replace with:

```
checkHostAppName(){return true}
```

### Patch 4: Fix GitHub Authentication Client ID

Open `%LOCALAPPDATA%\Programs\Antigravity IDE\resources\app\extensions\github-authentication\dist\extension.js`.

Search for `Ov23liqK7zUW1cAPleVE` and replace it with `01ab8ac9400c4e429b23`.

> [!TIP]
> You can do patches 3 and 4 quickly with PowerShell:
> ```powershell
> $extDir = "$env:USERPROFILE\.antigravity-ide\extensions"
> $lsDir = Get-ChildItem $extDir -Directory -Filter "ms-vsliveshare.vsliveshare-*" | Select-Object -First 1
> $extJs = Join-Path $lsDir.FullName "extension.js"
> $content = [System.IO.File]::ReadAllText($extJs)
> $content = $content.Replace(
>   'function checkHostAppName(){return["Visual Studio Code","Visual Studio Code - Insiders","Visual Studio Code - Exploration"].includes(r.env.appName)}',
>   'function checkHostAppName(){return true}'
> )
> [System.IO.File]::WriteAllText($extJs, $content)
> 
> $authJs = "$env:LOCALAPPDATA\Programs\Antigravity IDE\resources\app\extensions\github-authentication\dist\extension.js"
> $authContent = [System.IO.File]::ReadAllText($authJs)
> $authContent = $authContent.Replace("Ov23liqK7zUW1cAPleVE", "01ab8ac9400c4e429b23")
> [System.IO.File]::WriteAllText($authJs, $authContent)
> ```

### Restart & Re-authenticate

1. **Fully close and relaunch Antigravity IDE** (not just reload window).
2. Click the **Accounts** icon (bottom left), and select **Sign Out** from your GitHub account.
3. Sign back in using Live Share to grab a new token.

---

## Why This Is Needed

Live Share has three layers of protection preventing it from running on VS Code forks:

| Layer | What it checks | Where |
|-------|---------------|-------|
| API Proposals | Extension requests `notebookCellExecutionState` which was finalized | `package.json` + `product.json` |
| `checkHostApp()` | Obfuscated webpack chunk that validates the host app using encrypted checks against VS Code's `product.json` | `checkHostApp.js` |
| `checkHostAppName()` | Checks `vscode.env.appName` against a whitelist of VS Code variants | `extension.js` |
| GitHub OAuth Client ID | Antigravity uses a custom Client ID for GitHub OAuth, but Live Share expects the token to be issued to VS Code's Client ID | `github-authentication` extension |

> [!WARNING]
> These patches will be **overwritten** if you update or reinstall the Live Share extension. Re-apply after any update.

> [!NOTE]
> This was tested with Live Share v1.1.122 on Antigravity IDE v2.0.1. The `checkHostApp.js` webpack chunk ID (`0x218` / module `0x8aca`) may differ in other versions — but the approach remains the same: replace the obfuscated file with a simple `return true`.
