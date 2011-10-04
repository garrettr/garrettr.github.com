---
layout: post
title: Setting up cron with Django on Webfaction
categories:
- django
- sysadmin
published: true
---

[cron](http://en.wikipedia.org/wiki/Cron) is a Unix tool that allows you to schedule tasks to run at
regular intervals. I'm using it on The Alliance for Appalachia
[website](http://appalliance.webfactional.com) in conjunction
with [Django admin commands](https://docs.djangoproject.com/en/dev/howto/custom-management-commands/).
Getting it to work was more involved than I expected, so I thought I'd share what I learned.

## Basic crontab

Here's the basic crontab. Fill in `username`, `X.Y`, `yourproject`, and the path to your script to
fit your installation and needs.

    PYTHONPATH=/home/username/lib/pythonX.Y:/home/username/webapps/django/lib/pythonX.Y
    DJANGO_SETTINGS_MODULE=yourproject.settings
    ...
    */10 * * * * /usr/local/bin/pythonX.Y /home/username/path/to/script
    ...

The tricky part is commands given in cron aren't run in a shell - so any environment variables,
settings in `.bashrc` or other configuration files, and so on are not available to these commands
when they're run. In my experience, this lead to some weird stack traces from when the scripts
tried to run.

The solution is to

1.  Define `PYTHONPATH`. This should be a colon-separated list of absolute paths to directories
    that contain your Python modules. The ones listed above are the standard ones in a
    Webfaction Django installation.
2.  Define `DJANGO_SETTINGS_MODULE`. This is necessary if you're going to run management
    commands using manage.py.
3.  Define your cron commands with the absolute path to both the Python interpreter and
    the script. In my experience, cron *did* properly expand `~` to the user's $HOME, but I'm
    not sure if that behavior is universal.

That's it! I cobbled this solution together with help from several sources. For further reading,
see

1.  [Webfaction support thread on cron w/ Django](http://forum.webfaction.com/viewtopic.php?id=1258)
2.  [Webfaction documentation on cron](http://docs.webfaction.com/software/general.html#scheduling-tasks-with-cron)
3.  [Wikipedia on cron - helpful description of job syntax](http://en.wikipedia.org/wiki/Cron)

## Working with multiple Django/Python projects

You have multiple Python projects, but only one crontab. The only problem with the above solution is it seems to limit you to running manage.py scripts from a
single Django application, or potentially running into problems extending the `PYTHONPATH` for
multiple Python applications you could have running on your Webfaction instance.

Redditor [nik_doof](http://www.reddit.com/user/nik_doof) suggested a simple solution to this
problem in the comments on this article. You can re-define any environment variables set in a
crontab at any time. Cron commands will use the closest previously defined environment variables
when they run. For example:

    PYTHONPATH=/home/username/lib/pythonX.Y:/home/username/webapps/django/lib/pythonX.Y
    DJANGO_SETTINGS_MODULE=yourproject.settings
    ...
    */10 * * * * /usr/local/bin/pythonX.Y /home/username/path/to/script
    ...
    PYTHONPATH=/home/username/lib/pythonX.Y:/home/username/webapps/otherproject/lib/pythonX.Y
    DJANGO_SETTINGS_MODULE=yourotherproject.settings
    ...
    */10 * * * * /usr/local/bin/pythonX.Y /home/username/path/to/script
    ...

## Tips

If you're having trouble debugging your crontab, it's helpful to add

    MAILTO=youremail@domain.com

at the top of the file. This will e-mail you debugging information every time the jobs run - so
you may want to disable it when you're done debugging. You can also set multiple MAILTO variables,
so different cron jobs can have their debugging information sent to different e-mail addresses.

Another useful thing to do in your crontab is end your job lines like this:

    */10 * * * * /usr/local/bin/pythonX.Y /home/username/path/to/script > /dev/null 2>&1

You might want to do this if your scripts print anything to standard output in the course of their
execution.

    ... > /dev/null

redirects output to [a special Unix device](http://en.wikipedia.org/wiki//dev/null) that discards any input given to it. `2>&1` additionally redirects `stderr` to `stdout`, so all output from the
program will be discarded. This prevents your cron jobs from randomly printing stuff into your
terminal when they run, but allows you to keep code that prints debugging info for development
purposes.

You could also redirect to a special file if that would be useful, although I prefer to
do my logging inside the code, using Python's `logging` module. I may write more about logging at a later
date; in the meantime, you can check out some of the logging code I use [here](https://github.com/handsomeransoms/afa).
