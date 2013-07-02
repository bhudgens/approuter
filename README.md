This thing is intended to manage multiple versions of an application 
running simultaneously.  It will manage two types of running applications:
production and alternate.  

**production** - the 'in production' version of the application, this version
    will be managed as if it's presence were important and upgrades 
    to the underlying code (changes detected in the repo) will cause
    a graceful switch where the users should notice no downtime.

**alternate** - new 'in test' versions of the application.  Changes from the source
    repo for these applications are immediately applied with the existing version
    being forcibly replaced with the new.  The assumption here is that this 
    version is being worked on and you want your changes to be immediately
    available.

Other definitions

instance - a named instance of an application for example, the the case
    of having multiple alternate instances one might have alternate instances
    a, b, and c.  Production type applications only have two instances
    active and inactive.  

Routing amongst the application instances is managed by cookie, with the 
presence of the a type/instance cookie causing the request to be routed
to the correct type/instance.

## Filesystem Structure

**managed/** - This directory is where all the managed repos (applications) and the configuration
  data to manage them is stored.

**managed/var/** - This directory contains all the generated output from running the managed apps.
    This includes: log files, config files, run data (pidfiles) etc.  It also includes
    the generated crontab that can be used to monitor the managed applications for 
    remote changes (from their repository as defined by the repo.conf).

**managed/alternates/** - This directory is where the alternate versions of the application will be checked out
    and run from.  Under this directory will be a directory for each alternate configured
    e.g. managed/alternates/beta

**managed/prod/** - This directory will contain two directories (instance1 and instance2) that contain
    the production application instances that will be used to handle production traffic.  

**managed/var/log/** - This directory contains the log files generated by approuter and the applications
    running within.  Each production and alternate instance gets a directory by name
    which will contain the log files for that application instance.  The log for each
    application will be named 'current' e.g. managed/var/log/gamma/current.
    
#### Configuration 

etc/repo.conf
    Stores the repo url (git) that refers the the code managed by this instance of the
    approuter. This is the repository that contains the code that will be cloned into
    the production and alternate locations, and ultimately run.

etc/alternates.conf
    A list of branche names that will be run as alternates.  It is expected that the branches
    referenced already exist.  The file should contain one unique branch name
    per line.

etc/prod_branch
    The name of the production branch that should be used.  This is only necessary
    if it is desirable to use a branch other than the default branch as configured
    in git.

## AppRouter Controls

start
    Starts up the approuter environment, including all the managed applications. 
    Applications must be startable using an executable file in the root of the application
    named start, to which the first (and only) parameter passed will be the port to
    which the application should bind for incoming requests. The application will be
    started something like:

        pushd <app dir>
        ./start 3882

    The start executable should perform all necessary steps to setup the environment
    and get the application running.

    Once the environment is started, other controls become available

shutdown
    Shuts down the approuter environment in it's entirety, this will shutdown all the
    managed instances, nginx, and the process monitor responsible for keeping things 
    running.

### Other Controls
- **status** - Displays information about nginx and the running instances (think ps), something
    like this ( the second column is the service name which is used with other commands):


        [+ +++ +++]  beta       uptime: 159s/323s  pids: 35930/34714
        [+ +++ +++]  gamma      uptime: 323s/323s  pids: 34717/34716
        [+ +++ +++]  instance1  uptime: 323s/323s  pids: 34719/34718
        [+ +++ +++]  instance2  uptime: 323s/323s  pids: 34721/34720
        [+ +++ +++]  nginx      uptime: 323s/323s  pids: 34723/34722

- **switch_nginx_config** / **switch** - Toggles between the two running production instances, this switches out the configuration
    and HUPs nginx to make the switch seemless for connected users.

- **start <service name>** - Attempts to start a service managed by approuter, the status command will show
    a list of all the known service names.

- **stop <service name>** - Attempts to stop a single service managed by approuter.

- **which_prod** - Prints out the name (instance1 or instance2) of the production instance that is
    currently active (being sent traffic by the current ngingx configuration).

Usage
    - Edit the repo.conf, alternates.conf and starter.conf files according to the information
    given above
    - source the environment file in the root 
    - type start!


## Application Structure

- It is epxpected that a root level diagnostic page will exist at /diagnostic, this page
should return a 200 under normal operating conditions.
- As described in the section on 'start' the application must have an executable file
in it's root directory called 'start' that accepts a single parameter indicating the
port to which the application should bind to listen for incoming requests.

## Application Instance Routing
  Routing amongst instances managed by approuter is controlled by the presence of specific 
  cookies.  In the absence of an alternate routing cookie, the active production instance
  will serve the request.  To facilitate the setting of routing cookies, alternate specific
  urls are created as follows:

    http://approuter.host.com/set_alt_cookie_<alternate name>.html
    http://approuter.host.com/remove_alt_cookie_<alternate name>.html

    
    e.g.
    http://approuter.host.com/set_alt_cookie_gamma.html
    http://approuter.host.com/remove_alt_cookie_gamma.html


## Identity Managment (IN PROGRESS)

As a convenience approuter provides the ability to map inbound user
identifying cookie data to a different value.  This provides the
theoretical ability to (for instance) map an inbound id (e.g. the value
generated by hashing the user's email and password) to another arbitrary
value (e.g. user id).  Simple interfaces for generating this mappings
exist and are documented below.

### Configuration

In order to enable user identity management in approuter, you must
provide some basic configuration data as follows:

- **PROXY_AUTH** - set to any non empty string value to enable identity
managment.
- **ID_MAP_ROOT** - provides the location on disk where the identity
mapping information is stored.  This should be an existing directory
which may contain identity mapping files.
- **USER_ID_COOKIE** - this is the name of the cookie that will be used to
identify the user.  That is, this is the cookie that will be expected to
be on any inbound request to identify the user, if Identity Managment is
enabled, this cookie will be required.  Inbound requests that do not
contain this cookie (and value) will result in a 401.

