php-eds-daemon
==============
http://kvz.io/blog/2009/01/09/create-daemons-in-php

Create daemons in PHP
=====================
Everyone knows PHP can be used to create websites. But it can also be used to create desktop applications and commandline tools. And now with a class called System_Daemon, you can even create daemons using nothing but PHP. And did I mention it was easy?

This class was relevant in 2009, and may still be to some people, but if you want to daemonize a php script nowadays, a 5-line Ubuntu upstart script should suffice. Upstart will daemonize your script and can even respawn it if it goes down. Something that you'd otherwise have to use monit for. I have another article on daemonizing nodejs scripts with upstart, that can just as well be applied to PHP scripts.

If you're still convinced you need to do this with pure PHP, read on.

What is a Daemon?
=================
A daemon is a Linux program that run in the background, just like a 'Service' on Windows. It can perform all sorts of tasks that do not require direct user input. Apache is a daemon, so is MySQL. All you ever hear from them is found in somewhere in /var/log, yet they silently power over 40% of the Internet.

You reading this page, would not have been possible without them. So clearly: a daemon is a powerful thing, and can be bend to do a lot of different tasks.

Why PHP?
========
Most daemons are written in C. It's fast & robust. But if you are in a LAMP oriented company like me, and you need to create a lot of software in PHP anyway, it makes sense:

Reuse & connect existing code Think of database connections, classes that create customers from your CRM, etc.
Deliver new applications very fast PHP has a lot of build in functions that speed up development greatly.
Everyone knows PHP (right?) If you work in a small company: chances are there are more PHP programmers than there are C programmers. What if your C guy abandons ship? Admittedly it's a very pragmatic reason, but it's the same reason why Facebook is building HipHop.
Possible use cases

Website optimization If you're running a (large) website, jobs that do heavy lifting should be taken away from the user interface and scheduled to run on the machine separately.
Log parser Continually scan logfiles and import critical messages in your database.
SMS daemon Read a database queue, and let your little daemon interface with your SMS provider. If it fails, it can easily try again later!
Video converter (think Youtube) Scan a queue/directory for incoming video uploads. Make some system calls to ffmpeg to finally store them as Flash video files. Surely you don't want to convert video files right after the upload, blocking the user interface that long? No: the daemon will send the uploader a mail when the conversion is done, and proceed with the next scheduled upload
Deamons vs Cronjobs

Some people use cronjobs for the same Possible use cases. Crontab is fine but it only allows you to run a PHP file every minute or so.

What if the previous run hasn't finished yet? Overlap can seriously damage your data & cause siginificant load on your machines.
What if you can't afford to wait a minute for the cronjob to run? Maybe you need to trigger a process the moment a record is inserted?
What if you want to keep track of multiple 'runs' and store data in memory.
What if you need to keep your application listening (on a socket for example) Cronjobs are a bit rude for this, they may spin out of control and don't provide a
framework for logging, etc. Creating a daemon would offer more elegance & possibilities. Let's just say: there are very good reasons why a Linux OS isn't composed entirely of Cronjobs : )

How it works internally
========================
(*Nerd alert!*) When a daemon program is started, it fires up a second child process, detaches it, and then the parent process dies. This is called forking. Because the parent process dies, it will give the console back and it will look like nothing has happened. But wait: the child process is still running. Even if you close your terminal, the child continues to run in memory, until it either stops, crashes or is killed.

In PHP: forking can be achieved by using the Process Control Extensions. Getting a good grip on it, may take some studying though.

System_Daemon
=============
Because the Process Control Extensions' documentation is a bit rough, I decided to figure it out once, and then wrap my knowledge and the required code inside a PEAR class called: System_Daemon. And so now you can just:
```php
<?php
require_once "System/Daemon.php";                 // Include the Class

System_Daemon::setOption("appName", "mydaemon");  // Minimum configuration
System_Daemon::start();                           // Spawn Deamon!
?>
```

And that's all there is to it. The code after that, will run in your server's background. So next, if you create a while(true) loop around that code, the code will run indefinitely. Remember to build in a sleep(5) to ease up on system resources.

Features & Characteristics
==========================
Here's a grab of System_Daemon's features:
 - Daemonize any PHP-CLI script
 - Simple syntax
 - Driver based Operating System detection
 - Unix only
 - Additional features for Debian / Ubuntu based systems like:
 - Automatically generate startup files (init.d)
 - Support for PEAR's Log package
 - Can run with PEAR (more elegance & functionality) or without PEAR (for standalone packages)
 - Default signal handlers, but optionally reroute signals to your own handlers.
 - Set options like max RAM usage

Download
========
You could download the package from PEAR, or, if you have PEAR installed on your system: just run this from a console:
```
$ pear install -f System_Daemon
```
You can also update it using that last command.

