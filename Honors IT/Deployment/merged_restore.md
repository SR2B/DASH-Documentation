

# Introduction

This script is included on every DeployStudio workflow.

## Usage
The restore script should be used at the end of DeployStudio workflows, or can be run by itself if a computer needs the correct LaunchAgents and other files for its restore class.

| Restore Name  | Manifest Name               | Student Logins? | Shared Computer? | Backed Up? |
|---------------|-----------------------------|-----------------|------------------|------------|
| Admin Restore | admin_ManagedInstalls.plist | no              | no               | yes        |
| Advisor Restore| advisor_ManagedInstalls.plist| no            |no                |yes         |
| Bonner Lab Restore | lab_ManagedInstalls.plist| yes           | yes              | no         |
|Classroom/Podium Restore|consulting_ManagedInstalls.plist|yes  |yes               |no          |
|Consulting Restore|consulting_ManagedInstalls.plist|yes        |yes               |no          |
| DASH Lab Restore | dashlab_ManagedInstalls.plist | yes        | yes              | no         |
|Faculty/Staff Restore|facultystaff_ManagedInstalls.plist|no    |no                |yes         |
| Lab Computers | lab_ManagedInstalls.plist   | yes             | yes              | no         |
|Student Worker Computer Restore|advisor_ManagedInstalls.plist  |no |yes           |no          |

### Manual Execution
```merged_restore.sh``` needs to be run as root and has 4 parameters:

```restore_type``` has the following possible values, which map to the different types of computers in the College:

* ```lab```
* ```bonnerlab```
* ```dashlab```
* ```facultystaff```
* ```admin```
* ```advisor```
* ```presentation```
* ```consulting```

```shared``` has two possible values and determines whether the keychain reset and removal LaunchAgent is installed.

* ```shared``` - Installs Keychain reset LaunchDaemon.
* ```notshared``` - Does nothing.

```student_login``` has two possible values, and determines whether the HC Students Active Directory group is allowed to login to the computer. Student workers are a part of HC Authenticated Users and don't need to use HC Students.

* ```student``` - Adds the HC Students group to the allowed list of users.
* ```nostudent``` - Only allows HC Admins and HC Authenticated Users to login.

```backup``` has two possible values, and determines whether the backup LaunchDaemon is installed. Computers that are shared should not be backed up. This includes SSO and Recruitment.

* ```backup``` - Installs the LaunchAgent.
* ```nobackup``` - Does nothing.


## The Script
The contents of this version of the script are below, with comments that explain the purpose of each function and the execution flow of the program.


### Set interpreter and various constants

````
#!/bin/sh

# start here
kickstart="/System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart"
systemsetup="/usr/sbin/systemsetup"
networksetup="/usr/sbin/networksetup"
defaults="/usr/bin/defaults"
airport="/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport"
hcstorage="http://hc-storage.cougarnet.uh.edu"

````
### Turn off AirPort
This makes sure that all network communications run through the Ethernet port. Wi-Fi interferes with Cougarnet access. It also required an administrator password to turn Wi-Fi on.

````
function turnOffAirport {
	$networksetup -detectnewhardware
	echo "Turning off airport..."
	$networksetup -setairportpower en1 off
	$airport en1 prefs RequireAdminPowerToggle=YES
	$systemsetup -setwakeonnetworkaccess on
}
````
### Turn on SSH
This allows remote access through the command line.

````
function turnOnSSH {
	echo "Turning on ssh..."
	$systemsetup -setremotelogin on
}
````
### Turn on Remote Desktop
This allows Remote Access through Apple Remote Desktop.

````
function turnOnRemoteDesktop {
	echo "Turning on RemoteManagement..."
	$kickstart -activate -configure -access -on -users hcadmin -privs -all -restart -agent
}
````
### Get ManagedInstalls.plist
This uses the first parameter to get the correct list of software for the computer.

