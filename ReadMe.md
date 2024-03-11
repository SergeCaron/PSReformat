# Reformatting PowerShell scripts and data files

This script implements a fixed set of coding standards for PowerShell scripts.

The formatting rules are defined in the data file *FormattingRules.psd1* and must be placed in the same directory as the *PSReformat.ps1* script. The rules are documented *[here](https://github.com/PowerShell/PSScriptAnalyzer/tree/master/docs/Rules)*.

This is a Windows-oriented script:
- the input file is presumed coded UTF-8 and the output file is explicitly coded UTF-8 without a Byte Order Marker (BOM). On input, your editor of choice may not automatically change the file encoding to UTF-8 : see this *[UTF-8 Debugging Chart](https://www.i18nqa.com/debug/utf8-debug.html)* for tell-tale signs of corruption.
- by default, line delimiters are changed to LF. When the script is invoked from the terminal command line, there is a -UseCRLF switch to enable CFLF as the line delimiter. Note that Git may revert LF to CRLF.

## Outside VScode
You may not want to install VSCode just for the purpose of reformatting PowerShell scripts or your VSCode workspace may have implemented a different coding standard than what is expected in this project.
This *PSReformat.ps1* script relies on the PSScriptAnalyzer utility module *[PS-ScriptAnalyzer](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview?view=ps-modules)* to validate and reformat PowerShell scripts.

The utility module is installed in your environment using (you may have to also install the *Nuget* provider):
```
Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force
Install-Module -Name PSScriptAnalyzer -Force
```
On entry, the script validates the formatting rules and open a file browser to select what will be reformatted. The output file (if any) is placed in the same directory, implying write access and file creation privileges. The prefix "Reformatted" is added to the original file name.

The script performs a simple diff between the source and output files and shows the first and last lines of any difference: when there are no differences, this serves as a validation that the source script conforms to the formatting rules.

## Inside VSCode


The default PowerShell extension in VSCode allows reformatting a document using the Shift+Alt+F command or the *Format Document* context menu. This extension contains a hidden implementation of the PSScriptAnalyzer module which cannot be invoked outside VSCode.

The *PSReformat.ps1* script can also run from a VSCode terminal window under the same conditions as outlined in [Outside VSCode](#outside-vscode).

The following VSCode settings for this extension are the equivalen of the formatting rules defined in the data file *FormattingRules.psd1* supplied with this script:

```
{
    "powershell.codeFormatting.autoCorrectAliases": true,
    "powershell.codeFormatting.avoidSemicolonsAsLineTerminators": true,
    "powershell.codeFormatting.pipelineIndentationStyle": "IncreaseIndentationForFirstPipeline",
    "powershell.codeFormatting.preset": "Stroustrup",
    "powershell.codeFormatting.trimWhitespaceAroundPipe": true,
    "powershell.codeFormatting.useCorrectCasing": true,
}
```
VSCode implements a *Compare Selected* in the context menu of the Explorer view. This implies that you save the reformatted text in the same tree just for the purpose of checking code conformity. This condition is the same when using the *PSReformat.ps1* script.

## GitHub actions

The behavior of this *PSReformat.ps1* script is reproduced in the project Pull request. The following rules will validate all PowerShell scripts checked out from your project under the same conditions as outlined in [Outside VSCode](#outside-vscode):

```
      - name: Install PSScriptAnalyzer module
        shell: pwsh
        run: |
          Set-PSRepository PSGallery -InstallationPolicy Trusted
          Install-Module PSScriptAnalyzer -Force -ErrorAction Stop
          Write-Output $("Using Powershell version: " + $PSVersionTable.PSVersion.Major.ToString() + "." + $PSVersionTable.PSVersion.Minor.ToString())

      # Verify the rules set
      - name: Validate FormattingRules
        shell: pwsh
        run: |
          Invoke-ScriptAnalyzer -Path FormattingRules.psd1 -Recurse -Outvariable ProjectIssues
          $Glitches = $ProjectIssues.Where({$_.Severity -eq 'Error' -or $_.Severity -eq 'Warning'})
          if ($Glitches.Count) {  Write-Error "There are $($Glitches.Count) errors and warnings in FormattingRules.psd1." -ErrorAction Stop  }

      # Validate PowerShell scripts
      - name: Verify PowerShell script formatting
        shell: pwsh
        run: |
          ForEach ($File in $(Get-ChildItem -Filter *.ps1 -File -Name)) {
          Write-Host "Processing $File ..."
          $Script = $($(Get-Content $File -Encoding UTF8) -join "`n") + "`n"
          $Revision = Invoke-Formatter -ScriptDefinition $Script -Settings FormattingRules.psd1
          $Output = New-Object Text.UTF8Encoding
          # Prior to PowerShell V6
          # Set-Content -Path $File -Force -Value $Output.GetBytes($Revision) -Encoding Byte
          # Current PowerShell V7.x on GitHub
          Set-Content -Path $File -Force -Value $Output.GetBytes($Revision) -AsByteStream
          }
          git diff --exit-code

```
