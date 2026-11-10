# Horizon 26.3 in Horizon Launcher: Windows troubleshooting and a successful case

> **Status:** successfully tested on one Windows PC.  
> **Scenario:** Horizon Nightly `nightly-2026-11-08-fdc87ae`, Minecraft Java 26.3, locally imported installation, offline launch.  
> **Launcher:** updated to 0.2.58 during troubleshooting.  
> **This is not an official Horizon patch and is not a universal guarantee.** Treat it as a detailed report of one successful case.

## TL;DR

In this case Horizon could not finish preparing Minecraft 26.3. Logs showed Java `AccessDeniedException`, `javac.exe` export failures, and later a Minecraft registry-loading failure. During diagnosis, Java could not call `Path.toRealPath()` for paths under the Windows user profile, even though reading the JAR with `jar tf` worked.

After preparation was explicitly run for the **exact Nightly build** that was going to be launched, it completed successfully:

```text
Prepared Horizon native-client resources: ...\nightly-2026-11-08-fdc87ae\.local\horizon-client\26.3
Done in 32.8 s
nightly-2026-11-08-fdc87ae is prepared.
```

The game then launched successfully.

**Important causality caveat:** unusual explicit `Deny` ACL entries were found on the Windows profile and `AppData\Roaming`, but the proposed ACL-edit script stopped at its safety check (`Unexpected ACL`) and did not apply the change. Therefore, we cannot claim that removing those entries fixed the issue. What is confirmed: Java's `toRealPath()` initially failed on profile paths; later all tested paths returned `OK`; and preparing the exact Nightly build then succeeded.

## Symptoms and errors observed

Several errors appeared during the same investigation. They do not necessarily share one root cause.

### 1. Prepared files are missing

```text
Horizon's prepared game files are missing:
no horizon-resources.json in ...\.local\horizon-client\26.3
```

This means the prepared resource file is absent. It does not, by itself, explain why preparation failed.

### 2. Java cannot resolve a real path

```text
java.nio.file.AccessDeniedException: ...\server-game.jar
at ... WindowsLinkSupport.getRealPath
at ... WindowsPath.toRealPath
```

In this case the archive could still be read:

```powershell
jar tf $file > $null
```

But Java's separate `Path.toRealPath()` check failed. These are different operations: being able to read a file does not prove that Java can resolve its real path.

### 3. Compiler failure during export

```text
javac.exe failed (exit code: 1) while exporting block behavior
```

The `export.log` contained `cannot find symbol` errors for Horizon classes, followed by an `AccessDeniedException` while closing the ZIP/JAR file system for `jtracy-1.14.38.jar`.

After further diagnosis and preparation targeting a different version, block-behavior export passed, but another stage failed.

### 4. Command registry export failure

```text
javac.exe failed (exit code: 1) while exporting command registries
```

The `command-registries-export.log` contained Minecraft registry-loading errors. It also showed OSHI/WMI system-information warnings and a `NullPointerException` while generating a crash report.

Do not automatically treat OSHI/WMI warnings as the root cause. First identify the original registry/data-loading error and verify the preparation paths.

## What was checked

- The directory and JAR existed.
- The JAR did not have the ReadOnly attribute.
- `jar tf` could read the JAR contents.
- The file was not a Windows reparse point / symlink.
- The user had Full Control on many nested folders.
- Java could resolve `C:\`, `C:\Users`, and `C:\Users\Elena`, but initially returned `AccessDeniedException` for `AppData\Roaming` and descendants.
- `Path.toRealPath()` and `File.getCanonicalPath()` behaved differently in a separate test: the former initially failed, while the latter returned a path.
- ACL inspection found an explicit user Deny rule including Delete and Read on `C:\Users\Elena`; `AppData\Roaming` also had explicit Delete Deny entries for the user and SYSTEM.
- WMI repository verification reported a consistent repository. Performance-counter checks did not indicate that PerfProc/PerfOS were disabled.
- The launcher was updated to 0.2.58.
- Preparation was run for the exact Nightly `nightly-2026-11-08-fdc87ae`, rather than only targeting the `v0.2.54` directory.

## Safe diagnostics

Open PowerShell. Reading logs and running ordinary diagnostic checks does not require blanket permissions. An elevated PowerShell window may be needed for some ACL inspection, but do not run permission-changing commands until you understand the results.

### 1. Locate the launcher executable

```powershell
$launcher = "$env:LOCALAPPDATA\Programs\Horizon Launcher\horizon-launcher.exe"
Test-Path $launcher
```

If it returns `True`, the executable was found.

### 2. Check the available commands

```powershell
& $launcher --help
```

CLI syntax can change between versions. Use the `--help` output from your installed build as the authority.

### 3. Check Java and the JAR

```powershell
java -version