````
function getManagedInstallsPlist {
	echo "Getting ManagedInstalls.plist..."
	if [ "$1" == "facultystaff" ]
	then
		/usr/bin/curl -s --show-error $hcstorage/managedinstalls/facultystaff_ManagedInstalls.plist -o "/Library/Preferences/ManagedInstalls.plist"
	elif [ "$1" == "presentation" ]
	then
		/usr/bin/curl -s --show-error $hcstorage/managedinstalls/consulting_ManagedInstalls.plist -o "/Library/Preferences/ManagedInstalls.plist"
	elif [ "$1" == "consulting" ]
	then
		/usr/bin/curl -s --show-error $hcstorage/managedinstalls/consulting_ManagedInstalls.plist -o "/Library/Preferences/ManagedInstalls.plist"
	elif [ "$1" == "advisor" ]
	then
		/usr/bin/curl -s --show-error $hcstorage/managedinstalls/advisor_ManagedInstalls.plist -o "/Library/Preferences/ManagedInstalls.plist"
	elif [ "$1" == "lab" ]
	then
		/usr/bin/curl -s --show-error $hcstorage/managedinstalls/lab_ManagedInstalls.plist -o "/Library/Preferences/ManagedInstalls.plist"
	elif [ "$1" == "dashlab" ]
	then
		/usr/bin/curl -s --show-error $hcstorage/managedinstalls/dashlab_ManagedInstalls.plist -o "/Library/Preferences/ManagedInstalls.plist"
	elif [ "$1" == "bonnerlab" ]
	then
		/usr/bin/curl -s --show-error $hcstorage/managedinstalls/bonnerlab_ManagedInstalls.plist -o "/Library/Preferences/ManagedInstalls.plist"
	elif [ "$1" == "admin" ]
	then
		/usr/bin/curl -s --show-error $hcstorage/managedinstalls/admin_ManagedInstalls.plist -o "/Library/Preferences/ManagedInstalls.plist"
	else
		echo "Can't load ManagedInstalls.plist..."
	fi
}
````
### Enable Guest Account

````
function enableGuestAccount {
	defaults write /Library/Preferences/com.apple.loginwindow GuestEnabled -bool YES
}
````

### Install PaperCut LaunchAgent
This installs a script that keeps PaperCut constantly open on the lab computers.

````
function getPaperCutLaunchAgent {
	echo "Getting PaperCut login script..."
	/usr/bin/curl -s --show-error $hcstorage/plists/edu.uh.honors.papercut.plist -o "/Library/LaunchAgents/edu.uh.honors.papercut.plist"
	/bin/chmod 644 /Library/LaunchAgents/edu.uh.honors.papercut.plist
}
````

### Setting Guest account to automatically login
````
function setAutomaticGuestLogin {
	echo "Setting guest to automatic login..."
	/usr/bin/defaults write /Library/Preferences/com.apple.loginwindow autoLoginUser guest
}
````

### Install Screen Lock LaunchAgent
This installs a script on shared computers to disable the screen lock.

````
function getScreenLockLaunchAgent {
	echo "Installing script to disable screen lock..."
	/usr/bin/curl -s --show-error $hcstorage/scripts/disable_screen_lock.sh -o "/usr/bin/disable_screen_lock.sh"
	/bin/chmod +x /usr/bin/disable_screen_lock.sh

	/usr/bin/curl -s --show-error $hcstorage/plists/edu.uh.honors.disablescreenlock.plist -o "/Library/LaunchAgents/edu.uh.honors.disablescreenlock.plist"
	/bin/chmod 644 /Library/LaunchAgents/edu.uh.honors.disablescreenlock.plist
}
````
### Install Keychain Reset LaunchDaemon
This installs a script that resets the keychain nightly on shared computers.

````
function getKeychainResetLaunchDaemon {
	echo "Getting Keychain Fix Script..."
	/usr/bin/curl -s --show-error $hcstorage/scripts/reset_keychains.sh -o "/usr/bin/reset_keychains.sh"
	/bin/chmod +x /usr/bin/reset_keychains.sh

	/usr/bin/curl -s --show-error $hcstorage/plists/edu.uh.honors.resetkeychains.plist -o "/Library/LaunchDaemons/edu.uh.honors.resetkeychains.plist"
	/bin/chmod 644 /Library/LaunchDaemons/edu.uh.honors.resetkeychains.plist
}
````
### Restrict the logins to certain Groups

````
function restrictActiveDirectoryLogins {
	echo "Restricting Active Directory logins..."
	/usr/bin/dscl . -create /Groups/com.apple.loginwindow.netaccounts
	/usr/bin/dscl . -create /Groups/com.apple.loginwindow.netaccounts Password "*"
	/usr/bin/dscl . -create /Groups/com.apple.loginwindow.netaccounts RealName "Login Window's custom net accounts"
	/usr/bin/dscl . -create /Groups/com.apple.loginwindow.netaccounts PrimaryGroupID 206

	/usr/sbin/dseditgroup -o edit -n /Local/Default -a HC\ Admins -t group com.apple.loginwindow.netaccounts
	/usr/sbin/dseditgroup -o edit -n /Local/Default -a HC\ Authenticated\ Users -t group com.apple.loginwindow.netaccounts

	if [ "$1" == "student" ]
	then
		echo "Adding students to allowed user list..."
		/usr/sbin/dseditgroup -o edit -n /Local/Default -a HC\ Students -t group com.apple.loginwindow.netaccounts
	fi

	/usr/bin/dscl . -create /Groups/com.apple.access_loginwindow
	/usr/bin/dscl . -create /Groups/com.apple.access_loginwindow Password "*"
	/usr/bin/dscl . -create /Groups/com.apple.access_loginwindow PrimaryGroupID 223
	/usr/bin/dscl . -create /Groups/com.apple.access_loginwindow RealName "Login Window ACL"

	/usr/sbin/dseditgroup -o edit -n /Local/Default -a localaccounts -t group com.apple.access_loginwindow
	/usr/sbin/dseditgroup -o edit -n /Local/Default -a com.apple.loginwindow.netaccounts -t group com.apple.access_loginwindow
}
````
### Install Backup LaunchDaemon
This installs the backup script and schedules regular backups when run with the ```backup``` parameter.

