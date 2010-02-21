ModuleWriteError
================

Overview
--------
Provides a New-ModuleError function for the creation of an error record that is
populated and formatted with an imported module's advanced function information.

Generates a module specific custom exception on the fly and creates Error Records
using built-in exception types by the short, instead of fully qualified, exception
type string.

Reasoning
---------
The built-in Write-Error command can be fairly mis-leading or just plain wrong
at times which makes getting useful information from a normal user more difficult
than it needs to be, and makes debugging harder too.

- Write-Error always shows 'Write-Error' which makes it look like there was an
  error while trying to write.
- It can report two bugs instead of one. [MS Connect Bug](https://connect.microsoft.com/PowerShell/feedback/details/524467/)
- Usage of prefixes when importing a module does not carry through to error/exceptions. [MS Connect Bug](https://connect.microsoft.com/PowerShell/feedback/details/523964)

- It is nice to:
 - create exceptions using the short name instead of the fully qualified name.
 - have a stack-trace!
 - be able to handle exceptions like a Try/Catch without having to use terminating errors.


Usage
-----
This module is meant to be called from within your own PowerShell modules.
Simply place the ModuleWriteError folder and contents in your `$Env:PSModulePath` path.

Add the following code block to your module (.psm1) file. _Typically after all of your code blocks and prior to your module initializing function call._

    if (! (Get-Module ModuleWriteError) ) {
        Import-Module "ModuleWriteError" -Global -ErrorAction Stop
    }

Demo
----
A little demo module is included that shows how the output to the user will look
using different common parameters (ErrorAction and ErrorView).


Example Differences
-------------------

---

###Message generation command:
####_Write-Error (close to what we want):_

	Write-Error -Exception System.Management.Automation.ItemNotFoundException `
		-Category ObjectNotFound `
		-Message "Cannot get `'$Name`' because it does not exist.`n" `
		-TargetObject $name `
		-CategoryActivity "Get-Foo" `
		-CategoryReason "NormalWriteError"

You _can_ just pass Write-Error an error record, but that just introduces new issues._

####_New-ModuleError message:_

	$MyErrorRecord = (
		New-ModuleError -ErrorCategory ObjectNotFound `
		-message "Cannot get `'$Name`' because it does not exist." `
		-identifierText FooObjectNotFound -targetObject $($Name)
	)
	$PSCmdlet.WriteError( $MyErrorRecord )

---

###Message Output
####_Write-Error:_

><pre><code>Set-Foo : Cannot set 'bar' because it does not exist.
At C:\Users\almostautomated\datastore\repos\git\ModuleWriteError\ModuleWriteError.demo.psm1:141 char:10
+         Set-Foo <<<<  bar -usePSWriteError
    + CategoryInfo          : ObjectNotFound: (bar:String) [Write-Error], NormalWriteError
    + FullyQualifiedErrorId : System.Exception,Set-Foo
</code></pre>

- The `CategoryInfo` field shows `[Write-Error]` instead of the function that actually generated the error, which was Get-Foo or the function that generated it which was Set-Foo!
- The `Exception` shows System.Exception, Set-Foo.  Compare that to a normal `Get-ChildItemException Foo` and we can see that PowerShell's internal commands are getting a different treatment than a module when it comes to a non-terminating error.

####_New-ModuleError:_

><pre><code>Set-Foo : Cannot set 'bar' because it does not exist.
At C:\Users\almostautomated\datastore\repos\git\ModuleWriteError\ModuleWriteError.demo.psm1:148 char:10
+         Set-Foo <<<<  bar
    + CategoryInfo          : ObjectNotFound: (bar:String) [Set-Foo], ModuleWriteError_demoModuleException
    + FullyQualifiedErrorId : FooObjectNotFound, Module : ModuleWriteError.demo\Get-Foo,Set-Foo

- Notice that now the `CategoryInfo` shows the function emitting the error, as well as the module name.
- The `FullyQualifiedErrorId` now also shows the correct information including the module and the function
using the same syntax as you'd use to call it (it even includes module prefix info when present) of both
the function that generated the function as well as the function emitting the error.

---

### Error-Record Exception Values
####_Write-Error:_

    Exception             : System.Exception: System.Management.Automation.ItemNotFoundException
    TargetObject          : bar
    CategoryInfo          : ObjectNotFound: (bar:String) [Write-Error], NormalWriteError
    FullyQualifiedErrorId : System.Exception,Set-Foo
    ErrorDetails          : Cannot set 'bar' because it does not exist.
    InvocationInfo        : System.Management.Automation.InvocationInfo
    PipelineIterationInfo : {0, 1}
    PSMessageDetails      :

####_New-ModuleError:_

    Exception             : ModuleWriteError_demoModuleException: Cannot set 'bar' because it does not exist.
                             ---> ModuleWriteError_demoModuleException: Cannot get 'bar' because it does not exist.
                               --- End of inner exception stack trace ---
    TargetObject          : bar
    CategoryInfo          : ObjectNotFound: (bar:String) [Set-Foo], ModuleWriteError_demoModuleException
    FullyQualifiedErrorId : FooObjectNotFound, Module : ModuleWriteError.demo\Get-Foo,Set-Foo
    ErrorDetails          :
    InvocationInfo        : System.Management.Automation.InvocationInfo
    PipelineIterationInfo : {0, 1}
    PSMessageDetails      :

- Lastly, take a look at the `Exception` of the two records, the New-ModuleError has a _(drum roll)_...
>####*A formatted stack trace!  Without needing to terminate our command flow!  Woohoo!*


Hope you find it useful.
Have fun!
