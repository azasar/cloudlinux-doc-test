# X-Ray (BETA)

[[TOC]]

:::tip Note
Please note that X-Ray is a new experimental tool. It is in beta testing now. If you’d like to be a beta tester, please fill out [this form](https://cln.cloudlinux.com/console/dashboard/products).
:::

## Description

X-Ray is a tool developed for website performance monitoring and performance issues detection.

X-Ray can gather and visualize information about top N slowest system functions, external requests, software modules and database queries of the client’s website.

First release of X-Ray is offered for cPanel administrators and support to find the cause of website performance issues.

X-Ray can monitor websites that were developed on cPanel hosts and use PHP (see [PHP version list](/x-ray/#list-of-supported-php-versions)) or WordPress.

## Installation

If you are cPanel CloudLinux client you can use the X-Ray application.

1. Make sure you have installed **LVE Manager version 6.2 or later**. You can install it with the following command:
   
   <div class="notranslate">

    ```
    # yum install lvemanager --enablerepo=cloudlinux-updates-testing
    ```
    </div>

2. Then install the <span class="notranslate">`alt-php-xray`</span> package

    * Via user interface
        * Go to the <span class="notranslate">_X-Ray_</span> tab.
        * Click <span class="notranslate">_Install_</span> to start installation.

        ![](/images/XRayUI.png)

    * Via SSH by running the following command:
  
    <div class="notranslate">

    ```
    # yum install lvemanager alt-php-xray --enablerepo=cloudlinux-updates-testing
    ```
    </div>

3. After installation, use the <span class="notranslate">_Start tracing_</span> button to create your first tracing task for a slow site.

![](/images/XRayStartTracing.png)

## How to manage X-Ray 

### Create a new tracing task

1. Go to the <span class="notranslate">_X-Ray_</span> tab
2. Click the <span class="notranslate">_Start tracing_</span> button to create a new task
3. In the opened popup specify a website URL to trace
4. Click the <span class="notranslate">_Run_</span> button
5. Tracing will run in the default mode. In the default mode X-Ray traces the first 20 requests for a specified URL

![](/images/XRayTracingTask.png)

* **URL** should be a valid URL of the domain which exists on the current hosting server. The URL field supports wildcard matching. To learn more about wildcard matching, click _How to use special characters_.
* **Advanced settings** allow you to set an IP address and tracing options: by time or by number of queries.

    ![](/images/XRayAdvanced.png)

#### Advanced settings

* <span class="notranslate">**Client’s IP**</span>: it is an IPv4 address of a machine to trace. For example, if you have a production website that processes requests from different IP addresses and you do not want to add these requests to the tracing task. So, you can set a specific IP address and X-Ray will analyze requests only from this specific IP address.
Record for
* <span class="notranslate">**Time period**</span>: how much time X-Ray collect the requests (2 days max)
* <span class="notranslate">**Requests**</span>: the amount of requests that X-Ray will collect

After creating, the task appears in the list of tracing tasks.

![](/images/XRayTrcingTaskList.png)

### View tracing tasks list

![](/images/XRayTrcingTaskList1.png)

#### Tracing status

A tracing task can have the following statuses:

* <span class="notranslate">**Running**</span> – tracing is in progress
* <span class="notranslate">**Stopped**</span> – tracing was stopped by administrator
* <span class="notranslate">**On hold**</span> – the same URL already exists in the lists. Task processing will not start automatically. Administrator should start it manually.
* <span class="notranslate">**Completed**</span> – period of time is finished or number of requests is reached.

### Stop tracing task

Click ![](/images/XRayStop.png) to stop the tracing task.

![](/images/XRayStopped.png)

The tracing task status will be changed to <span class="notranslate">**Stopped**</span>. Data will not be collected anymore but you can see already collected information or continue tracing later by clicking ![](/images/XRayStart.png).

### Delete tracing task 

Click ![](/images/XRayDelete.png) to delete the tracing task.

:::warning Warning!
When you have deleted a tracing task, all collected data will be unavailable.
:::

### View collected requests for tracing task

:::warning Warning!
Collected requests are available in the UI for two weeks.
:::

Click ![](/images/XRayView.png) to open a list of collected requests.

#### Tracing tasks

![](/images/XRayCollectedRequests.png)

The slowest request is highlighted.

![](/images/XRaySlowestRequest.png)

* <span class="notranslate">**Collected requests**</span> displays how many requests were collected according to tasks requirements.
* <span class="notranslate">**Pending requests**</span> displays how many of collected requests are not visible in the table yet.

X-Ray collects the following data for each request:

* **Top issues** – the slowest items of a request
* **Software modules/plugins** by execution time (only for WordPress plugins)
* **Database queries** by execution time 
* **External requests** by execution time
* **Other system functions** by execution time 

#### Software modules/plugins

![](/images/XRaySoftwareModulesPlugins.png)

The <span class="notranslate">_Software modules/plugins_</span> section displays the following data:

* **Software type** – a type a module/plugin. For now, X-Ray can analyze only WordPress software
* **Software module** – a name of the WordPress plugin
* **Duration** – plugin execution time
* **Duration (%)** – plugin execution time as a percentage of the total duration of the request

#### Database queries

![](/images/XRayDatabaseQueries.png)

The <span class="notranslate">_Database queries_</span> section displays the following data:

* **Query** – the executed SQL-query
* **File** – the file and the line of the executed query and backtrace
* **Software module** – a WordPress plugin name from which the request was completed. If the request does not belong to any of the WordPress plugin, the name of the function that executed the given request is displayed
* **Calls** – the number of identical SQL queries
* **Duration** – execution time as a percentage of the total duration of a request and the function processing time (in brackets)
 
#### External requests

![](/images/XRayExternalRequests.png)

The <span class="notranslate">_External requests_</span> section displays the following data:

* **URL** – the URL of the executed request
* **File** – the file and the line of the executed request and backtrace
* **Duration** – execution time as a percentage of the total duration of a request and the function processing time (in brackets)
 
#### System functions

![](/images/XRaySystemFunctions.png)

The <span class="notranslate">_System functions_</span> section displays the following data:

* **Function** – the executed function
* **File** – the file and the line of the executed request
* **Duration** – execution time as a percentage of the total duration of a request and the function processing time (in brackets)

## X-Ray client

X-Ray client is a PHP extension named <span class="notranslate">`xray.so`</span>. It analyzes the processing time of the entire request and its parts and then sends the data to the X-Ray agent.

### List of supported PHP versions

The list of currently supported PHP versions:

| | |
|-|-|
|**ALT PHP**:|**EA PHP**:|
| <ul><li>alt-php54</li><li>alt-php55</li><li>alt-php56</li><li>alt-php70</li><li>alt-php71</li><li>alt-php72</li><li>alt-php73</li><li>alt-php74</li></ul>|<ul><li>ea-php54</li><li>ea-php55</li><li>ea-php56</li><li>ea-php70</li><li>ea-php71</li><li>ea-php72</li><li>ea-php73</li><li>ea-php74</li></ul>|

### Functions that X-Ray client can hook

#### Database queries

* Functions from [MySQL](https://www.php.net/manual/ru/book.mysql.php) extension:
    * `mysql_query`
    * `mysql_db_query`
    * `mysql_unbuffered_query`
* Functions from [MySQLi](https://www.php.net/manual/ru/book.mysqli.php) extension:
    * `mysqli_query`
    * `mysqli::query`
    * `mysqli_multi_query`
    * `mysqli::multi_query`
    * `mysqli_real_query`
    * `mysqli::real_query`
* Functions from [PDO](https://www.php.net/manual/ru/book.pdo.php) extension:
    * `PDO::exec`
    * `PDO::query`
    * `PDOStatement::execute`

#### External requests

* Function [curl_exec](https://www.php.net/manual/ru/function.exec)

#### System PHP functions

It may be any PHP system function which can be related to a PHP engine or other PHP extension, for example `fopen()` or `json_encode()`. A list of these functions can be found [here](https://www.php.net/manual/en/indexes.functions.php).

### Configuration Options

#### xray.enabled

**Syntax**: `xray.enabled=On/Off`

**Default**: On

**Changeable**: PHP_INI_SYSTEM

**Description**: Enable or disable X-Ray extension from php.ini

-----

#### xray.database_queries

**Syntax**: `xray.database_queries=[number]`

**Default**: 20

**Changeable**: PHP_INI_SYSTEM 

**Description**: The number of the slowest SQL queries which will be sent to the X-Ray agent. The min value is 0 and the max value is 100. If the variable value is more, the default value will be used.

-----

#### xray.external_requests

**Syntax**: `xray.external_requests=[number]`

**Default**: 20

**Changeable**: PHP_INI_SYSTEM  

**Description**: The number of the slowest external requests (the curl_exec function) which will be sent to the X-Ray agent. The min value is 0 and the max value is 100. If the variable value is more, the default value will be used.

-----

#### xray.system_functions

**Syntax**: `xray.system_functions=[number]`

**Default**: 20

**Changeable**: PHP_INI_SYSTEM  

**Description**: The number of the slowest system functions which will be sent to the X-Ray agent. 
The min value is 0 and the max value is 100. If the variable value is more, the default value will be used.

-----

#### xray.backtrace_depth

**Syntax**: `xray.backtrace_depth=[number]`

**Default**: 10

**Changeable**: PHP_INI_SYSTEM  

**Description**: The backtrace depth to the main() function which will be sent to the X-Ray agent. The min value is 0 and the max value is 20. If the variable value is more, the default value will be used.

-----

#### xray.processor

**Syntax**: `xray.processor=[processor_name]`

**Default**: xray

**Changeable**:  PHP_INI_SYSTEM 

**Description**: Tells the X-Ray client which processor to use. The new processors may be added in the future. The default processor is xray which means to send data to the X-Ray agent.

-----

#### xray.tasks

**Syntax**: `xray.tasks=host:uri:ip:id`

**Default**: no value

**Changeable**:  PHP_INI_SYSTEM 

**Description**: The current tracing tasks for the given PHP request. This directive is added automatically by the X-Ray manager when creating a task. It is not allowed to edit manually, as X-Ray may stop working.

-----

#### xray.to_file

**Syntax**: `xray.to_file=On/Off`

**Default**: Off

**Changeable**:  PHP_INI_SYSTEM 

**Description**: Only for debug purposes. Writes to a file data which is sent to the processor.

-----

#### xray.debug

**Syntax**: `xray.debug=On/Off`

**Default**: Off

**Changeable**: PHP_INI_SYSTEM 

**Description**: Only for debug purposes. Enables debug output during request processing. In the On mode can slow down the domain.

-----

#### xray.debug_file

**Syntax**: `xray.debug_file=[path_to_file]`

**Default**: `/tmp/xray-debug.log`

**Changeable**: PHP_INI_SYSTEM 

**Description**: Only for debug purposes. Specifies a file for logging debug information.

## X-Ray agent 
This is a service that receives data from the X-Ray client and sends it to the remote storage.

### Managing X-Ray service

The X-Ray agent is managed by the <span class="notranslate">`service`</span> utility.

* To start the X-Ray agent, run the following command:

    <div class="notranslate">

    ```
    # service xray-agent start
    ```
    </div>

* To stop the X-Ray agent, run the following command:

    <div class="notranslate">

    ```
    # service xray-agent stop
    ```
    </div>

* To restart the X-Ray agent, run the following command:

    <div class="notranslate">

    ```
    # service xray-agent restart
    ```
    </div>

## FAQ

### Does X-Ray affect website performance?

X-Ray affects website performance. Our tests show 5-10 % overhead from a website loading time with X-Ray tracing enabled.
X-Ray allows you to find website performance issues and should not be enabled permanently. If your website is very slow, you can enable X-Ray to find the cause and then disable it.

### What should I do if I see the warning “Task is duplicated by URL”?

This warning means that you already have a task to trace this URL in the list of your tracing tasks. If you see this warning, a new task can be created only with the <span class="notranslate">_On hold_</span> status and you will be able to run it only when the previous task with the same URL will be completed.

Note that the URL field supports wildcard matching and you can have a case when X-Ray is tracing the `URL=domain.com/*` and you are trying to create a new task with `URL=domain.com/xray.php`. In this case, you will see that warning because the `*` URLs array includes `xray.php`.

###  I started a tracing task and made requests to URL but did not see any results in the UI. What should I do?

1. Check that <span class="notranslate">**xray**</span> extension is enabled for the domain. To do so, go to the `phpinfo()` page and make a request. In the phpinfo output try to find the following section:
   
    ![](/images/XRayPHPInfo.png)

If you cannot see that section, try to restart PHP processes for that user (the simplest way is to restart Apache) and check that you can see the <span class="notranslate">**xray**</span> extension.


2. If you can see the <span class="notranslate">**xray**</span> extension in the phpinfo, check that X-Ray agent service is running with the service xray-agent status command. If it is not running, start it with the <span class="notranslate">`service xray-agent start`</span> command.
3. If you set a client’s IP when creating the tracing task, check that your requests come to the server with this IP via phpinfo (since there may be NAT between your local machine and the server).
   
    ![](/images/XRayPHPInfoRemoteAddr.png)

4. If, after checking the previous items, the issue persists, [contact our support team](https://cloudlinux.zendesk.com/hc/en-us/requests/new).