````
function getBackupLaunchDaemon {
	echo "Getting Backup Script..."
	/usr/bin/curl -s --show-error $hcstorage/scripts/backup.sh -o "/usr/bin/backup.sh"
	/bin/chmod +x /usr/bin/backup.sh

	/usr/bin/curl -s --show-error $hcstorage/plists/edu.uh.honors.backup.plist -o "/Library/LaunchDaemons/edu.uh.honors.backup.plist"
	/bin/chmod 644 /Library/LaunchDaemons/edu.uh.honors.backup.plist
}
````

### Install Office Setup LaunchAgent
This installs a script that suppresses the Office Setup screen when a user logs into a computer.

````
function getOfficeSetupLaunchAgent {
	echo "Getting Office setup script..."
	/usr/bin/curl -s --show-error $hcstorage/scripts/curl_office_plists.sh -o "/usr/bin/curl_office_plists.sh"
	/bin/chmod +x /usr/bin/curl_office_plists.sh

	echo "Getting Office preferences login script..."
	/usr/bin/curl -s --show-error $hcstorage/plists/edu.uh.honors.curlofficeprefs.plist -o "/Library/LaunchAgents/edu.uh.honors.curlofficeprefs.plist"
	/bin/chmod 644 /Library/LaunchAgents/edu.uh.honors.curlofficeprefs.plist
}
````
### Disable System Sleep
````
function disableSystemSleep {
	echo "Disabling system sleep..."
	/usr/bin/pmset sleep 0
}
````

### Disable save Window state at logout
````
function disableSaveWindowState {
	echo "Disable the save window state at logout..."
	/usr/bin/defaults write com.apple.loginwindow 'TALLogoutSavesState' -bool false
}
````

### Disable Automatic Software Updates
````
function disableAutomaticSoftwareUpdates {
	echo "Disabling Software Update Automatic Checks"
	softwareupdate --schedule off
}
````
### Disable Gatekeeper
````
function disableGatekeeper {
	echo "Disabling Gatekeeper..."
	spctl --master-disable
}
````

### Show username & password fields in Login Window
````
function enableUsernameAndPasswordFields {
	echo "Enabling username and password fields..."
	/usr/bin/defaults write /Library/Preferences/com.apple.loginwindow SHOWFULLNAME -bool TRUE
}
````

### Munki in Bootstrap mode
````
function bootstrapMunki {
	echo "Setting munki to bootstrap mode..."
	touch /Users/Shared/.com.googlecode.munki.checkandinstallatstartup
}
````

### Run actions
````
echo "Running firstboot script..."
turnOffAirport
turnOnSSH
turnOnRemoteDesktop
getManagedInstallsPlist $1
enableGuestAccount

#Enable persistent PaperCut on lab computers
if [ "$1" == "lab" ]
then
	getPaperCutLaunchAgent
fi

#Automatic guest login on classroom and podium computers
if [ "$1" == "presentation" ]
then
	setAutomaticGuestLogin
fi

#If computer is shared, we want keychains to reset, and to not lock the screen
if [ "$2" == "shared" ]
then
	getScreenLockLaunchAgent
	getKeychainResetLaunchDaemon
fi

#If students are logging in, this can't be an employee computer. Therefore, if not a student, run all other actions.
if [ "$3" == "student" ]
then
	restrictActiveDirectoryLogins student
else
	restrictActiveDirectoryLogins nostudent
	#Converted to mobileconfig and deployed through munki
	#getNetworkMountLaunchAgent
fi

if [ "$4" = "backup" ]
then
	getBackupLaunchDaemon
fi
````
### Run actions common to all systems
````
getOfficeSetupLaunchAgent
disableSystemSleep
disableSaveWindowState
disableAutomaticSoftwareUpdates
disableGatekeeper
enableUsernameAndPasswordFields
bootstrapMunki

echo "Done."

sleep 15

exit 0
```
