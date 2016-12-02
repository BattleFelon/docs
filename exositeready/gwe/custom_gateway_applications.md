# Custom Gateway Applications

By itself, ExositeReady™ Gateway Engine (GWE) hosts Custom Gateway Applications (CGA) and Gateway Message Queuing server waits for incoming requests from Custom Gateway Applications, so, without a Custom Gateway Application to host, GWE does not really do much. This section is dedicated to defining how Custom Gateway Applications fit into the GWE hosting framework.

The illustration, below, shows the basic layout of a typical IoT gateway from the physical layer up to the cloud.

![communication_topology](/exositeready/gwe/communication_topology.png)

This illustration shows an application written in the context of and hosted by GWE. As shown, the custom application has flexible options as far as how it can communicate on the internet and with Exosite. As a developer of a CGA, you may choose to use the Gateway Message Queue (GMQ) for writing sensor data, Device Client (GDC) for reading configuration data and some other tool like `curl` or another library for some other service (e.g., ntpdate, ping, a nodejs library, etc.).

The Custom Gateway Application is the customized logic the developer writes for a specific IoT solution. A Custom Gateway Application should be designed to be installed via the GWE installer and OTAU feature.

## Get the GWE Development Tools

Navigate to the [Release Packages](release_packages.md) section and download the latest copy of GWE to your development machine.

It is highly recommended that you create a virtual Python environment before installing GWE onto your development machine.

```
pip install virtualenvwrapper
mkvirtualenv gwe-devtools --python=python2.7
```

Unpack the GWE release and install the development tools. If using a virtual python environment, make sure to activate it with `workon gwe-devtools` before proceeding.

```
mkdir ~/sandbox
mv path/to/GatewayEngine/download.tar.gz ~/sandbox
cd ~/sandbox
tar zxvf GatewayEngine.v...tar.gz
cd gateway-engine
python GatewayEngine/cli.py -d DEBUG -I device-client.v*.tar.gz
python setup.py install
```

## Initialize a Repository

The first thing to do is to create a new sandbox folder to put the CGA files. If using a virtual python environment, make sure to activate it with `workon gwe-devtools` before proceeding.

### Project Directory

```
mkdir ~/sandbox/my_gwe_hosted_app
cd ~/sandbox/my_gwe_hosted_app
```

Executing the following command will prompt you for a name for the new buildfile, the name you want to give the new app and the version of the new app. After the buildfile is created, a summary of the buildfile contents are printed to the console. See the example below.

### Create Buildfile

```
$ gwe --create-buildfile
Provide build file name (my_app.json): gwe-buildfile.json
Provide app name (my_app): my_gwe_hosted_app
Provide app version (1): 1
==================================================
==============  Build file created  ==============
==================================================

A bare-bones JSON build file has been created. It
supports the following parameters:

 "name"             -   This is the name Gateway
                        Engine will use as your
                        application's name.

 "version"          -   This is the version of
                        your application. Any
                        time you edit the code
                        or anything in your
                        application, you should
                        increment or otherwise
                        change the version.

 "includes"         -   This is a list of files
                        that you want to include
                        in your application 
                        tarball. Add files to 
                        this list to suit the
                        needs of your applicaiton.

 "excludes"         -   This is a list of files
                        that you don't want in
                        your application tarball.
                        This option is often used
                        when specifying "include"
                        lists with globs like "*".

 "pre_build_cmds"   -   This is a list of system
                        commands you want to run
                        before the tarball is 
                        created. Common uses for
                        this feature are to do
                        "wget" to fetch resources
                        on the internet that you
                        want included in your
                        application tarball.

 "post_build_cmds"  -   This is a list of system
                        commands you want run
                        after the tarball is 
                        created. This is handy 
                        for cleaning up build
                        artifacts after the 
                        build completes.

For more information on Gateway Engine
please visit:

    gateway-engine.exosite.io

Build file location:

    /Users/willcharlton/sandbox/GatewayEngine/gmq-sine-demo/gwe-buildfile.json

==================================================
===========  gatewayengine@exosite.com  ==========
==================================================


```

### Create `install.sh` Script

To get started, create an `install.sh` file GWE can use to install `example.sh` to `/usr/local/bin`. Making sure to use a shebang as the first line of the file and having the correct file mode is important.

```
echo '#!/bin/sh' > install.sh
echo 'echo "Installing example.sh..."' >> install.sh
echo 'cp example.sh /usr/local/bin' >> install.sh
echo 'echo "Installation of example.sh complete.' >> install.sh
chmod +x install.sh
```

### Create the `supervisor.conf` File

