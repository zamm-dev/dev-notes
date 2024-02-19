# PowerShell

## Environment variables

Instead of

```bash
$ echo $SCREENSHOTS_BASE_DIR
```

in Powershell, you do

```powershell
$Env:SCREENSHOTS_BASE_DIR
```

to check on the value of an environment variable.

## Autocomplete setup

We try setting up autocomplete for PowerShell by running this command as an Administrator:

```powershell
Install-Module -Name PSReadLine -Scope CurrentUser -Force -AllowClobber
```

IF a dialog box pops up asking if you'd like to install NuGet, click "Yes."

Once that command finishes, set PSReadLine to use the history search feature:

```powershell
$ Set-PSReadLineOption -HistorySearchCursorMovesToEnd
Set-PSReadLineOption : The 'Set-PSReadLineOption' command was found in the module 'PSReadLine', but the module could not be loaded. For more information, run 'Import-Module PSReadLine'.
At line:1 char:1
+ Set-PSReadLineOption -HistorySearchCursorMovesToEnd
+ ~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (Set-PSReadLineOption:String) [], CommandNotFoundException
    + FullyQualifiedErrorId : CouldNotAutoloadMatchingModule
 
```

We run that command as requested:

```powershell
$ Import-Module PSReadLine
Import-Module : File C:\Users\Amos Ng\Documents\WindowsPowerShell\Modules\PSReadLine\2.3.4\PSReadLine.psm1 cannot be loaded because running scripts is disabled on this system. For more information, see about_Execution_Policies at https://go.microsoft.com/fwlink/?LinkID=135170.
At line:1 char:1
+ Import-Module PSReadLine
+ ~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : SecurityError: (:) [Import-Module], PSSecurityException
    + FullyQualifiedErrorId : UnauthorizedAccess,Microsoft.PowerShell.Commands.ImportModuleCommand

```

We go to the website as requested, and try

```powershell
$ Set-ExecutionPolicy -ExecutionPolicy Unrestricted
```

Now it succeeds, but we realize ChatGPT has misunderstood what we asked for. We find [this post](https://www.hanselman.com/blog/adding-predictive-intellisense-to-my-windows-terminal-powershell-prompt-with-psreadline) and try to follow the instructions there:

```powershell
$ code $profile
```

and put in

```powershell
Import-Module PSReadLine
Set-PSReadLineOption -PredictionSource History
Set-PSReadLineOption -PredictionViewStyle ListView
Set-PSReadLineOption -EditMode Windows
```

Now autocomplete text starts appearing, but when we try to run `Ctrl-E` as on Unix, we find that nothing happens. Same thing happens for tab. We find [this page](https://devblogs.microsoft.com/powershell/announcing-psreadline-2-1-with-predictive-intellisense/?WT.mc_id=-blog-scottha) and find out that we need to

```powershell
Set-PSReadLineKeyHandler -Chord "Ctrl+e" -Function ForwardChar
```

Now it works as we are used to on Unix! We add this to `code $profile` as before.
