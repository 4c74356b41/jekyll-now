---
id: 5775
title: SQL Server 2016 AlwaysOn automated deployment on Azure
date: 2017-09-04T11:40:00+00:00
author: rootilo
layout: post
guid: http://4c74356b41.com/post5775
permalink: /post5775
categories:
  - Azure
  - SQL
---

Hey folks,

I needed to create 2016 SQL AlwaysOn with Windows Server 2016 automated deployment for Azure. I took the [working, but outdated (sql2014-ws2012r2) example](https://github.com/Azure/azure-quickstart-templates/tree/master/sql-server-2014-alwayson-existing-vnet-and-ad) and modified it to suit my needs.  
Changes made:
1. It works with 2016sp1\2016 (at least at the time of writing)
2. It is using cloud witness (so 1 less vm to pay for)
3. I've removed as much legacy code as I could (without having to rewrite everything)
4. It runs in parallel (so all the dsc extensions start at the same time and work out their way)
5. Minor code clean-ups

xSqlServer module is the thing I've been using, but there are many issues with it, unfortunately. I've had to manually modify at least one place with a fix to make this work, I didn't have to investingate the issue because its documented in issues on the github, still it doesnt work without the workaround ;). I've had a few back and forths with the [module maintainer on twitter](https://twitter.com/johanljunggren) to get some basic idea about different things (I'm not really a sql guy).

**SQL2016SP1-WS2016** and do not work:
https://github.com/chagarw/MDPP/tree/master/301-sql-alwayson-md  
https://github.com/robotechredmond/301-sql-alwayson-md  
**SQL2016-WS2012** does work, but won't with **SQL2016SP1-WS2016**:
https://github.com/mspnp/reference-architectures/tree/master/sharepoint/sharepoint-2016

The below is the first error you will run into when trying to deploy 2016\2016 examples above :), you will encounter many other errors after fixing this one ;).
```
[[xSqlEndpoint]SqlAlwaysOnEndpoint] Creating database mirroring endpoint for SQL AlwaysOn ...
VERBOSE: [2017-09-01 07:53:32Z] [ERROR] The input object cannot be bound to any parameters for the command either because the command does not take pipeline input or the input and its properties do not match any of the parameters that take pipeline input.

[[xSqlEndpoint]SqlSecondaryAlwaysOnEndpoint_ttt-sql-1] Creating database mirroring endpoint for SQL AlwaysOn ...
VERBOSE: [2017-09-01 07:54:15Z] [ERROR] The input object cannot be bound to any parameters for the command either because the command does not take pipeline input or the input and its properties do not match any of the parameters that take pipeline input.
```

I've had to alter several dsc modules here and there to get this thing working end to end. I don't remember all of them at this point (probably shoulda take notes next time), sadly. I've had to alter `xSqlCreateVirtualDataDisk` to work with 2016 (as it didn't), this can be replaced with something like cdisk or cdisktools (but I'm unsure if they are in better shape), `xSQLServerAlwaysOnAvailabilityGroup` or `xSQLServerAlwaysOnAvailabilityGroupListener` has a bug described [here](https://github.com/PowerShell/xSQLServer/issues/649) and [here's the link to a branch](https://github.com/johlju/xSQLServer/tree/fix-issue-649) with a fix.

Among other top finds:

1. xDatabase doesn't work with sql2016 (what year is this?). [Fix](https://github.com/PowerShell/xDatabase/pull/31), but nobody cares.
2. xWebDeploy doesn't work at all (or how does it?).

Links:
1. [DSC modules](https://github.com/AvyanConsultingCorp/PCI_Reference_Architecture/tree/master/artifacts/configurationscripts)
2. [ARM Template](https://github.com/AvyanConsultingCorp/PCI_Reference_Architecture/blob/master/templates/resources/application/azuredeploy.json). Note: Arm template that deploys sql needs to have vnet and domain already in place. Also, you will get a hard time trying to deploy this template outside of that framework, so probably just copy out dsc extension\dsc scripts and use in your deployments.
3. [Parameters file example](https://github.com/AvyanConsultingCorp/PCI_Reference_Architecture/blob/master/templates/resources/azuredeploy.parameters.json)

Update:  
I switched to using `Microsoft.SqlServer.Management\SqlIaaSAgent` extension and that does allow me to get rid of all the xSql stuff and retain functionality. So right now I'm using only xSqlServer, which is really nice, as it appears to work in a more consistent fashion.
 
 Another Update:  
 Turns out the `SqlIaaSAgent` is setting SQL Server Service account to a local one and AlwaysOn requires domain service accounts (or certificate auth), so I had to add additional steps to fix SPN's and change service account to domain one.
 
