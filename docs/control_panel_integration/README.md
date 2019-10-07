# Control panel integration (Draft)

:::tip Note
We encourage you to create a pull request with your feedback at the bottom of the page.
:::

## General

To integrate CloudLinux into a control panel, you should implement the following interfaces required by the CloudLinux OS utilities:
* [CPAPI](/control_panel_integration/#control-panel-api-integration) — a small scripts/files-type interface to get the required information from the control panel
* [CP Hooks Integration](/control_panel_integration/#control-panel-hooks-integration) — a set of the CloudLinux OS scripts/commands. Control panel calls this set in response to its internal events
* [Web UI Integration(optional)](/control_panel_integration/#web-ui-integration) — various configuration parameters and scripts to embed interface SPA application into the control panel native interface. Optional, if CLI is enough
* [CageFS Integration](/control_panel_integration/#how-to-integrate-cagefs-with-a-control-panel) — some changes may be required in the config files to work and integrate with CageFS as well as additional support from a control panel

Configuration related to integration should be placed in a valid INI file (further “integration config file”):

```
-rw-r--r-- root root /opt/cpvendor/etc/integration.ini
```
The panel vendor should deliver/update this file. All users (and in the CageFS too) should have read access to this file as some CPAPI methods have to work from end-users too.

We recommend to deliver a set of scripts for CPAPI and integration file in a separate RPM package. This can greatly simplify support for vendors as they will be able to release any updates and compatibility fixes separately from panel updates and use dependencies/conflicts management provided by a package manager.

Assumed that this package contains the list of paths to the integration scripts (or to the one script if it deals with all `cpapi` commands).

#### **Example of the integration config**:

```
[integration_scripts]

get_cp_name = /usr/bin/cl_panel_script some-command-to-get-cpname --any-args
get_cp_description = /usr/bin/script_get_cp_description
db_access = /usr/bin/cl_panel_script_db_access


[lvemanager_config]
ui_user_info = /panel/path/ui_user_info.sh
base_path = /panel/path/lvemanager
base_uri = /lvemanager

```
## Control panel API integration

To integrate CPAPI, set the paths to the scripts in the integration file in the `integration_scripts` section (see example above.

You can use different scripts for different CPAPI methods or only one script and call it with different arguments. The script is any executable file written in any programming language, that returns a valid JSON file of the specified format in response to arguments it is expected to be called.

### Integration scripts requirements

* By default, integration scripts should run only from UNIX user with the same credentials as LVE Manager is run, unless otherwise stated in the script description.
* Some scripts are run from both root and end-user. If the script is run from end-user it should return information related to this user only, filtering the irrelevant data for security/isolation purposes or return `PermissionDenied` if script is not intended to be run as this user at all.
* Scripts run from the end-user should also work from inside CageFS. You can find details on how to implement this [here](/control_panel_integration/#working-with-cpapi-cagefs)

#### **Legend**:
* Necessity `always` means it’s required nearly by all CloudLinux code and should be implemented no matter what features you are interested in. Other methods might not be implemented and this will affect only some CloudLinux features leaving others to work fine.
* `All UNIX users` means any Linux system account (from `/etc/passwd`) that is able to execute CloudLinux utilities directly or indirectly.


| | | | |
|-|-|-|-|
|Script name|Necessity|Accessed by|Must work inside CageFS also|
|panel_info|Always|All UNIX users|+|
|db_info|Only for LVE-Stats, otherwise optional|admins (UNIX users)|-|
|packages|For limits functionality|admins (UNIX users)|-|
|users|Always|admins (UNIX users)|-|
|domains|Selectors, some UI features|All UNIX users|+|
|resellers|Always|admins (UNIX users)|-|
|admins|Always|admins (UNIX users)|-|

### Working with CPAPI & CageFS

Some integration scripts required to be called not only from root but from end-user too, so they should be able to work inside [CageFS](/cloudlinux_os_components/#cagefs) too. This means that all required information (a code, all its dependencies, all called binaries, all configs/files/caches it reads) should either:
* be available in CageFS
* the script itself should be run in the real system via [proxyexec](/cloudlinux_os_components/#executing-by-proxy) mechanism

You can find details on how to configure CageFS properly here:
* [Configuration. General information](/cloudlinux_os_components/#configuration-2)
* [How to integrate CageFS with any control panel](/control_panel_integration/#how-to-integrate-cagefs-with-a-control-panel)

### Expected structure of replies

A reply from any integration script should be a valid JSON and UTF-8 encoding is required.

There are two expected formats of integration script replies. In case of success, the metadata.result field is `ok` and the data field contains data according to the specification of the specific integration script. Return code is `0`.

```
{
  "data": object or array,
  "metadata": {
    "result": "ok"
  }
}
```

In case of error, the output should be one of the following.

#### Internal errors

In case when data is temporarily unavailable due to internal errors in the integration script (database unavailable, file access problems, etc), the return code is `1` and the output  is as follows:

```
{
  "data": null,
  "metadata": {
    "result": "InternalAPIError",
    "message": "API is restarting, try again later"
  }
}
```

**Data description**

| | | |
|-|-|-|
|Key|Nullable|Description|
|message|False|Error message for an administrator printed by CloudLinux utility|

#### Restricted access

In case when data is unavailable due to restricted access of a user that called the script, the return code is `2` and the output  is as follows: 

```
{
  "data": null,
  "metadata": {
    "result": "PermissionDenied",
    "message": "Only sudoers can view this section."
  }
}
```

**Data description**

| | | |
|-|-|-|
|Key|Nullable|Description|
|message|False|Error message for an administrator printed by CloudLinux utility|

#### Error in arguments

In case when known to be false arguments are passed to the utility, for example:
* unknown argument
* unknown fields in `--fields`

The return code is `3` and the output is as follows:

```
{
  "data": null,
  "metadata": {
    "result": "BadRequest",
    "message": "Unknown field ‘nosuchfield’"
  }
}
```

**Data description**

| | | |
|-|-|-|
|Key|Nullable|Description|
|message|False|Error message for an administrator printed by CloudLinux utility|


#### Nonexistent entities

In case when during data filtering the target entity doesn't exist in the control panel, the return code is `4` and the output is as follows:

```
{
  "data": null,
  "metadata": {
    "result": "NotFound",
    "message": "No such user `unix_user_name`"
  }
}
```

**Data description**

| | | |
|-|-|-|
|Key|Nullable|Description|
|message|False|Error message for an administrator printed by CloudLinux utility|

### The list of the integration scripts

#### panel_info

Returns the information about the control panel in the format specified.

**Usage example**

```
/scripts/panel_info
```

**Output example**

```
{
	"data": {
		"name": "CentosWebPanel",
		"version": "1.0.1",
		"user_login_url": "https://{domain}:1111/"
	},
	"metadata": {
		"result": "ok"
	}
}
```

**Data description**

| | | |
|-|-|-|
|Key|Nullable|Description|
|name|False|Control panel name|
|version|False|Control panel version|
|user_login_url_template|True|URL template for a user entering to control panel. Used in the lve-stats default templates reporting that user is exceeding server load. You can use the following placeholders in the template: `{domain}`. CloudLinux utility automatically replaces  placeholders to the real values. **Example**:`“user_login_url_template”: “https://{domain}:2087/login”` CloudLinux utility automatically replaces `{domain}`, and the link will look like `https://domain.zone:2087/login`|


#### db_info

Returns the information about databases that are available to the control panel users and are managed by the control panel.

:::warning WARNING!
For now, CloudLinux supports control panels with only one MySQL database.
:::

Integration script is optional, when there is no script, [lve-stats won’t collect SQL snapshots](cloudlinux_os_components/#plugins).

**Usage example**

```
/scripts/db_info
```

**Output**

```
{
  "data": {
    "mysql": {
      "access": {
        "login": "root",
        "password": "Cphxo6z",
        "host": "localhost",
        "port": 3306
      },
      "mapping": {
        "unix_user_name": [
          "db_user_name1",
          "db_user_name2"
        ],
        "unix_user_name2": [
          "db_user_name3",
          "db_user_name4"
        ]
      }
    }
  },
  "metadata": {
    "result": "ok"
  }
}
```

**Data description**

| | | |
|-|-|-|
|Key|Nullable|Description|
|mysql|True|Data section for the MySQL database. Used in lve-stats; if there is no data, snapshots of  SQL queries won’t be collected.|
|mysql.access.login|False|User name to enter to the database to view a list of SQL queries that are executing at the moment.|
|mysql.access.password|False|Open password.|
|mysql.access.host|False|Host to access the database.|
|mysql.access.port|False|Port to access the database.|
|mysql.mapping|False|A mapping file where the key is the name of a UNIX user managed by the control panel and the value is a list of the corresponding database users.|


#### packages

The package is an abstraction that represents a group of users that have the same default limits. 

**Usage example**

```
/scripts/packages [--owner=<owner>]
```

**Arguments**

| | | |
|-|-|-|
|Key|Required|Description|
|--owner, -o|False|Set the name of the packages' owner to output.|

**Output example**

```
{
  "data": [
    {
      "name": "package",
      "owner": "root"
    },
    {
      "name": "package",
      "owner": "admin"
    },
    {
      "name": "package",
      "owner": "reseller"
    }  
  ],
  "metadata": {
    "result": "ok"
  }
}
```

**Data description**
| | | |
|-|-|-|
|Key|Nullable|Description|
|name|False|Package name|
|owner|False|Package’s owner name. The owner can be the administrator or reseller.|

#### users

Returns information about UNIX users, created and managed by the control panel. You will be able to manage the limits of these users via LVE Manager.

If a reseller or administrator except for the account in the control panel has UNIX-user in the system, it should be included in this list.

**Usage example**

```
/scripts/users [--owner=<string> | (--package-name=<string> & --package-owner=<string>) | --username=<string> | --unix-id=<int>] [--fields=id,username,package,...]
```

**Arguments**

| | | |
|-|-|-|
|Key|Required|Description|
|--owner, -o|False|Used for filtering. When the argument is set, output only this owner packages.|
|--package-name|False|Used for filtering. Set the name of the package which users to output. Used ONLY in pair with `--package-owner`.|
|--package-owner|False|Used in pair with package.name. Set the owner of the specified package.|
|--username|False|Used for filtering. When the parameter is set, output the information about this UNIX user only.|
|--unix-id|False|Used for filtering. When the parameter is set, output the information about this UNIX user only.|
|--fields|False|List fields to output. Fields are divided with a comma; when there is no key, output all available fields.|

The key is used to reduce the data size for the large systems. You can ignore the key and always output the full set of data. Note that in this case, CloudLinux will work more slowly.

**Output example**

```
{
  "data": [
    {
      "id": 1000,
      "username": "ins5yo3",
      "owner": "root",
      "domain": "ins5yo3.com",
      "package": {
        "name": "package", 
        "owner": "root"
      },
      "email": "ins5yo3@ins5yo3.com",
      "locale_code": "EN_us"
    },
    {
      "id": 1001,
      "username": "ins5yo4",
      "owner": "root",
      "domain": "ins5yo4.com",
      "package": {
        "name": "package", 
        "owner": "root"
      },
      "email": "ins5yo4@ins5yo4.com",
      "locale_code": "EN_us"
    }  
  ],
  "metadata": {
    "result": "ok"
  }
}
```

**Example to output specific data fields**

```
/scripts/users --filter=id,email
```

**Output**

```
{
  "data": [
    {
      "id": 1000,
      "email": "ins5yo3@ins5yo3.com"
    },
    {
      "id": 1001,
      "email": "ins5yo3@ins5yo3.com"
    }  
  ],
  "metadata": {
    "result": "ok"
  }
}
```

**Data description**

| | | |
|-|-|-|
|Key|Nullable|Description|
|id|False|ID of the UNIX account in the system.|
|username|False|The name of the UNIX account in the system.|
|owner|False|The name of the account owner in the control panel. The owner can be an administrator (in this case he should be included in the `admins()` output) or a reseller (in this case he should be included in the `resellers()` output).|
|locale_code|True|The control panel locale selected by a user (the values are according to ISO 639-1). Used when sending lve-stats emails. If not selected, EN_us used.|
|email|True|Email to send lve-stats notifications about hitting limits. If there is no email, should return null.|
|domain|False|Main domain of user (used to display in lvemanager UI and create applications in selectors)|
|package|True|Information about the package to which a user belongs to. If the user doesn’t belong to any package, should return null.|
|package.name|False|The name of the package to which a user belongs to.|
|package.owner|False|The owner of the package to which a user belongs to (reseller or administrator).|


#### domains

Returns key-value object, where a key is a domain (or subdomain) and a value is a key-value object contains the owner name (UNIX users), the path to the site root specified in the HTTP server config, and the domain status (main or alternative).

::: warning WARNING
To make Python/Node.js/Ruby/PHP Selector workable, this script should be executed with user access and inside CageFS.
:::

**Usage example**

```
/scripts/domains ([--owner=<unix_user>] | [--name=<name>])
```

**Arguments**

| | | |
|-|-|-|
|Key|Required|Description|
|--owner, -o|False|Used for filtering output; set a name of the owner whose domains to print|
|--name, -n|False|Used for filtering output; set a name of the domain, to display information about it|

**Output example**

```
{
  "data": {
    "domain.com": {
      "owner": "username",
      "document_root": "/home/username/public_html/",
      "is_main": true
    },
    "subdomain.domain.com": {
      "owner": "username",
      "document_root": "/home/username/public_html/subdomain/",
      "is_main": false  
    }
  },
  "metadata": {
    "result": "ok"
  }
}
```

**Data description**

| | | |
|-|-|-|
|Key|Nullable|Description|
|name|False|Domain name (subdomains are acceptable)|
|owner|False|UNIX user who is a domain owner|
|document_root|False|Absolute path to the site root directory|
|is_main|False|Is the domain the main domain for a user|

#### resellers

Gives information about resellers who can be users owners in the control panel. Resellers do not obligatory have their own same-name UNIX accounts in the system and could exist only as an account in the control panel.

**Usage example**

```
/scripts/resellers ([--id=<id>] | [--name=<name>])
```

**Arguments**

| | | |
|-|-|-|
|Key|Required|Description|
|--id|False|Used for filtering output; set a reseller ID to output information about it|
|--name, -n|False|Used for filtering output; set a reseller name to output information about it|

**Output example**

```
{
  "data": [
    {
      "name": "reseller",
      "locale_code": "EN_us",
      "email": "reseller@domain.zone",
      "id": 10001   
    }
  ],
  "metadata": {
    "result": "ok"
  }
}
```

**Data description**

| | | |
|-|-|-|
|Key|Nullable|Description|
|name|False|Reseller name in the control panel; unique on the server, should not be the same as administrators names|
|locale_code|True|Control panel locale selected by a reseller (according to ISO 639-1). Default is EN_us.|
|id|True|Unique reseller ID in the system. From `MAX_UID` to `0xFFFFFFFF`, where `MAX_UID` is a value from `/etc/login.defs`. Required for the second level reseller limits. Set null if not needed.|
|email|True|Reseller email to send notifications from lve-stats about reseller’s users limits hitting. If no email, output null.|


#### admins

Gives information about panel’s administrators, output information about all panel’s administrators who:
* could be (or actually are) the owners of the users, listed in users() 
* could be (or actually are) the owners of the packages, listed in packages()
* has UNIX users with the rights to run LVE Manager UI

**Usage example**

```
/scripts/admins ([--name=<string>] | [--is-main=<true|false>])
```

**Arguments**

| | | |
|-|-|-|
|Key|Required|Description|
|--name, -n|False|Used for filtering; set the administrator name to output information about it|
|--is-main, -m|False|Used for filtering output to get the main and the alternative administrators of the control panel|

**Output example**

```
{
  "data": [
    {
      "name": "admin1",
      "unix_user": "admin",
      "locale_code": "EN_us",
      "email": "admin1@domain.zone",
      "is_main": true
    },
	{
      "name": "admin2",
      "unix_user": "admin",
      "locale_code": "Ru_ru",
      "email": "admin2@domain.zone",
      "is_main": false
    },
  ],
  "metadata": {
    "result": "ok"
  }
}
```

**Data description**

| | | |
|-|-|-|
|Key|Nullable|Description|
|name|False|The name of the administrator in the control panel; unique, should not be the same as resellers’ names|
|unix_user|True|The name of the UNIX account in the system; it corresponds to the administrator account in the control panel. If an administrator already has UNIX account, it is automatically added to the special group on the server using hook, so this administrator can enter to the LVE Manager and manage users limits. If an administrator doesn't have UNIX account and he can only own users, output null.|
|locale_code|True|Control panel locale selected by the administrator (values are according to  ISO 639-1). Default is EN_us.|
|email|True|Email to send notifications from lve-stats. If email is not set in the control panel, output null.|
|is_main|False|Assumed that one of the control panel’s administrators is the main administrator. That particular administrator will receive server status notifications from lve-stats to his email.|


## Control panel hooks integration

For CloudLinux proper operation, you should implement hooks mechanism. These hooks are called with the specified arguments in the proper time. All scripts should be executed by the control panel from the system root user.

### Managing administrators

_Some control panels allow to create several administrator accounts, each of which has a system user who is used to work with the LVE Manager plugin. In this case, all administrator accounts should be added to the sudoers and clsupergid. To do so, you can use our user-friendly interface._

::: tip
If you don’t have several administrator accounts (or LVE Manager plugin always runs from the root) you don’t need to implement these hooks.
:::

After creating a new administrator, the control panel should call the following command:
```
/usr/share/cloudlinux/hooks/post_modify_admin.py create --username admin
```

| | | |
|-|-|-|
|Argument |Default |Description |
|--name, -n | - |The name of the newly created administrator account |

Before removing the administrator, the following command should be called by the control panel:
```
/usr/share/cloudlinux/hooks/pre_modify_admin.py delete --name admin
```

| | | |
|-|-|-|
|Argument |Default |Description |
|--name, -n | - |The name of administrator account to be removed |

### Managing users

To manage user limits properly, CloudLinux utilities need information about the following events occurred in the control panel:

After creating a new user, the following script should be called:
```
/usr/share/cloudlinux/hooks/post_modify_user.py create --username user --owner owner
```

| | | | |
|-|-|-|-|
|Argument |Mandatory |Default |Description |
|--username, -u |Yes | - |The name of administrator account to be removed |
|--owner, -o |Yes | - |A name of the account owner. The owner can be an administrator or a reseller |

After renaming a user (when a user name or a home directory was changed), the following script should be called:
```
/usr/share/cloudlinux/hooks/post_modify_user.py rename --old-username user --new-username newuser
```

| | | | |
|-|-|-|-|
|Argument |Mandatory |Default |Description |
|--old-username |Yes | - |The user name to be changed |
|--new-username |Yes | - |A new user name |

After moving the user to the new owner, the following command should be run:
```
/usr/share/cloudlinux/hooks/post_modify_user.py transit --new-owner new_owner
```

| | | |
|-|-|-|
|Argument |Default |Description |
|--new-owner | - |The name of the new account owner. The owner can be an administrator or a reseller. |
|--username, -u | - |The account name (should be the same as the name in /etc/passwd). |

Before removing a user, the following command should be run:
```
/usr/share/cloudlinux/hooks/pre_modify_user.py delete --username user
```

| | | |
|-|-|-|
|Argument |Default |Description |
|--username, -u | - |The name of the account to be removed (should be the same as the name in /etc/passwd). |

After removing the user, you should call the following command:
```
/usr/share/cloudlinux/hooks/post_modify_user.py delete --username user
```

| | | |
|-|-|-|
|Argument |Default |Description |
|--username, -u | - |The name of the removed account |

### Managing domains

After renaming a domain (or any equivalent domain removal operation with transfer of all data to another domain), the following command should be run:
```
/usr/share/cloudlinux/hooks/post_modify_domain.py rename -u user --old-domain old_domain --new-domain new_domain [--include-subdomains]
```

| | | | |
|-|-|-|-|
|Argument |Mandatory |Default |Description |
|--username, -u |Yes | - |The name of the new domain owner (should be the same as the name in /etc/passwd). |
|--old-domain |Yes | - |The domain name to be changed |
|--new-domain |Yes | - |A new domain name |
|--include-subdomains |No |False |If set, all subdomains are renamed as well, i.e. when renaming domain.com → domain.eu the corresponding subdomain will be renamed as well test.domain.com → test.domain.eu.|

## Web UI integration

### ui-user-info script

This script should return information about the current user opening Web UI
Used only in UI part of the LVE Manager and utilities.

* Input value (first positional argument) — the current authentication token, passed on plugin open (for example, <span class="notranslate">`open.php?token=hash`</span>)
* Expected data format

```
{
    "userName": "user1",
    "userId": 1000,
    "userType": "user",
    "lang": "en",
    "userDomain": "current-user-domain.com"
}
```

* **userName** is used in queries to Selectors.
* **userId** - system user ID if exist. May be used in future functions.
* **userType** - can have the following values: “user”, “reseller”, “admin”
* **lang** - used to identify a current locale in the user interface (‘en’, ‘ru’ - two-character language code)
* **userDomain** - a current user domain. It can be the default domain or any selected user domain. Used to display in Selector. For example, in DirectAdmin the control panel opens only if a domain is selected — the selected domain is set. In cPanel, a domain is not selected — the default domain is set. If a domain is absent —  leave empty.

The following configuration file parameters are used to determine the location of the plugin UI part.

* **base_path** - the path to copy file assets to make them available from the control panel. Optional if `/usr/share/l.v.e-manager/commons` and `/usr/share/l.v.e-manager/panelless-version/lvemanager` are available from the control panel and the paths to this directory in web server are set.
Files are copied or replaced by <span class="notranslate">`yum update lvemanager`</span> command.

* **base_uri** - the URI of LVE Manager files

## PHP-based integration of WEB UI with the control panel

The first use case:
```
require_once('lvemanager/widget.php');
```

This script defines the type of user and displays icons to open plugins included in LVE Manager.
You can customize your styles by copying the file content and setting your control panel styles.
In this case, all LVE Manager plugins will be opened in a new window.

The second use case:

Embedding the specific plugin into the control panel page.
```
<?php
require_once('lvemanager/LveManager.php');
$manager = new LveManager(‘<pluginName>’);
$manager->setCSRFToken();
?>
<html><body>
<?php
echo $manager->insertPanel();
?>
</body></html>
```

Pass the plugin name to the class constructor to initialize the plugin (user name will be defined).
At the time of the insertPanel method invocation, the SPA application with the specified Selector will be embedded.

Queries to the backend are created separately in the points (PHP files) which are located in the LVE Manager catalog.

### UI with CageFS enabled

LVE Manager scripts are run outside CageFS (on production systems). When calling API, LVE Manager scripts enter into CageFS if needed (Node.js Selector, Python Selector). PHP Selector API does not work if the script is started inside CageFS and returns an error.

In case of Node.js Selector and Python Selector, this is due to the possibility of starting scripts from the UI (npm run, pip install). They are run inside LVE.

In case of PHP Selector, there is not possible. When PHP Selector is called, the resources for such queries are not limited and when the user’s site is reaching limits, the UI will work without enforcing limits.

## How to integrate CageFS with a control panel

CageFS documentation can be found here: [CageFS](/cloudlinux_os_components/#cagefs)

### CageFS MIN_UID  

CageFS has MIN_UID setting. Users with UIDs < MIN_UID will not be limited by CageFS. This setting is configured based on UID_MIN setting from `/etc/login.defs` file by default. So, typically MIN_UID is 500 on CloudLinux 6 and 1000 on CloudLinux 7. You can obtain current MIN_UID value by executing
```
cagefsctl --get-min-uid`
```
You can set MIN_UID by executing:
```
cagefsctl --set-min-uid <value>
```
For example, to set MIN_UID to 10000, you should execute
```
cagefsctl --set-min-uid 10000
```

### PAM configuration for CageFS

CageFS is enabled for su, ssh and cron services by default.
[PAM Configuration](/cloudlinux_os_components/#pam-configuration)

Also you can enable CageFS for any service that uses PAM.
{LVE PAM Module](/cloudlinux_os_components/#lve-pam-module)

:::tip
It is safe to enable an interactive shell (e.g. /bin/bash) for users when CageFS is enabled.
:::

### Integrating CageFS with Apache

You should apply CloudLinux patches to integrate CageFS with Apache. Details can be found here:
[Integration of Apache modules with Control Panels](/control_panel_integration/#integration-of-apache-modules-with-control-panels)

Note that it may be required to execute “cagefsctl --force-update” after rebuild of Apache in order to update CageFS.

### Running commands inside CageFS

You may want to execute commands inside CageFS for security reasons. To do so, you can execute a command inside CageFS in the following way:
```
/bin/su -s /bin/bash -c “$COMMAND” - $USER
/sbin/cagefs_enter_user $USER "$COMMAND"
```

The commands above require root privileges. You can use the following command when running as user:
```
/bin/cagefs_enter "$COMMAND"
```

### Updating CageFS skeleton

Updating CageFS skeleton is required after update of system RPM packages. It may be also required after update of hosting control panel itself. Execute
```
cagefsctl --force-update
```

in order to update CageFS skeleton.
CageFS RPM package installs a daily cron job that updates CageFS skeleton
```
/etc/cron.d/cp-cagefs-cron
```
You can set update period for CageFS skeleton in days using
```
cagefsctl --set-update-period <days>
```

### Setting up directories for PHP sessions

Each user in CageFS has its own `/tmp` directory that is not visible for other users. PHP scripts save sessions in `/tmp` directory by default (when `session.save_path` directive is empty). However, you can set up a different location for PHP sessions specific to each PHP versions using CageFS per-user [mounts](/cloudlinux_os_components/#mount-points) like it is done for alt-php. Simply add a line like below to `/etc/cagefs/cagefs.mp`
```
@/opt/alt/php54/var/lib/php/session,700
```

and then execute
```
cagefsctl --remount-all
```

to apply changes. After that, session files will be stored in `/opt/alt/php54/var/lib/php/session` inside user’s CageFS. In real file systems these files can be found inside `.cagefs` directory in user’s home directory. For above example, given that user’s home directory is `/home/user`, the files will reside in `/home/user/.cagefs/opt/alt/php54/var/lib/php/session` directory. TMP directory for the user will be located in `/home/user/.cagefs/tmp`. 
Temporary files including php sessions in `/tmp` directories in CageFS are cleaned using `tmpwatch`. You can change the period of removing old temporary files or set up additional directories for cleaning according to the documentation: 
[TMP directories](/cloudlinux_os_components/#tmp-directories).

Knowing the location where PHP sessions are stored (described above), you can also implement any custom script for cleaning PHP sessions. Remember to drop permissions (switch to the appropriate user) when removing the files (for example using sudo).

### Creating new user account

When user or admin account has been created, you should execute an appropriate hook script, so all needed actions are performed.
More about CloudLinux hooks subsystem: [Control Panel Hooks Integration](/control_panel_integration/#control-panel-hooks-integration)

:::tip Note
It is safe to enable an interactive shell (/bin/bash) for users when CageFS is enabled.
:::

:::warning IMPORTANT
Execute the post hook described in the link above after the account has been created completely. When the account is being transferred to another server or restored from backup, all files in user’s home directory should be written/restored completely BEFORE executing the post hooks, including files in `~/.cl.selector` directory. This is required to setup/restore PHP Selector settings for the user correctly. If PHP Selector settings were not restored correctly, you can recreate them by executing

```
rm -rf /var/cagefs/`/usr/sbin/cagefsctl --getprefix $username`/$username/etc/cl.selector
rm -rf /var/cagefs/`/usr/sbin/cagefsctl --getprefix $username`/$username/etc/cl.php.d
/usr/sbin/cagefsctl --force-update-etc "$username"
```
:::

:::warning IMPORTANT
You should create `/etc/cagefs/enable.duplicate.uids` empty file when your control panel creates users with duplicate UIDs. This is required for CageFS to operate properly.
:::

### Modifying user accounts

You should execute appropriate hook script when an account has been modified: [Control Panel Hooks Integration](/control_panel_integration/#control-panel-hooks-integration)

### Removing user accounts

CageFS has userdel hook script `/usr/bin/userdel.cagefs`, that is configured in `/etc/login.defs` file:
```
USERDEL_CMD /usr/bin/userdel.cagefs
```

This script performs all needed actions when a user is being removed.
Anyway, you should execute appropriate hook script when removing an account:
[Control Panel Hooks Integration](/control_panel_integration/#control-panel-hooks-integration)

### Excluding users from CageFS

If you need to exclude some system users from CageFS that are specific to your control panel, you can do this by creating a file (any name would work) inside `/etc/cagefs/exclude` folder, and list users that you would like to exclude from CageFS in that file (each user in separate line). Then execute
```
cagefsctl --user-status USER
```

to apply changes and check that the command shows _Disabled_:
[Excluding users](/cloudlinux_os_components/#excluding-users)

### How to add a file or a directory to CageFS

There are two major ways to add files and directories to CageFS:

1. copy files and directories to CageFS
2. mount an entire directory with needed files to CageFS


#### Copy files and directories to CageFS

In order to copy files and directories to CageFS, you can manually create a custom config file in the <span class="notranslate">`/etc/cagefs/conf.d`</span> directory.

Also, you can add files and directories that belong to `rpm` package to CageFS as follows:

* execute <span class="notranslate">`cagefsctl --addrpm PACKAGE`</span> command
* execute <span class="notranslate">`cagefsctl --force-update`</span> command to apply changes, i.e. to copy files and directories to the <span class="notranslate">`/usr/share/cagefs-skeleton`</span> directory.

There is a daily cron job that copies new and removes unneeded files and directories, i.e. updates cagefs-skeleton.

You should execute <span class="notranslate">`cagefsctl --force-update`</span> after each change of system configuration or after installing new software. Otherwise, CageFS will be updated for 24 hours by a daily cron job.

The benefits of this approach:
* users in CageFS cannot modify files in the real file system (for example, in case of wrong permissions), because files in cagefs-skeleton are the copies of files from the real file system (and not mounted to CageFS)
* you can copy specific _safe_ files from some directory from the real file system to CageFS, and not copy all files and not mount the entire directory.

You can find more info [here](/cloudlinux_os_components/#file-system-templates).

#### Mount an entire directory with needed files to CageFS

The second way to add files and directories to CageFS is to mount the entire directory with needed files to CageFS.

This approach should be used for frequently updated files (like UNIX sockets) or for directories with a large number of files or with a big size of data.

The benefits of this approach:
* no delay between update of the file in the real file system and update of the file in CageFS; when file or directory is changed in the real file system, then they will be changed in CageFS immediately
* data is mounted but not copied to CageFS, so this minimizes IO load while updating cagefs-skeleton

Please make sure that the mounted directory does not contain sensitive data.

You can mount a directory from the real system to CageFS via <span class="notranslate">`/etc/cagefs/cagefs.mp`</span> config file.

To apply changes of mount points in /etc/cagefs/cagefs.mp file you should execute the following command:

<div class="notranslate">

```
cagefsctl --remount-all
```
</div>

You can find more info [here](/cloudlinux_os_components/#mount-points).

### Users' home directory

CageFS mounts users' home directories to CageFS automatically. Usually, there is no need to configure anything. However, if your control panel uses a custom path to users' home directories (for example `/home/$USER/data` instead of `/home/$USER`), then it may be necessary to configure Base Home Directory setting:
[Base Home Directory](/cloudlinux_os_components/#base-home-directory)

The modes of mounting users' home directories into CageFS are described here: 
[Mounting user’s home directory inside CageFS](/cloudlinux_os_components/#mounting-users-home-directory-inside-cagefs)

### Excluding files from CageFS

CageFS has a default set of files and directories that are visible to users inside CageFS. If you wish to exclude some of these files or directories from CageFS, you can do this like described below:
[Excluding files](/cloudlinux_os_components/#excluding-files) 

### Executing commands outside CageFS via proxyexec

SUID programs cannot run inside CageFS due to “nosuid” mounts. But you can give users in CageFS the ability to execute some specific commands outside CageFS via `proxyexec`.

Typically, these commands are suid programs or other programs that cannot run inside CageFS (due to complex dependencies, for example). Examples are <span class="notranslate">`passwd`</span>, <span class="notranslate">`exim`</span> or <span class="notranslate">`sendmail`</span> commands.

* Regular (not suid) commands will be executed as user inside the user’s LVE, but outside CageFS.
* SUID commands will be executed in the user’s LVE outside CageFS with the effective UID set to the owner of the file being executed.

You can find more details [here](/cloudlinux_os_components/#executing-by-proxy).

:::tip Note
You should use this feature with caution because it gives users the ability to execute specified commands outside CageFS. SUID commands are extremely dangerous because they are executed not as a user, but as an owner of the file (typically root). You should give users the ability to execute safe commands only. These commands should not have known vulnerabilities.
You should check that users cannot use these commands to get sensitive information on a server. You can disable specific _dangerous_ options of programs executed via proxyexec (see below).
:::


### Filtering options for commands executed by proxyexec

It is possible to disable specific *dangerous* options of programs executed via `proxyexec`. More details:
[Filtering options for commands executed by proxyexec](/cloudlinux_os_components/#filtering-options-for-commands-executed-by-proxyexec).

## Integrating CloudLinux PHP Selector

Complete documentation for CloudLinux PHP Selector can be found here:
[PHP Selector](/cloudlinux_os_components/#php-selector)

:::tip Note
CloudLinux PHP Selector requires CageFS, so integration of CageFS must be completed before starting PHP Selector integration.
:::

If your control panel has some PHP version selector installed already, we suggest you not to enable CloudLinux PHP Selector for your users, so they will not be confused by two multiple PHP Selectors. However, if you want to use CloudLinux PHP Selector anyway, you can refer to the following article to configure loading of extensions for alt-php: [PHP extensions](/cloudlinux_os_components/#php-extensions)

For proper operation of CloudLinux PHP Selector, you should specify location of native PHP binaries of your control panel in `/etc/cl.selector/native.conf`. Then execute
```
cagefsctl --setup-cl-selector
```

to apply changes.
More details: [Native PHP Configuration](/cloudlinux_os_components/#native-php-configuration)

You can configure CloudLinux PHP Selector to allow your customers an ability to select any custom PHP version specific to your control panel. Here's how to do this:
[Roll your own PHP](/cloudlinux_os_components/#roll-your-own-php)

Also, you can compile and add your own PHP extensions to CloudLinux PHP Selector:
[Compiling your own extensions](/cloudlinux_os_components/#compiling-your-own-extensions) 

## Integration of Apache modules in control panels

We provide control panels with the ability to build their own Apache packages with modules such as alt-mod-passenger, mod_lsapi, mod_hostinglimits. This is due to the fact that control panels often have own custom Apache. At the same time, the modules must be compiled with the version of Apache and APR libraries that control panels use. Therefore, if your control panel uses an Apache that is provided by CloudLinux OS, you can use native packages with modules. The following table represents the RPM package compatibility.


RPMs of Apache provided by CloudLinux, if you are using:

| | |
|-|-|
|httpd |httpd24-httpd |
|then install:<br>- mod_lsapi<br>- mod_hostinglimits<br>- alt-mod-passenger |then install:<br>- httpd24-mod_lsapi<br>- httpd24-mod_hostinglimits<br>- httpd24-alt-mod-passenger|


If you use custom Apache, follow this documentation on how to build a package for your Apache.

### alt-mod-passenger


1. Download alt-mod-passenger sources
```
yumdownloader --source alt-mod-passenger --enablerepo=*source
```

2. Rebuild the package with your Apache using rpmbuild
```
rpmbuild --rebuild alt-mod-passenger-ver.cloudlinux.src.rpm
```

3. Remove dependencies from httpd, apr, apr-util packages ( -devel packages as well) and set your Apache packages instead, in the alt-mod-passenger spec file. Also, set the path for the directory where your Apache configs are located with __apache_conf_dir in the spec file.

4. Rebuild the package again, this time you should get to the compilation stage. During compilation, you may encounter such errors:
```
* Checking for Apache 2 development headers...
     Found: no
* Checking for Apache Portable Runtime (APR) development headers...
     Found: no
* Checking for Apache Portable Runtime Utility (APU) development headers...
     Found: no
```

In this case, you should specify the paths of your custom Apache in  `alt-mod-passenger-5.3.7/utils/passenger` file. You need to add paths to binaries such as httpd, apu-1-config, and apxs. For example the following lines should be added before the passenger-install-apache2-module call:

```
export APU_CONFIG="${PATH_IN_YOUR_SYSTEM}/bin/apu-1-config"
export HTTPD="${PATH_IN_YOUR_SYSTEM}/sbin/httpd"
export APXS2="${PATH_IN_YOUR_SYSTEM}/bin/apxs"
```

5. Rebuild the package again, if everything is set correctly, there shouldn't be any problems.

6. Install the module, check that it is successfully loaded into Apache.

### mod_hostinglimits

1. Download mod_hostinglimits sources
```
yumdownloader --source mod_hostinglimits --enablerepo=*source
```

2. Rebuild the package with your Apache using rpmbuild
```
rpmbuild --rebuild mod_hostinglimits-ver.el7.cloudlinux.1.src.rpm
```

3. Remove dependencies such as httpd, ( -devel packages as well) and set your Apache packages instead, in the mod_hostinglimits spec file. Also, check the paths that lead to the “modules” or “conf.d” directories and change them if they are different on your system.

4. Rebuild the package again, this time you should get to the compilation stage. During compilation, you may encounter the following errors:
```
-- Not Found Apache Bin Directory: APACHE2_2_HTTPD_BIN-NOTFOUND, APACHE2_2_HTTPD_MODULES-NOTFOUND
-- Can't find Apache2.2: APACHE2_2_HTTPD_INCLUDE_DIR-NOTFOUND
```

In this case, you should specify the paths of your custom Apache in  `mod_hostinglimits-1.0/cmake/FindApacheForBuild.cmake` file. According to macros above, add the paths to the directory where httpd bin is located, to the dir where apache modules are located and to the dir where include files are located. It may look the following way:
```
FIND_PATH (APACHE2_2_HTTPD_INCLUDE_DIR 
   NAMES 
      httpd.h
   PATHS
      ${APACHE2_2_INCLUDES_DIR}
      /usr/local/include/httpd
      /usr/include/httpd
      /usr/include/apache
      /usr/local/apache/include/       # added line
)
```

5. Rebuild the package again, if you set everything correctly, there shouldn't be any problems.

6. Install the module, check that it is successfully loaded into Apache.

### mod_lsapi

1. Install the packages needed to build mod_lsapi from the source
```
yum install -y rpm-build
yum install -y liblsapi liblsapi-devel
yum install -y autoconf cmake apr-devel apr-util-devel
yum install -y python-lxml pytest python-mock
```

On CloudLinux 7 you also should install criu related packages
```
yum install -y criu-lve python-criu-lve criu-lve-devel crit-lve
```

2. Download mod_lsapi sources
```
yumdownloader --source mod_lsapi --enablerepo=*source
```

3. Rebuild the package with your Apache using rpmbuild
```
rpmbuild --rebuild mod_lsapi-ver.el7.cloudlinux.src.rpm
```

4. Remove building dependency from httpd-devel in the `mod_lsapi.spec` spec file located in `~/rpmbuild/SPECS/` directory.

5. Change the paths to copy `mod_lsapi.so` and `mod_lsapi.conf` in the %install section of the `mod_lsapi.spec` file. For example, if Apache root is `/usr/local/apache`, then modules directory and config files directory will be `/usr/local/apache/modules` and  `/usr/local/apache/conf.d` respectively. So for the config file you should substitute the following line in the _%install_ section:
```
install -D -m 644 conf/mod_lsapi.conf $RPM_BUILD_ROOT%{g_path}/confs/lsapi.conf
```

with the following line:
```
install -D -m 644 conf/mod_lsapi.conf $RPM_BUILD_ROOT/usr/local/apache/conf/lsapi.conf
```

And, also, for the module you should substitute the following line in the _%install_ section:
```
install -D -m 755 build/lib/mod_lsapi.so $RPM_BUILD_ROOT%{g_path}standard/mod_lsapi.so 
```

with the following line:
```
install -D -m 755 build/lib/mod_lsapi.so $RPM_BUILD_ROOT/usr/local/apache/modules/mod_lsapi.so
```

Also you should add mentions of both module and config files in the _%files_ section of mod_lsapi.spec. For example, for `/usr/local/apache` Apache root the following two lines  should be added:
```
%config /usr/local/apache/conf.d/lsapi.conf
/usr/local/apache/modules/mod_lsapi.conf
```

6. Rebuild the package again, this time you should get to the compilation stage. During compilation, you may encounter some errors:
```
-- Not Found Apache Bin Directory: APACHE2_2_HTTPD_BIN-NOTFOUND, APACHE2_2_HTTPD_MODULES-NOTFOUND
-- Can't find Apache2.2: APACHE2_2_HTTPD_INCLUDE_DIR-NOTFOUND
```

In this case, you should specify the paths of your custom Apache in  `mod_lsapi-1.1/cmake/FindApacheForBuild.cmake` file. According to macros above, add the paths to the directory where httpd binary is located, to the dir where apache modules are located and to the dir where include files are located. It may look a such way:
```
FIND_PATH (APACHE2_2_HTTPD_INCLUDE_DIR 
   NAMES 
      httpd.h
   PATHS
      ${APACHE2_2_INCLUDES_DIR}
      /usr/local/include/httpd
      /usr/include/httpd
      /usr/include/apache
      /usr/local/apache/include/       # added line
)
```

7. Rebuild the package again, if you set everything correctly, there shouldn't be any problems.

8. Install the module, check that it is successfully loaded into Apache.


## MySQL Governor

1. Install MySQL Governor
```
yum install governor-mysql
```

2. MySQL Governor supports only cl-MariaDB** or cl-MySQL** packages. Follow the documentation to figure out all supported versions. 
```
/usr/share/lve/dbgovernor/mysqlgovernor.py --mysql-version=mysqlXX
```

3. Backup your databases.

4. Run the cl-MySQL/cl-MariaDB installation.
```
/usr/share/lve/dbgovernor/mysqlgovernor.py --install
```

5. After installation, check that the database server is working properly. If you have any problems, contact support http://helpdesk.cloudlinux.com/

6. Configure user mapping to the database. The mapping format is described in the following [section](/cloudlinux_os_components/#mapping-a-user-to-a-database). The control panel should automatically generate such mapping and write it to `/etc/container/dbuser-map`. Usually, it is enough to write a hook when adding, deleting or renaming a database for a user. The control panel should implement such a mechanism for MySQL Governor to operate properly. MySQL Governor automatically applies changes from the dbuser-map file every five minutes.

7. MySQL Governor configuration can be found in the following [section](/cloudlinux_os_components/#configuration-3).

8. MySQL Governor CLI tools description can be found in the following [section](/command-line_tools/#mysql-governor)

9. After you've configured the mapping, use dbtop to see the current user load on the database (you'd need to make some database queries). 
```
dbtop
```

10. If the load appears in the dbtop output, then you have successfully configured MySQL Governor.

## CloudLinux LVE and CageFS patches

:::tip Note
If you are using Apache from the CloudLinux repository (such as httpd or httpd24-httpd), skip this chapter.
:::

If you use custom Apache, you need to apply patches so that the processes launched by the Apache are worked with LVE and CageFS properly. 
Cloudlinux provides patches for the following packages: 

*suphp
*suexec
*php-fpm
*apr library
*Apache mpm itk

First of all, download patches using the following link [https://repo.cloudlinux.com/cloudlinux/sources/da/cl-apache-patches.tar.gz](https://repo.cloudlinux.com/cloudlinux/sources/da/cl-apache-patches.tar.gz)

To apply the patch, follow these steps.

1. Untar archive which you were going to patch and go to the created directory with sources.

2. Transfer the patch to the directory with sources.

3. Use the patch utility inside the directory with sources

4. Recompile your project.

Here's an example with apr library:
```
# ls 
apr-1.4.8.tar.bz2  apr-2.4-httpd.2.patch
# tar xvjf apr-1.4.8.tar.bz2
# cp apr-2.4-httpd.2.patch ./apr-1.4.8
# cd ./apr-1.4.8
# patch -p3 < apr-2.4-httpd.2.patch 
patching file include/apr_thread_proc.h
Hunk #1 succeeded at 204 (offset -2 lines).
patching file threadproc/unix/proc.c

Patch applied Successfully, recompile apr. 
```

If you build custom RPM packages, it might make sense to apply patches using spec file.

In this case, add to the Source section (example for apr):
```
Source0: apr-1.4.8.tar.bz2
Patch100: apr-2.4-httpd.2.patch
```

Then  apply this patch in the _%setup_ section:
```
%prep
%setup -q
%patch100 -p3 -b .lve
```

If no errors appear during the build, the patch has been successfully applied.

## Hardened PHP

All versions of alt-php should work properly in any control panel.

To install all PHP versions, run:
```
yum groupinstall alt-php
```

To install a particular version of PHP, run:
```
yum groupinstall alt-php{some_version}
```

For details, see [PHP Selector Installation and Update](/cloudlinux_os_components/#installation-and-update-4)

## CloudLinux kernel set-up

### /proc filesystem access control

CloudLinux kernel allows the server administrator to control the user access to the `/proc` filesystem with the special parameters.
Unprivileged system users will not be able to see the processes of other system users, they can only see their processes. See details [here](/cloudlinux_os_kernel/#virtualized-proc-filesystem).

If the <span class="notranslate">`fs.proc_can_see_other_uid`</span> parameter is not defined in `sysctl` config during the CageFS package installation, it is set to `0`: <span class="notranslate">`fs.proc_can_see_other_uid=0`</span>.

You can also set this parameter in the <span class="notranslate">`/etc/sysctl.conf`</span> file and then run the <span class="notranslate">`sysctl -p`</span> command.

<span class="notranslate">[`hidepid`](/cloudlinux_os_kernel/#remounting-procfs-with-hidepid-option)</span> parameter is set based on the <span class="notranslate">`fs.proc_can_see_other_uid`</span> parameter value automatically during the <span class="notranslate">`lve_namespaces`</span> service starting (i.e. during <span class="notranslate">`lve-utils`</span> package installation and server booting).

#### Integrating control panel with CloudLinux

Run the following command to add administrators into <span class="notranslate">`proc_super_gid`</span> group (if the panel administrator is not a root user or there are several administrators):

<div class="notranslate">

```
/usr/share/cloudlinux/hooks/post_modify_admin.py create --username <admin_name>
```
</div>

:::tip Note 
This administrator should exist as a system user.
:::

This command also adds a specified administrator to the <span class="notranslate">`clsudoers`</span> group, so the LVE Manager admin's plugin works properly.

This command is a hook that the control panel should call after adding the admin user to the system.

If the [admins](/control_panel_integration/#admins) list script exists when installing <span class="notranslate">`lve-utils`</span> package then the existing administrators will be processed automatically. It the script does not exist, you should run the above mentioned command for each administrator.

### Symlink protection

CloudLinux kernels have symlink attacks protection. When this protection is enabled, the system does not allow users to create symlinks (and hard links) to non-existent/not their own files.

For details see [here](/cloudlinux_os_kernel/#link-traversal-protection). Here you can find all protection mechanism parameters.

See also: [the special module to protect web-server](/cloudlinux_os_kernel/#symlink-owner-match-protection).

You can run <span class="notranslate">`cldiag`</span> utility with the <span class="notranslate">`--check-symlinkowngid`</span> parameter to check web-server protection mechanism proper set-up.

<div class="notranslate">

```
# cldiag --symlinksifowner
Check fs.enforce_symlinksifowner is correctly enabled in sysctl conf:
	OK: fs.enforce_symlinksifowner = 1

There are 0 errors found.
# cldiag --check-symlinkowngid
Check fs.symlinkown_gid:
	OK: Web-server user is protected by Symlink Owner Match Protection

There are 0 errors found.
```
</div>

* Symlink traversal protection is disabled by default.
* Symlink owner match protection is enabled by default.

#### How to enable symlink protection

Set the <span class="notranslate">`symlinkown_gid`</span> parameter to the value of the GID group, which the web-server process has.

Before enabling symlink protection, ensure it does not break the control panel functionality.

For example, if an unprivileged user is allowed to create symlinks or hard links to files or directories he doesn’t own then when enabling symlink traversal protection he will not be able to do it.

To allow an unprivileged user to create such symlink or hard link, a file/directory which symlink is referred to should have <span class="notranslate">`linksafe`</span> group.

### File system quotas

Some of CloudLinux OS utilities (`cagefsctl`, `selectorctl`, `cloudlinux-selector`, etc) writes files to the user’s home directory on behalf of this user.

So they are subject to the disk quotas if any are set for the user. The utilities can override these quotas to avoid failures.
See [File system quotas](/cloudlinux_os_kernel/#file-system-quotas) for kernel parameters controlling these permissions. This parameter is used only for XFS file system.