This file is used by GWE during installation to determine if this is a long-running, hosted application or if it is just a script to run. By not including a `supervisor.conf` file, the GWE installer will have no way to configure `supervisord` to automatically start the Custom Gateway Application on boot, or to restart it if/when it crashes. Sometimes there are cases in which you want this behavior. Custom Gateway Applications that have no `supervisor.conf` configuration file are effectively **ONE-OFF** applications. Restated, these can still be considered applications, but they execute only once - when GWE runs the `install.sh` script. This can be a handy tool if you want to just send a command to a gateway like `reboot`.

For long-running, hosted CGAs in which a `supervisor.conf` file *is* provided, there are some defaults and constraints to understand before proceeding.

* The file must be an INI-style configuration file with a single `[supervisord]` section.
* Any option specified in the `[supervisord]` will override any default option that GWE provides.

Below is an example of a `supervisor.conf` file that `supervisord` can use to start `/usr/local/bin/example.sh`, keep running, rotate the logs and restart it it crashes.

```
echo '[supervisord]' > supervisor.conf
echo 'command = /bin/sh /usr/local/bin/example.sh' >> supervisor.conf
cat supervisor.conf
```

#### A Note on the Default Supervisor Options

Below are the default options that GWE will impose if the `supervisor.conf` file you provide does not contain them. If your `supervisor.conf` file contains any of these options, then the defaults will be overridden by them.

    ```
    ; This section is mandatory. If a supervisor.conf file doesn't
    ; have this section, it won't be recognized during installation.
    [supervisord]

    ; This tells supervisord to redirect your app STDERR to STDOUT.
    redirect_stderr = true

    ; This tells supervisord to keep the logfile size less than or equal to 200KB.
    stdout_logfile_maxbytes = 200KB

    ; This tells supervisord to keep only 1 backup of the logfile.
    stdout_logfile_backups = 1

    ; This is the command supervisord will use to run/execute your application. 
    ; The default command is "command". This will always fail and is just a placeholder 
    ; for the actual command needed to start your app (e.g. "command": /usr/local/bin/my_app).
    command = command

    ; This is the logfile path prefix supervisord will redirect your STDOUT and STDERR to. 
    ; Gateway Engine will use the name of the tarball to fill in the rest of the logfile name
    ; if you don't specify your own. The default value, below, is just a prefix. 
    stdout_logfile = /var/log
    ```

### Create the Application

Next, create a simple application that you can package up into an OTAU package and deploy to your gateway. A simple example is provided below.

```
cat > example.sh << EOF
#!/bin/sh

while true ; do
    echo 'Running...'
done
EOF
```

### Update the Buildfile

Add the new repository files to the build file so they're included in the OTAU package during the build step. An example of the edited `gwe-buildfile.json` file from a previous step.

```
{
  "excludes": [], 
  "name": "my_gwe_hosted_app", 
  "includes": [
    "install.sh",
    "supervisor.conf",
    "example.sh"
  ], 
  "pre_build_cmds": [], 
  "post_build_cmds": [], 
  "version": "1"
}
```

### Check the Buildfile for Errors

Use the following command to check for errors and issues with the build file.

```
gwe --check-buildfile gwe-buildfile.json
```

With the example buildfile, above, the following output should be seen on your terminal:

    ```
    [PASS]
    ```

## Build the Application

With an application to build and a buildfile, GWE can build an OTAU application with the following command:

```
gwe --build-app gwe-buildfile.json
```

**NOTE**: If you are experiencing errors or problems with installs, you can add more debug output to the build command with the `-d DEBUG` cli flag.

The output of this command will look something like this:

    ```
    /home/user/sandbox/my_gwe_hosted_app/my_gwe_hosted_app.v1.tar.gz
    ```

## Deploy the Application

The following MrMurano commands can be used to upload and deploy the new application a gateway:

```
mr content upload my_gwe_hosted_app.v1.tar.gz my_gwe_hosted_app.v1.tar.gz
mr product device write <SERIAL_NUMBER_OF_GATEWAY> engine_fetch '{"install": [{"name": "my_gwe_hosted_app.v1.tar.gz"}]}'
```

## Verify the Deployment

The STDOUT of the `install.sh` script is written to GWE's `fetch_status` dataport after it completes. Use the following MrMurano command to check on the status of the installation. If there were any errors during the deployment, the will show up here:

```
mr product device read <SERIAL_NUMBER_OF_GATEWAY> fetch_status
```

# A Good Example

If you navigate to the [gmq-sine-demo](https://github.com/exosite/gmq-sine-demo "GMQ Sine Demo") you can see an example of a simple application that GWE can host and install over the air.


