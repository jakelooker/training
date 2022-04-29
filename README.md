# userConfig Script

## Level 1 - The Main Process

*skip to line*

        if ((Test-Path -Path $statusfile) -eq $false){

Ignore the classes, variables and functions... This *main* part of the program is what it actually does.
It should be somewhat easy to read through it and understand the process.
The process is broken into sections using checks on the **status file**, sometimes seperated by reboots.

## Level 2 - Status File

*reference line 114*

*skip to line*

        if ((Test-Path -Path $statusfile) -eq $false){

On the initial run, the status file doesn't exist, so the first if statement runs.
When we get to the end of that if statement, we create the status file with the contents "0". Therefore the next if statement is true.
This pattern continues through each step of the status file, until finally we set it to "100", the final step and finish.

## Level 3 - Continuing after reboots

*reference function "Startup"*

In this function we copy the batch file launcher to the system wide startup folder.

        Copy-Item -Path $launcherBatch -Destination "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp\runme.bat" -ErrorAction Stop

But if we just ran the script each boot, it would do the same thing each time...

*reference line*

        Set-Content -Path $statusfile -Value "1"

At the end of this if statement, we create the status file and then optionally reboot. *Let's ignore the fact it's optional for the moment*.

*reference function "RebootComputer"*

We are now running the RebootComputer function. As part of this there is this line.

        Start-Sleep 400

This means the PC will initiate the reboot before the script is able to continue.
When the PC boots back up, it runs the batch file and starts the script.
It works it's way to the if statements around the status file and then picks up to the next section.

## Level 4 - Data/Variables

*reference line*

        $loginDomain = "tycoelectronics"

The first batch of variables defines where the local files will go on the machine. The working directory is defined first and then the rest of the files go inside it.

*reference line*

        $psexecEXE = "\\gb.tycoelectronics.com\netlogon\UKRSD-Scripts\userConfig\psexec.exe"

The next batch is items that live on the network that will get copied to the local PC.

*reference line*

        $dellUpdateCLI = "C:\Program Files (x86)\dell\CommandUpdate\dcu-cli.exe"

The location of the Dell Command Update Command Line tool. This could change if DCU gets updated/change as part of the base Dell Image.

*reference line*

        $global:packages = @()

Creating a variable in this way initialises it as a blank array (list). This means we create a list of many packages and store it inside one variable.

### General Notes

Package installer uses a CSV to store all package information.
The file copy process uses a json file for it's folder/registry information.

## Level 5 - Piping

Function or cmdlets can be added together with a pipe

        Get-Contents "file.txt" | Out-File "output.txt"

The output from the first cmdlet is passed as an input into the second cmdlet. This is used a lot with the functions of the script.

## Level 6 - Understanding the functions

### Common Themes

All functions use Try, Catch blocks.

        Try {That} 
        Catch {Do this if "That" went wrong, otherwise ignore it and carry on}

The catch section is the same on all functions, it's just to allow easier debugging and stop the script if something goes wrong.

        Catch {
            Write-Host "Error in EnableAutoLogon"
            Write-Host $_
            Write-Host $_.ScriptStackTrace
            Cleanup
            Pause
            Throw -1
        }

Firstly a simple message is shown to point to the failing function.
The $_ is an automatically created error variable that should include the full error message.
$_.ScriptStackTrace should shown the line number of the error.
The Cleanup function is run which will clear the auto-startup of the script and remove some of the txt files.
Pause to allow the message to be kept on screen.
Throw will exit the script with a failure code -1.

### Simple Function - **CleanUp**

It doesn't take any parameters and it doesn't return any data. You can just call the function on it's own.

### Intermediate Function **AddUserToAdminGroup**

It has a parameter block at the start -

        param (
                [Parameter(ValueFromPipeline)][pscredential]$cred
            )

**$cred** is the name we give to the variable that has been piped into the function, we use this to reference it inside the function.

"ValueFromPipeline" allows the function to read data that is piped into it. Meaning when we call it, we can feed data directly into it, instead of creating extra variables.

        RetrieveCredentials "customer" | AddUserToAdminGroup

Also note the parameter has a strict data type of [pscredential], if we try to send something different to it, it will error (and therefore trigger the Catch block). This reduces the need to check for mismatched data in the function. We can just assume it will be right type.

### Advanced Function **DellUpdater**

This function also has a parameter block that accepts data, although not from the pipeline this time. However it also returns data back.

        if ($operationSuccessful -eq $true){ 
            Start-Sleep 3
            return "Dell Command Update - $($message) - Success"
        } else { #If process failed, return this message
            Start-Sleep 3
            return "Dell Command Update - $($message) - Error Occurred"
        }

Therefore when we call this function, we should do something with the data it returns.

        DellUpdater -operation scheduleManual | SummaryCollector

So we call the function with the required parameter "operation". And then take the data it returns and pipe it into the "SummaryCollector" function.

## Level 7 - Saving Menu Options

### Practical Test

Run the script on a spare PC. Run through the first menu choosing options as you would for a normal build. Exit the script once all the menus are done. Check inside the C:\temp\userconfig\ folder for the various txt files that have been created and look at the contents.

### Storing Options

*reference line*

        ToggleMenu -menuInputs $mainMenuChoices -menuTitle "Main Menu - Which functions do you want to run?" -defaultState $true | StoreOptions -optionName "main"

*reference function "StoreOptions"*

StoreOptions accepts two inputs, one is the data to save, one is the name of the file to say it to.
So with the above example we create a toggle menu for the main menu items. Once the choices have been confirmed the items that are true get passed to StoreOptions. Each entry will be saved on a seperate line of main.txt

### Retrieving Options

*reference line*

        PackageInstaller (RetrieveOptions -optionName packages)

*reference function "RetrieveOptions"*

We have to pass the "optionName" parameter to RetrieveOptions. The function will then look for a .txt file with that name. If it finds it, it will read the data line by line and create an array of data. This can be passed on to another functions, for example PackageInstaller.

## Level 8 - Saving Credentials

### Practical Test

Take a look at the customer.txt and engineer.txt files. The username is in plain text but the password is encrypted.

### Storing

*reference line*

        GetCredentials "Engineer" | StoreCredentials -filename "engineer"

*reference function "GetCredentials", "StoreCredentials"*

First we call GetCredentials with "Engineer" as the $promptname. This function creates a prompt to type in credentials. It also has a second Try, Catch block that tests the login you have enterred by running a process. If the process cannot start, you will be sent in a loop to re-enter the details. Because this function uses the built-in "Get-Credentials" cmdlet, the password is stored as secure-string, not as plain text.

StoreCredentials breaks up the username and password into seperate entries and stores them into a txt file. The password has to be converted from a secure-string into an encrypted standard string. The limitation of this method is that decryption is only possible from the account that did the initial encryption.

## Retrieving

*reference line*

        RetrieveCredentials engineer | EnableAutoLogon

*reference function "RetrieveCredentials"*

We now retrieve the login information we need by referencing the text file we saved it to. The password will be changed back into a secure-string.

## Level 9 - What the hell is a Class?

[Microsoft Page on Classes](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_classes?view=powershell-7.2)

A Class is a custom data type. For example if we had a class called "Vehicle" we could include

* Number of wheels
* Fuel type
* Number of doors
* Make
* Model

*reference line*

        class ToggleItem {

So we have a collection of toggle items. Each item has a name and a toggle state of on/off. This is used to create a menu where you have the name of the item and true/false whether it's enabled or not.
Using a class means we can group the data together rather than using a seperate variable for each piece of data.

Then in a function like "ToggleMenu" we can import a list of items and turn them into ToggleItems.

        foreach ($item in $menuInputs){
            $MenuItems += [ToggleItem]::new($item,$defaultState)
        }

And then manipulate the state of the toggle on each item as required.

        if ($MenuItems[$Selection].toggle -eq $false){
            $MenuItems[$Selection].toggle = $true
        }
        elseif ($MenuItems[$Selection].toggle -eq $true){
            $MenuItems[$Selection].toggle = $false
        }
