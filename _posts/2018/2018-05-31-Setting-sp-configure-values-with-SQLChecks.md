---
layout: post
title: Setting sp_configure values with SQLChecks
share-img: http://tjaddison.com/assets/2018/2018-05-31/TODO.jpg
---

As of v1.0 SQLChecks now contains a command which allows you to take a file that documents a server configuration and apply that to a server.  The configuration file is the same one which is used by Pester tests ([perhaps in combination with some like dbachecks](https://github.com/taddison/dbachecks-wrapper)), which makes it doubly valuable to document your server configuration (set it and ensure it stays set!).

In order to apply the configuration to a single server you would run the following (note that SQLChecks configuration files contain the instance name, which is why we don't have to specify a server):

```powershell
  Get-Content -Path "c:\configs\proddb1.config.json" -Raw | ConvertFrom-Json -OutVariable config | Out-Null
  Set-SpConfig -Config $config
}
```

Assuming there were a few changes to make, this gives you the following output:

###TODO: PICTURE###

If there are no changes, you'll get a message that nothing was updated.
<!--more-->

You can easily apply changes to a whole estate of servers by using the below script, which will find every config file in a folder (and all children) and apply the `sp_configure` values to the servers.

```powershell
$configs = Get-ChildItem -Filter *.config.json -Recurse
foreach($file in $config) {
  Get-Content -Path ".\examples\SingleCheck\localhost.config.json" -Raw | ConvertFrom-Json -OutVariable config | Out-Null
  Set-SpConfig -Config $config
}
```

Under the hood this command leverages the excellent [dbatools](https://dbatools.io/), specifically the commands [Get-DbaSpConfigure](https://dbatools.io/functions/get-dbaspconfigure/) and [Set-DbaSpConfigure](https://dbatools.io/functions/set-dbaspconfigure/).  The latter command will also run the `reconfigure` statement after the conifguration value is change.

###CHECK THE ABOVE THING ABOUT ALTER IS TRUE###
###ALSO CHECK IT ISNT DOG SLOW ALTERING LOTS OF THINGS###

>Note the command compares the configured value against the expected value - if the configured value is correct but the runtime value is wrong then this will neither fail the Pester tests, nor update the value when using `Set-SpConfig`.