Beta
=====
Though I have quite some daemons set up this way, it's officially still beta. So please report any bugs over at the PEAR page. Other comments may be posted here.

Complex Example
===============
Ready to dig a little deeper? This example program is called 'logparser', it takes a look at a more complex use of System_Daemon. For instance, it introduces command line arguments like:

--no-daemon              # just run the program in the console this time
--write-initd            # writes a startup file
Read this source to get the picture. Don't forget the comments!
```php
#!/usr/bin/php -q
<?php
/**
 * System_Daemon turns PHP-CLI scripts into daemons.
 *
 * PHP version 5
 *
 * @category  System
 * @package   System_Daemon
 * @author    Kevin &lt;kevin@vanzonneveld.net>
 * @copyright 2008 Kevin van Zonneveld
 * @license   http://www.opensource.org/licenses/bsd-license.php
 * @link      http://github.com/kvz/system_daemon
 */

/**
 * System_Daemon Example Code
 *
 * If you run this code successfully, a daemon will be spawned
 * but unless have already generated the init.d script, you have
 * no real way of killing it yet.
 *
 * In this case wait 3 runs, which is the maximum for this example.
 *
 *
 * In panic situations, you can always kill you daemon by typing
 *
 * killall -9 logparser.php
 * OR:
 * killall -9 php
 *
 */

// Allowed arguments & their defaults
$runmode = array(
    'no-daemon' => false,
    'help' => false,
    'write-initd' => false,
);

// Scan command line attributes for allowed arguments
foreach ($argv as $k=>$arg) {
    if (substr($arg, 0, 2) == '--' && isset($runmode[substr($arg, 2)])) {
        $runmode[substr($arg, 2)] = true;
    }
}

// Help mode. Shows allowed argumentents and quit directly
if ($runmode['help'] == true) {
    echo 'Usage: '.$argv[0].' [runmode]' . "\n";
    echo 'Available runmodes:' . "\n";
    foreach ($runmode as $runmod=>$val) {
        echo ' --'.$runmod . "\n";
    }
    die();
}

// Make it possible to test in source directory
// This is for PEAR developers only
ini_set('include_path', ini_get('include_path').':..');

// Include Class
error_reporting(E_STRICT);
require_once 'System/Daemon.php';

// Setup
$options = array(
    'appName' => 'logparser',
    'appDir' => dirname(__FILE__),
    'appDescription' => 'Parses vsftpd logfiles and stores them in MySQL',
    'authorName' => 'Kevin van Zonneveld',
    'authorEmail' => 'kevin@vanzonneveld.net',
    'sysMaxExecutionTime' => '0',
    'sysMaxInputTime' => '0',
    'sysMemoryLimit' => '1024M',
    'appRunAsGID' => 1000,
    'appRunAsUID' => 1000,
);

System_Daemon::setOptions($options);

// This program can also be run in the forground with runmode --no-daemon
if (!$runmode['no-daemon']) {
    // Spawn Daemon
    System_Daemon::start();
}

// With the runmode --write-initd, this program can automatically write a
// system startup file called: 'init.d'
// This will make sure your daemon will be started on reboot
if (!$runmode['write-initd']) {
    System_Daemon::info('not writing an init.d script this time');
} else {
    if (($initd_location = System_Daemon::writeAutoRun()) === false) {
        System_Daemon::notice('unable to write init.d script');
    } else {
        System_Daemon::info(
            'sucessfully written startup script: %s',
            $initd_location
        );
    }
}

// Run your code
// Here comes your own actual code

// This variable gives your own code the ability to breakdown the daemon:
$runningOkay = true;

// This variable keeps track of how many 'runs' or 'loops' your daemon has
// done so far. For example purposes, we're quitting on 3.
$cnt = 1;

// While checks on 3 things in this case:
// - That the Daemon Class hasn't reported it's dying
// - That your own code has been running Okay
// - That we're not executing more than 3 runs
while (!System_Daemon::isDying() && $runningOkay && $cnt <=3) {
    // What mode are we in?
    $mode = '"'.(System_Daemon::isInBackground() ? '' : 'non-' ).
        'daemon" mode';

    // Log something using the Daemon class's logging facility
    // Depending on runmode it will either end up:
    //  - In the /var/log/logparser.log
    //  - On screen (in case we're not a daemon yet)
    System_Daemon::info('{appName} running in %s %s/3',
        $mode,
        $cnt
    );

    // In the actuall logparser program, You could replace 'true'
    // With e.g. a  parseLog('vsftpd') function, and have it return
    // either true on success, or false on failure.
    $runningOkay = true;
    //$runningOkay = parseLog('vsftpd');

    // Should your parseLog('vsftpd') return false, then
    // the daemon is automatically shut down.
    // An extra log entry would be nice, we're using level 3,
    // which is critical.
    // Level 4 would be fatal and shuts down the daemon immediately,
    // which in this case is handled by the while condition.
    if (!$runningOkay) {
        System_Daemon::err('parseLog() produced an error, '.
            'so this will be my last run');
    }

    // Relax the system by sleeping for a little bit
    // iterate also clears statcache
    System_Daemon::iterate(2);

    $cnt++;
}

// Shut down the daemon nicely
// This is ignored if the class is actually running in the foreground
System_Daemon::stop();
?>
```
Console action: Controlling the Daemon
=======================================
Now that we've created an example daemon, it's time to fire it up! I'm going to assume the name of your daemon is logparser. This can be changed with the statement: System_Daemon::setOption('appName', 'logparser'). But the name of the daemon is very important because it is also used in filenames (like the logfile).

