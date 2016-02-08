ASP.NET Core 1.0 Vagrantfile
============================

Getting Started
---------------

This is a drop-in Vagrantfile for multi-platform testing of an ASP.NET Core 1.0
app. To get started, copy `Vagrantfile` into the root directory of the app and
run the following script to get started:

    vagrant plugin install vagrant-hostmanager
	vagrant up

Then go to http://windows.web.dev and http://linux.web.dev and check out your app.

Overview
--------

The stacks provided are shown below:

| Virtual Machine | Stack 
|-----------------|-------
| `web`           | Windows Server 2012 R2 running IIS with [HttpPlatformHandler](http://www.iis.net/downloads/microsoft/httpplatformhandler).
| `web_docker`    | [boot2docker](http://boot2docker.io) running an NGINX reverse proxy container and Docker containers for the app.

Each virtual machine runs both the full CLR version (via Mono on Linux) and the
CoreCLR version of your app, hosted as different virtual hosts. The full CLR
version is the default when you go to http://windows.web.dev or
http://linux.web.dev, however the CoreCLR versions are available by going to
http://coreclr.windows.web.dev or http://coreclr.linux.web.dev.

The build script on Linux will **NOT** use the Dockerfile from the same
directory of the Vagrantfile: this is because it is necessary to change the
base container to use the CoreCLR version for the CoreCLR container. To make
changes to the  Dockerfiles used to build these versions, create files called
`Dockerfile.Mono` or `Dockerfile.CoreCLR` in the project directory.

Lastly, if you have dependencies based on the ASP.NET MyGet feed, add a custom
`NuGet.Config` file to the project directory, since your global feed will not
be picked up in the virtual machines.

When you make a change to your app and you want to test the change, you can run
`vagrant reload` to update the virtual machines. For a faster experience on the
Windows virtual machine, you can also run
`vagrant provision --provision-with build-app` to build the app without
restarting.

FAQs / Known Issues
-------------------

1. **The Windows machine times out downloading.** It's 3.5 GB. The download may
time-out a few times. If you want to download the box before running
`vagrant up`, you can use the command
`vagrant box add opentable/win-2012r2-standard-amd64-nocm`.

2. **I can't connect to any of the URLs.** Make sure that your `/etc/hosts`
file is being updated by Vagrant, and that you ran
`vagrant plugin install vagrant-hostmanager`.

3. **I keep getting `502 Bad Gateway` on the Docker VM.** Wait a few minutes and
try again. ASP.NET apps take some time to start up.

4. **I'm running `vagrant provision --provision-with build-app` but my changes are not showing up in the Docker VM.**
Currently the only way to update the Docker VM is with `vagrant reload`, since
rsync only runs when the VM is started. Run `touch project.json` before
`vagrant reload` to force Docker to reinstall the NuGet package dependencies.

5. **The Windows VM is taking a long time to boot.** Often, if the VM has been
around for a little while, it will decide to update itself, which takes a long
time. To determine if this is happening, start the VirtualBox GUI and double
click on the VM to bring up what's happening on it.

6. **Can I use another VM provider?** Only VirtualBox is supported right now.
If you add AWS credentials, the Linux VM works well with the Ubuntu 14.04 AMI,
however the Windows VM does not because there is no (simple) way to copy files
to an AWS VM running Windows programatically, and Vagrant won't automatically
install rsync.

7. **My app needs the beta of RC2.** The full runtime string is needed for
`dnu publish` and tools like that; the beta versions often change the runtime
string. There isn't an official RC2 Docker image yet either. When RC2 is
officially released, this `Vagrantfile` will be updated.  

8. **My app needs a SQL Server database.** SQL Server is notoriously annoying
to install in Vagrant and takes up about 10 GB of hard drive space. In the
interest of time to boot, this Vagrantfile does not start a SQL Server virtual
machine, however [there are a few Vagrant boxes which purport to provide it](https://atlas.hashicorp.com/boxes/search?utf8=✓&sort=&provider=&q=sql+server)
that can easily be added with (using `sitestacker/win12-sql14` as an example):
`config.vm.define 'mssql' do |mssql| mssql.vm.box = 'sitestacker/win12-sql14'; end`.
