---
layout: post
title: Ensuring your Describe Tags are unique in Pester tests
share-img: https://tjaddison.com/assets/2018/2018-07-21/DescribeTagsAppVeyor.png
---

The name of each test in SQLChecks is used as both the setting name in the configuration files, and to tag the Describe block.  After seeing the benefit of fine-grained control over test execution (from in Claudio Silva's post [dbachecks - a different approach...]) this method of test invocation became the preferred way to leverage the SQLChecks library:

```powershell
$sqlCheckConfig = Read-SqlChecksConfig -Path $sqlChecksConfigPath
foreach($check in $sqlCheckConfig | `
          Get-Member -Type NoteProperty | `
          Where-Object { $_.Name -ne "ServerInstance"} | `
          Select-Object -ExpandProperty Name) {

  # Note we invoke by -Tag $check - a test with no tag will never get invoked
  Invoke-SqlChecks -Config $sqlCheckConfig -Tag $check
}
```

There isn't yet a convention for how to name a test, and we've already had some tests built with pretty-similar sounding names - it is only a matter of time before we get a duplicate.  To prevent duplicate tests accidentally getting checked in, I recently added a test that parses the Pester test files and ensures that each tag is not just unique within the file, but globally unique within SQLChecks.

![Describe tags test on AppVeyor](/assets/2018/2018-07-21/DescribeTagsAppVeyor.png)

You can find the [full test on GitHub][tag uniqueness test on GitHub], or read on for an explanation of how it is implemented.  For a more thorough exploration of tests you can run on Describe blocks see SQLDBAWithABeard's blog [Using the AST in Pester for dbachecks]
 (which inspired this test), or the TechNet post [learn how it pros can use the PowerShell AST].

<!--more-->

## Overview

At a high level the test:

- Finds all of the Pester test files that are shipped with the module
- Get the content of each file (we use `-Raw` to get the full text, not a collection of lines)
- Parses each file so we can interact with them as objects (e.g. Function, Describe block, Operator) rather than strings of text
- Finds every Describe block
- Records the tag (if it exists) in an array of all tags
- Loops through the array of tags and checks if any of them has a count greater than 1

This gives the Pester test a fairly useful output (`Tag X is a duplicate`) rather than (`There are duplicate tags`).  If your tests are going to fail you want them to be maximally helpful in debugging the failure.

Most of the requirements (get all the files, loop through the array) are fairly forward PowerShell - if we take out the parsing code the test looks like this:

```powershell
Describe "Module test Describe tags are unique" {
  $tags = @()

  Get-ChildItem -Filter *.tests.ps1 -Path $PSScriptRoot\..\src\SQLChecks\Tests | Get-Content -Raw | ForEach-Object {
      #TODO: Add the tag to the array
  }

  foreach($tag in $tags) {
      It "$tag is a unique tag within the module" {
          ($tags | Where-Object {$_ -eq $tag}).Count | Should Be 1
      }
  }
}
```

## Parsing PowerShell

Luckily for us parsing PowerShell is a well-trodden path and we have some excellent tools available, with the [Parser][Parser class on MSFT docs] allowing us to take a string and deconstruct it into an [Abstract Syntax Tree] (AST).  The tree structure turns our script into an incredibly rich object graph that we can interact with and query, and is one of the first steps taken by a compiler to take a script (in any language) and execute it.

We build the AST by piping the content from our files to it (this would sit on the inside of the `Get-Content | ForEachObject {` block)

```powershell
$ast = [Management.Automation.Language.Parser]::ParseInput($_, [ref]$null, [ref]$null)
```

>The `[ref]$null`s are used as the `ParseInput` has two additional mandatory parameters - in our case we don't care about saving the tokens or errors (see the [ParseInput documentation] for more details)

Once the AST is built we can then run a query to find all nodes that satisfy a set of predicates.  In our case we want to find:

- Every Describe command (Remember, Describe is a PowerShell function!)
- Where there is a second parameter (we'll assume the first parameter is the description, e.g. `Describe "Some Test"`)
- Where that second parameter is called Tag

And once we have found every Describe command that satisfies those predicates, we want to take the fourth element (called the `CommandElement`) which will be the Tag value.  Translated into PowerShell our query looks like this (remember the find result will produce multiple results, so we have to harvest the tag from each of them):

```powershell
$ast.FindAll({
          param($node)
          $node -is [System.Management.Automation.Language.CommandAst] -and
          $node.CommandElements[0].Value -eq "Describe" -and
          $node.CommandElements[2] -is [System.Management.Automation.Language.CommandParameterAst] -and
          $node.CommandElements[2].ParameterName -eq "Tag"
      }, $true) | ForEach-Object {
          $tags += $_.CommandElements[3].Value
      }
```

The FindAll functions takes a predicate function which should return `$true` if the node matches.  The first line of our predicate (`...-is [System...`) limits our search to commands only (not comments, blocks, etc.).

> You'll note we don't check there is a fourth parameter - this is invalid syntax (missing parameter value) and that sounds like another test that could be written.

## The complete test

```powershell
Describe "Module test Describe tags are unique" {
  $tags = @()

  Get-ChildItem -Filter *.tests.ps1 -Path $PSScriptRoot\..\src\SQLChecks\Tests | Get-Content -Raw | ForEach-Object {
      $ast = [Management.Automation.Language.Parser]::ParseInput($_, [ref]$null, [ref]$null)
      $ast.FindAll({
          param($node)
          $node -is [System.Management.Automation.Language.CommandAst] -and
          $node.CommandElements[0].Value -eq "Describe" -and
          $node.CommandElements[2] -is [System.Management.Automation.Language.CommandParameterAst] -and
          $node.CommandElements[2].ParameterName -eq "Tag"
      }, $true) | ForEach-Object {
          $tags += $_.CommandElements[3].Value
      }
  }

  foreach($tag in $tags) {
      It "$tag is a unique tag within the module" {
          ($tags | Where-Object {$_ -eq $tag}).Count | Should Be 1
      }
  }
}
```

If you've not worked with an AST or parser before (or you have but not in PowerShell) some of this might look a little intimidating (it certainly took me a while to grok it), but I'd encourage you to persevere as the idea is incredibly powerful.  Also, it's pretty cool to have Pester tests for your Pester tests.

![I heard your like Pester tests](/assets/2018/2018-07-21/YoDawg.jpg)

## Adding more tests

Once we have the framework for parsing a test we can then add additional tests fairly easy - the below is a full example that will check every test (`Describe` block) has a tag parameter.  The only changes are we perform our checks per-describe block (inside a context so you can easily find the failure), and we've moved the tests for parameter/parameter value out of the `FindAll` and onto the other side of a `Should` test.

```powershell
Describe "Every test has a tag" {
    Get-ChildItem -Filter *.tests.ps1 -Path $PSScriptRoot\..\src\SQLChecks\Tests | ForEach-Object {
        $content = Get-Content $_.FullName -Raw
        Context "$_" {
            $ast = [Management.Automation.Language.Parser]::ParseInput($content, [ref]$null, [ref]$null)
            $ast.FindAll({
                param($node)
                $node -is [System.Management.Automation.Language.CommandAst] -and
                $node.CommandElements[0].Value -eq "Describe"
            }, $true) | ForEach-Object {
                It "$($_.CommandElements[1]) has a tag" {
                    $_.CommandElements[2] -is [System.Management.Automation.Language.CommandParameterAst] -and
                        $_.CommandElements[2].ParameterName -eq "Tag" | Should Be $true
                }
            }
        }
    }
}
```

Which looks something like this:

![Every test has a tag](/assets/2018/2018-07-21/EveryTestHasATag.png)

[dbachecks - a different approach...]: https://claudioessilva.eu/2018/02/22/dbachecks-a-different-approach-for-an-in-progress-and-incremental-validation/
[tag uniqueness test on GitHub]: https://github.com/taddison/SQLChecks/blob/master/tests/SQLChecks.Module.tests.ps1
[Using the AST in Pester for dbachecks]: https://sqldbawithabeard.com/2018/01/15/using-the-ast-in-pester-for-dbachecks/
[learn how it pros can use the PowerShell AST]: https://blogs.technet.microsoft.com/heyscriptingguy/2012/09/26/learn-how-it-pros-can-use-the-powershell-ast/
[Parser class on MSFT docs]: https://docs.microsoft.com/en-us/dotnet/api/system.management.automation.language.parser?view=powershellsdk-1.1.0
[Abstract Syntax Tree]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
[ParseInput documentation]: https://docs.microsoft.com/en-us/dotnet/api/system.management.automation.language.parser.parseinput?view=powershellsdk-1.1.0