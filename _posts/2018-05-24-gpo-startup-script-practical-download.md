---
layout: post
title: GPO Startup Script Practical Download
# date: 0-02-07 20:46
author: VertigoRay
comments: true
categories: [Group Policy]
---
# GPO Startup Script - Practical Download

I like logs of things I'm doing, so a simple log for something like this is to use PowerShell's `Start-Transcript` function.
It should be the first command that you run.
We'll also need to send stuff to the log; `Write-Host` or `Write-Output` will do the job well.
Here's the download command that we use, including a legit `$path`:

```powershell
Start-Transcript -LiteralPath ('{0}\Logs\Download-HiddenPowershell.ps1.log' -f $env:SystemRoot) -IncludeInvocationHeader -Force

$path = ('{0}\UNT' -f $env:SystemRoot)
$uri = 'https://raw.githubusercontent.com/UNT-CAS/HiddenPowershell/v1.0/HiddenPowershell.vbs'

Write-Host ('# Path: {0}' -f $path)
Write-Host ('# URI: {0}' -f $uri)

Write-Host '# Ensure Path Exists ...'
New-Item -Type 'Directory' -Path $path -Force

Write-Host ('# Net.ServicePointManager: {0}' -f [Net.ServicePointManager]::SecurityProtocol)
Write-Host '# Setting TLS 1.2 ...'
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Write-Host ('# Net.ServicePointManager: {0}' -f [Net.ServicePointManager]::SecurityProtocol)

Write-Host '# Downloading from URI ...'
Invoke-WebRequest -Uri $uri -OutFile ('{0}\HiddenPowershell.vbs' -f $path) -UseBasicParsing -Verbose 4>&1

Stop-Transcript
```

## Issue