Execute
=======
Just make your daemon script executable, and then execute it:
```
$ chmod a+x ./logparser.php
$ ./logparser.php
```

Check
======
Your daemon has no way of communicating through your console, so check for messages in:
```
$ tail /var/log/logparser.log
```

And see if it's still running:
```
$ ps uf -C logparser.php
Kill
```

Without the start/stop files (see below for howto), you need to:
```
$ killall -9 logparser.php
```

Autch.. Let's work on those start / stop files, right?

Start / Stop files (Debian & Ubuntu only)

Real daemons have an init.d file. Remember you can restart Apache with the following statement?
```
$ /etc/init.d/apache2 restart
```
That's a lot better than killing. So you should be able to control your own daemon like this as well:
```
$ /etc/init.d/logparser stop
$ /etc/init.d/logparser start
```
Well with System_Daemon you can write autostartup files using the writeAutoRun() method, look:
```php
<?php
$path = System_Daemon::writeAutoRun();
?>
```
On success, this will return the path to the autostartup file: /etc/init.d/logparser, and you're good to go!

Run on boot
===========
Surely you want your daemon to run at system boot.. So on Debian & Ubuntu you could type:
```
$ update-rc.d logparser defaults
```
Done your daemon now starts every time your server boots. Cancel it with:
```
$ update-rc.d -f logparser remove
```

Logrotate
=========
Igor Feghali shares with us a logrotate config file to keep your log files from growing too large. Just place a file in your /etc/logrotate.d/.
```
/var/log/mydaemon.log {
   rotate 15
   compress
   missingok
   notifempty
   sharedscripts
   size 5M
   create 644 mydaemon_user mydaemon_group
   postrotate
       /bin/kill -HUP `cat /var/run/mydaemon/mydaemond.pid 2>/dev/null` 2> /dev/null || true
   endscript
}
```

Obviously, replace the mydaemon occurances with values corresponding to your environment. Thanks Igor!

Troubleshooting
===============
Here are some issues you may encounter down the road.

Don't use echo() Echo writes to the STDOUT of your current session. If you logout, that will cause fatal errors and the daemon to die. Use System_Daemon::log() instead.
Connect to MySQL after you start() the daemon. Otherwise only the parent process will have a MySQL connection, and since that dies.. It's lost and you will get a 'MySQL has gone away' error.
Error handling Good error handling is imperative. Daemons are often mission critical applications and you don't want an uncatched error to bring it to it's knees.
Reconnect to MySQL A connection may be interrupted. Think about network downtime or lock-ups when your database server makes backups. Whatever the cause: You don't want your daemon to die for this, let it try again later.
PHP error handler As of 0.6.3, System_Daemon forwards all PHP errors to the log() method, so keep an eye on your logfile. This behavior can be controlled using the logPhpErrors (true||false) option.
Monit Monit is a standalone program that can kickstart any daemon, based on your parameters. Should your daemon fail, monit will mail you and try to restart it.
Watch that memory Some classes keep a history of executed commands, sent mails, queries, whatever. They were designed without knowing they would ever be used in a daemonized environment. Cause daemons run indefinitely this 'history' will expand indefinitely. Since unfortunately your server's RAM is not infinite, you will run into problems at some point. This makes it's very important to address these memory 'leaks' when building daemons.
Statcache will corrupt your data If you do a file_exists(), PHP remembers the results to ease on your disk until the process end. That's ok but since the Daemon process does not end, PHP will not be able to give you up to date information. As of 0.8.0 you should call System_Daemon::iterate(2) instead of e.g. sleep(2), this will sleep & clear the cache and give you fresh & valid data.

I know I'm saying MySQL a lot, but you can obviously replace that with Oracle, MSSQL, PostgreSQL, etc.
