ℹ️ This requires version 2.2.0 or higher

Commands like `usbipd list` and `usbipd wsl list` output their information in a human-readable table format where some of the strings may be truncated if they go beyond the column width. Furthermore, they only display a subset of the state.

To facilitate scripted automation, two features were added in version 2.2.0.

# JSON

A new command was added which outputs *all* state information in JSON format:
```pwsh
usbipd state
```

# PowerShell

A PowerShell module was added that further processes the JSON output to strongly-typed objects that also adds some metadata that can be inferred from the information. The PowerShell module is compatible with Powershell 5.1 (the default that comes with Windows 10) as well a PowerShell 7 (from the Windows Store). The PowerShell module is tied to the version of `usbipd-win`, so it will not be released to PowerShell Gallery separately. Instead, you will have to import the module manually as follows:
```pwsh
Import-Module $env:ProgramFiles'\usbipd-win\PowerShell\Usbipd.Powershell.dll'
```

The PowerShell module exposes a single command:
```pwsh
Get-UsbipdDevice
```

## Example 1

To display a table of BusId and Description of all devices currently attached to WSL:
```pwsh
Get-UsbipdDevice | Where-Object {$_.IsWslAttached} | Format-Table BusId,Description
```

## Example 2
To display all USB devices connected your computer that are currently *not* shared:
```pwsh
Get-UsbipdDevice | Where-Object {$_.IsConnected -and -not $_.IsBound}
```