The problem with deploying this via a GPO Startup Script with [PowerShell's `-Command` parameter](https://docs.microsoft.com/en-us/powershell/scripting/core-powershell/console/powershell.exe-command-line-help) is that **GPO's *Script Parameter* has a limit: 520 characters.**
This command/script is currently 902 characters and that's just the command; we still have to add the rest of the powershell parameters: `-ExecutionPolicy Bypass -NoProfile -NonInteractive -WindowStyle Hidden -Command "..."`
That's 82 addition characters.

## Resolution

There are two ways, that I know of, to reduce the size of this command:

1. Use/Create [PowerShell Aliases](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-alias).
1. Use just enough of a function's parameter name to be unique.
   - i.e.: [The `Invoke-WebRequest` function's](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) `UseBasicParsing` parameter can be shortened to `-UseB` (case-insensitive of course); not `-U` because there's also a `UserAgent` parameter and the first three letters of both parameters are the same.

Let's start trimming things down ...

### PowerShell Command Line parameters

PowerShell's command line parameters follow the same rule shortening as as functions.
So here's what we want to shorten; 82 characters not including the ellipsis:

```powershell
-WindowStyle Hidden -ExecutionPolicy Bypass -NoProfile -NonInteractive -Command "..."
```

Here's the shortest we can make it; 24 characters not including the ellipsis:

```powershell
-W H -Ex B -NoP -NonI "..."
```

*Note: The `-Command` parameter is assumed.*
*Parameter values that have a set list of possible values are auto-completed, just like the parameter name.*

### Adjust Code Using Aliases and Shortened Parameters

If we're going to create an alias:

- Be sure it's not already in use.
- Be sure we're using the function enough times to make a difference.

Take advantage of positional parameters when code shortening.

Here's the adjusted code block from the top of this article:

```powershell
$a='HiddenPowershell'
$b=$env:SystemRoot
Start-Transcript $b'\Logs\Download-'$a'.ps1.log' -I -F

$c=$b+'\UNT'
$d='https://raw.githubusercontent.com/UNT-CAS/{0}/v1.0/{0}.vbs' -f $a

'# Path: '+$c
'# URI: '+$d

'# Ensure Path Exists …'
ni -I D $c -F

$e=[Net.ServicePointManager]::SecurityProtocol
$f='Net.ServicePointManager'
'# '+$f+': '+$e
'# Setting TLS 1.2 …'
$e=[Net.SecurityProtocolType]::Tls12
'# '+$f+': '+$e

'# Downloading from URI …'
iwr $d -O $c'\'$a'.vbs' -UseB -V 4>&1

Stop-Transcript
```

Turns out I didn't need to make any aliases.
If I needed to, I would have done so something like this: `nal w write`.
Explained:

- `nal` is an alias for `New-Alias`
- `w` is my alias name
- `write` is an alias for `Write-Output`
  - Yes, it passes through.

I could have shortened the strings even more, but I can only use single quotes inside of the command because the whole command needs to be passed inside of double quotes. There's a few options for a putting a variable in a string; the first option can't be done and the third is the shortest, in this case, so we used it:

- `"$b\Logs\Download-$a.ps1.log"`
- `('{0}\Logs\Download-{1}.ps1.log' -f $b,$a)`
- `$b'\Logs\Download-'$a'.ps1.log'`
  - Can only use this one when passing to a parameter.
- `$b+'\Logs\Download-'+$a+'.ps1.log'`

I switch to using the ellipsis ascii character (`…`; `&#8230;`) instead of three dots (`...`). Super handy to know that exists.

*Note: Consider shortening the URL with your favorite URL shortening service; such as [Google URL Shortener](https://goo.gl).*
*If you make the URL within an account, you can usually see click/download counts.*

*Note: I kept the empty lines in there for keeping a readabile comparison, but I will remove them before proceeding to the next section.*

### Final Command

So, let's use our command, from the previous section, but reduced down even more; 492 characters:

```powershell
$a='HiddenPowershell';$b=$env:SystemRoot;Start-Transcript $b'\Logs\Download-'$a'.ps1.log' -I -F;$c=$b+'\UNT';$d='https://raw.githubusercontent.com/UNT-CAS/{0}/v1.0/{0}.vbs' -f $a;'# Path: '+$c;'# URI: '+$d;'# Ensure Path Exists …';ni -I D $c -F;$e=[Net.ServicePointManager]::SecurityProtocol;$f='Net.ServicePointManager';'# '+$f+': '+$e;'# Setting TLS 1.2 …';$e=[Net.SecurityProtocolType]::Tls12;'# '+$f+': '+$e;'# Downloading from URI …';iwr $d -O $c'\'$a'.vbs' -UseB -V 4>&1;Stop-Transcript
```

You probably want to test it to make sure it's working.
Here's how I do it:

```powershell
$command = @'
$a='HiddenPowershell';$b=$env:SystemRoot;Start-Transcript $b'\Logs\Download-'$a'.ps1.log' -I -F;$c=$b+'\UNT';$d='https://raw.githubusercontent.com/UNT-CAS/{0}/v1.0/{0}.vbs' -f $a;'# Path: '+$c;'# URI: '+$d;'# Ensure Path Exists …';ni -I D $c -F;$e=[Net.ServicePointManager]::SecurityProtocol;$f='Net.ServicePointManager';'# '+$f+': '+$e;'# Setting TLS 1.2 …';$e=[Net.SecurityProtocolType]::Tls12;'# '+$f+': '+$e;'# Downloading from URI …';iwr $d -O $c'\'$a'.vbs' -UseB -V 4>&1;Stop-Transcript
'@
Invoke-Expression $command
```

If that runs as expected, we can run `$command` via a powershell.exe command line parameter:

```powershell
powershell.exe -W H -Ex B -NoP -NonI "$a='HiddenPowershell';$b=$env:SystemRoot;Start-Transcript $b'\Logs\Download-'$a'.ps1.log' -I -F;$c=$b+'\UNT';$d='https://raw.githubusercontent.com/UNT-CAS/{0}/v1.0/{0}.vbs' -f $a;'# Path: '+$c;'# URI: '+$d;'# Ensure Path Exists …';ni -I D $c -F;$e=[Net.ServicePointManager]::SecurityProtocol;$f='Net.ServicePointManager';'# '+$f+': '+$e;'# Setting TLS 1.2 …';$e=[Net.SecurityProtocolType]::Tls12;'# '+$f+': '+$e;'# Downloading from URI …';iwr $d -O $c'\'$a'.vbs' -UseB -V 4>&1;Stop-Transcript"
```

Yah!!
That's **516** characters of arguments!!
We can event put the URL in a shortener, as previously suggested, to get it down to **479** characters:

```powershell
powershell.exe -W H -Ex B -NoP -NonI "$a='HiddenPowershell';$b=$env:SystemRoot;Start-Transcript $b'\Logs\Download-'$a'.ps1.log' -I -F;$c=$b+'\UNT';$d='https://goo.gl/PZcPxi' -f $a;'# Path: '+$c;'# URI: '+$d;'# Ensure Path Exists …';ni -I D $c -F;$e=[Net.ServicePointManager]::SecurityProtocol;$f='Net.ServicePointManager';'# '+$f+': '+$e;'# Setting TLS 1.2 …';$e=[Net.SecurityProtocolType]::Tls12;'# '+$f+': '+$e;'# Downloading from URI …';iwr $d -O $c'\'$a'.vbs' -UseB -V 4>&1;Stop-Transcript"
```

Now, just put it in GPO's *Script Parameters* field; as shown:

![GPO Startup Script Properties](https://i.imgur.com/iztADwi.png)

## Notes

- If you don't have room for it, remove the `Stop-Transcript` from the end.
  - It just gives a nice log footer with an *end time*; thus you can calculate total run time, if you so desire.
  - Additionally, you don't *have* to do logging at all.
- If you have a longer PowerShell script that you want run on mobile devices, consider:
  - Drop the script in a repo.
    - Public: github.com or gist.github.com work great.
    - Private: gitlab.com works great. Just make a [Personal Access Token](https://gitlab.com/profile/personal_access_tokens) and [tack it on the end of the URL](https://gitlab.com/help/api/README.md#personal-access-tokens) for the *raw* download, like this:
      - `https://gitlab.com/UNT-CAS/StartupScripts/raw/master/deploy.ps1?private_token=9koXpg98eAheJpvBs5tK`
  - Download the script with the first Startup Script.
    - Be sure to download the *raw* version of the script.
  - Execute it with a second Startup Script.
    - GPO Startup Scripts are run in order; hence the ability to order them.