$file = "$env:APPDATA\horizon-launcher\versions\nightly-2026-11-08-fdc87ae\.local\minecraft\26.3\jars\server-game.jar"
Test-Path $file
Get-Item -LiteralPath $file -Force | Format-List FullName,Length,Attributes,IsReadOnly
jar tf $file > $null
```

A successful `jar tf` confirms that Java could read the ZIP/JAR contents. It does not guarantee that `toRealPath()` will work.

### 4. Probe real-path resolution with Java

Create a Java source file:

```powershell
@'
import java.nio.file.Path;

public class RealPathProbe {
    public static void main(String[] args) {
        for (String value : args) {
            try {
                System.out.println("OK   " + Path.of(value).toRealPath());
            } catch (Exception e) {
                System.out.println("FAIL " + value);
                System.out.println("     " + e);
            }
        }
    }
}
'@ | Set-Content "$env:TEMP\RealPathProbe.java" -Encoding ascii
```

Run it against the path chain (replace the Nightly tag if needed):

```powershell
$base = "$env:APPDATA\horizon-launcher\versions\nightly-2026-11-08-fdc87ae"
$paths = @(
    "$env:APPDATA",
    "$env:APPDATA\horizon-launcher",
    "$base",
    "$base\.local",
    "$base\.local\minecraft",
    "$base\.local\minecraft\26.3",
    "$base\.local\minecraft\26.3\jars",
    "$base\.local\minecraft\26.3\jars\server-game.jar"
)

java "$env:TEMP\RealPathProbe.java" $paths
```

- `OK` means Java resolved that exact path.
- `FAIL` identifies the exact path where the check failed.

This is a diagnostic only. It does not modify anything.

### 5. Inspect ACLs and save a backup

```powershell
$profile = $env:USERPROFILE
$acl = Get-Acl -LiteralPath $profile
$acl.Sddl | Set-Content "$env:TEMP\profile-acl-backup.sddl" -Encoding ASCII

icacls "$env:USERPROFILE"
icacls "$env:USERPROFILE\AppData"
icacls "$env:APPDATA"

(Get-Acl -LiteralPath "$env:APPDATA") |
    Format-List Owner,AreAccessRulesProtected,Access
```

For a detailed view of each rule:

```powershell
$targets = @(
    "$env:USERPROFILE",
    "$env:USERPROFILE\AppData",
    "$env:APPDATA",
    "$env:APPDATA\horizon-launcher"
)

foreach ($p in $targets) {
    Write-Host "`n========== $p ==========" -ForegroundColor Cyan
    try {
        $a = Get-Acl -LiteralPath $p -ErrorAction Stop
        "OWNER: $($a.Owner)"
        "PROTECTED: $($a.AreAccessRulesProtected)"
        $a.Access | Select-Object IdentityReference,AccessControlType,
            FileSystemRights,IsInherited,InheritanceFlags |
            Format-Table -AutoSize -Wrap
    } catch {
        Write-Host "ACL ERROR: $($_.Exception.Message)" -ForegroundColor Red
    }
}
```

**Do not copy or automatically remove the Deny rules found in this case.** A Delete deny is not, by itself, a read denial. For every rule, inspect the exact rights, principal, inheritance, and scope. Removing a Deny entry is not guaranteed to fix `toRealPath()`.

## Prepare the exact Horizon version

The key practical detail in this case: prepare the same Nightly build that you intend to launch.

First list available versions:

```powershell
& $launcher versions
```

Find the exact installed Nightly tag in the output, then use that tag in the prepare command. Successful example from this case:

```powershell
& $launcher prepare --version nightly-2026-11-08-fdc87ae --minecraft "D:\minetcraft"
```

Here `D:\minetcraft` is **the path used in one case only**. Replace it with the actual location of your local Minecraft Java 26.3 installation.

Important notes:

- `prepare` is a launcher subcommand. Invoke `horizon-launcher.exe prepare`, not an unqualified `horizon-launcher prepare` command that may not exist in PATH.
- `--version` targets the selected installed version.
- For an offline/local-import scenario, point the launcher at your local Minecraft directory. Do not use a Mojang download path if you do not have the required Microsoft/Minecraft account or intentionally use a local import.
- Do not delete the entire Windows `AppData`, the Windows user profile, or the Horizon folder without a backup.

## Verify the result

After preparation, look for a successful final message similar to:

```text
Prepared Horizon native-client resources: ...\.local\horizon-client\26.3
Done in 32.8 s
nightly-2026-11-08-fdc87ae is prepared.
```

You can also check that the resource file exists:

```powershell
$resources = "$env:APPDATA\horizon-launcher\versions\nightly-2026-11-08-fdc87ae\.local\horizon-client\26.3\horizon-resources.json"
Test-Path $resources
```

Expected result: `True`. Then launch the same Nightly build from Horizon Launcher.

## Where to find logs

The prepare command may print the exact path to an export log. Open the path from the current error message.

Examples from this case:

```powershell
Get-Content "$env:APPDATA\horizon-launcher\versions\v0.2.54\.local\horizon-client-block-behavior\export.log" -Tail 100

Get-Content "$env:APPDATA\horizon-launcher\versions\nightly-2026-11-08-fdc87ae\.local\horizon-client\26.3\command-registries-export.log" -Tail 120
```

Paths depend on the launcher version and selected game version. Do not assume every user's log is in the same location.

You can also inspect the installed log command:

```powershell
& $launcher logs --help
```

## Common messages and next steps

| Message | Meaning / next step |
|---|---|
| `Horizon's prepared game files are missing` | Preparation for this version did not complete, or resources were generated for a different version profile. Check the selected tag and `horizon-resources.json`. |
| `AccessDeniedException` + `toRealPath` | Probe each parent directory with Java `toRealPath()`. Then inspect ACLs without granting blanket permissions. |
| `jar tf` works but `toRealPath` fails | These are different operations. Do not treat a successful JAR listing as proof that real-path resolution works. |
| `javac.exe failed` | Open the export log named in the error and inspect the first meaningful errors, not just the final stack-trace line. |
| `cannot find symbol` | May be caused by an incomplete/wrong classpath or incomplete preparation. The full export log is needed. |
| `Registry Loading`, `Registry must be non-empty` | Look earlier in the same log for the original vanilla-data/resource-loading error. Do not automatically assume an OSHI error is the root cause. |
| `Invalid Class Win32_...`, `PDH Counter`, OSHI `process is null` | Windows system-information collection may be failing and may interfere with crash-report generation. Separate this from the original Minecraft/Horizon error. |
| PowerShell cannot find `horizon-launcher` | Invoke the executable by its full path with the call operator `&`. |

## What not to do

- Do not run `icacls C:\Users\... /reset /T` without understanding the consequences.
- Do not grant `Everyone:F` to the whole profile or drive.
- Do not delete Windows `AppData` or rename the entire user profile.
- Do not delete `D:\minetcraft`; the local source installation may be needed for preparation.
- Do not treat every `AccessDeniedException` as identical: inspect the exact path and operation.
- Do not claim that Deny/Delete was definitely the cause. It was observed in this case, but the attempted ACL modification did not complete.

## Successful-case timeline

1. Initial launch reported that `horizon-resources.json` was missing.
2. Windows ACL and JAR checks did not show a simple JAR-read failure.
3. `prepare` ran but failed during block-behavior export, then during command-registry export.
4. Logs showed Java `toRealPath()` failures for paths under `AppData\Roaming`.
5. A standalone Java test was created to probe every parent path and the JAR.
6. On a later run, all eight probed paths returned `OK`.
7. `prepare` was run with explicit `--version nightly-2026-11-08-fdc87ae` and the local Minecraft directory.
8. All export stages completed and Horizon reported that the Nightly was prepared.
9. The game launched successfully.

This timeline is a useful diagnostic case study, not proof that one specific ACL flag was the sole cause.

## If it still does not work

When opening an issue, remove personal data. Redact Windows account names, private local paths, account names, tokens, and other secrets.

Useful information to include:

- Horizon Launcher version and exact Nightly tag.
- Windows version and `java -version` output.
- Exact error text and 50–150 lines from the relevant export log.
- `RealPathProbe` output with the Windows username redacted.
- ACL/`icacls` output for affected directories, with personal names redacted.
- Whether you use a locally imported installation or the official sign-in flow.

---

**Disclaimer:** Horizon is an independent project. This repository is a community troubleshooting note and is not affiliated with or endorsed by the Horizon developers or Mojang/Microsoft.
