# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# Build and deploy an ASP.NET Core 1.0 web app on Windows Server 2012 R2
# Standard or Docker.
# Author: Robert Howe <rc@rchowe.com>
# Date: 28 January 2016
#

#------------------------------------------------------------------------------
# Virtual machine configuration

Vagrant.configure(2) do |config|
	# We do hostmanager configuration later.
	config.hostmanager.enabled = false
	
	#
	# IIS Web Server
	# Windows Server 2012 R2 Web VM
	config.vm.define "web" do |web|
		web.vm.box = "opentable/win-2012r2-standard-amd64-nocm"
		web.vm.communicator = "winrm"
		web.vm.guest = "windows"
		
		# Networking
		web.vm.network "private_network", ip: "10.2.2.11"
		web.vm.network "forwarded_port", id: "rdp", guest: 3389, host: 3389,  autocorrect: true
		web.vm.network "forwarded_port", id: "ssh", guest: 22,   host: 10022, autocorrect: true
		
		# Provisioning
		web.vm.provision "install-iis",    type: "shell", inline: INSTALL_IIS_SCRIPT,    keep_color: true
		web.vm.provision "install-aspnet", type: "shell", inline: INSTALL_ASPNET_SCRIPT, keep_color: true
		web.vm.provision "build-app",      type: "shell", inline: BUILD_ASPNET_SCRIPT,   keep_color: true, run: "always"
		
		# Set up a hostname alias
		web.hostmanager.aliases = ["web.dev", "windows.web.dev", "clr.windows.web.dev", "coreclr.windows.web.dev"]
		web.hostmanager.ip_resolver = proc { |m| '10.2.2.11' } # Temporary fix until Hostmanager works again.
		web.vm.provision :hostmanager
		
		web.vm.post_up_message = WINDOWS_MESSAGE
	end
	
	#
	# Web Apps behind an NGINX proxy
	# Docker
	config.vm.define "web_docker" do |web_docker|
		web_docker.vm.box = "codekitchen/boot2docker"

		web_docker.vm.hostname = "web-docker"
		web_docker.hostmanager.aliases = ["linux.web.dev", "mono.linux.web.dev", "coreclr.linux.web.dev"]
		web_docker.vm.network "private_network", ip: "10.2.2.21"		
		web_docker.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: ".git"
		web_docker.hostmanager.ip_resolver = proc { |m| '10.2.2.21' } # Temporary fix until Hostmanager works again.
		
		web_docker.vm.provision "docker", images: ["microsoft/aspnet:1.0.0-rc1-update1",
			"microsoft/aspnet:1.0.0-rc1-update1-coreclr", "jwilder/nginx-proxy"]
		web_docker.vm.provision "build-app", type: "shell", run: "always", inline: DOCKER_BUILD_SCRIPT, keep_color: true
		web_docker.vm.provision :hostmanager
		
		web_docker.vm.post_up_message = DOCKER_MESSAGE + VB_DOCKER_MESSAGE
	end
	
	#
	# Global VirtualBox Options
	config.vm.provider "virtualbox" do |vb|
		vb.linked_clone = true
		vb.memory = 1024
	end
	
	#
	# Allow configuring of DNS via /etc/hosts
	config.hostmanager.manage_host = true
	config.hostmanager.ignore_private_ip = true
	config.hostmanager.include_offline = true
end

#------------------------------------------------------------------------------
# Messages

DOCKER_MESSAGE = <<-EOF
*-----------------------------------------------------------------------------*
|                        LINUX/DOCKER VM HAS BOOTED!                          |
|                                                                             |
| You may need to run `gulp min` first and re-provision this VM to make sure  |
| that all of your static assets are properly compiled.                       |
|                                                                             |
| If you make changes to your web app, run the following command;             |
|                                                                             |
|     vagrant provision web_docker --provision-with build-app                 |
|                                                                             |
*-----------------------------------------------------------------------------*
EOF

VB_DOCKER_MESSAGE = <<-EOF
|                       LINUX/DOCKER VM ON VIRTUALBOX                         |
|                                                                             |
| Go to http://linux.web.dev or http://10.2.2.21 to see your web site. You    |
| may have to wait a few minutes for the web site to become available.        |
|                                                                             |
| Additionally, try the following URLs for specific .NET runtimes:            |
|                                                                             |
| * http://coreclr.linux.web.dev/       * http://mono.linux.web.dev/          |
|                                                                             |
*-----------------------------------------------------------------------------*
EOF

AWS_DOCKER_MESSAGE = <<-EOF
|                         LINUX/DOCKER VM ON AMAZON                           |
|                                                                             |
| AWS virtual machines require more configuration.                            |
|                                                                             |
| 1. You need to add a DNS record pointing to this server. You can find this  |
|    server's public IP address in the EC2 console web interface or by using  |
|    the `describe-instance` CLI command.                                     |
|                                                                             |
| 2. You need to add a security group to allow traffic on port 80 to this VM. |
|                                                                             |
| 3. The Mono/Full CLR version will be used by default. To change this, edit  |
|    the Vagrantfile.                                                         |
|                                                                             |
| 4. We suggest that you use Amazon RDS to provide a SQL Server database (if  |
|    necessary). Instructions for launching an instance can be found in the   |
|    README.                                                                  |
|                                                                             |
*-----------------------------------------------------------------------------*
EOF

WINDOWS_MESSAGE = <<-EOF
*-----------------------------------------------------------------------------*
|                          WINDOWS VM HAS BOOTED!                             |
|                                                                             |
| Go to http://web.dev or http://10.2.2.11 to see your web site. You          |
| may have to wait a few minutes for the web site to become available.        |
|                                                                             |
| If you make changes to your web app, run the following command;             |
|                                                                             |
|     vagrant provision web --provision-with build-app                        |
|                                                                             |
| Additionally, try the following URLs for specific .NET runtimes:            |
|                                                                             |
| * http://coreclr.windows.web.dev/     * http://clr.windows.web.dev/         |
|                                                                             |
*-----------------------------------------------------------------------------*
EOF

#------------------------------------------------------------------------------
# Linux Build Scripts

DOCKERFILE_MONO = <<-EOF
FROM microsoft/aspnet:1.0.0-rc1-update1
RUN mkdir /app
COPY project.json ./NuGet.Config* /app/
WORKDIR /app
RUN ["dnu", "restore"]
COPY . /app
EXPOSE 5000/tcp
ENTRYPOINT ["dnx", "-p", "project.json", "web", "--server.urls", "http://*:5000"]
EOF

DOCKERFILE_CORECLR = <<-EOF
FROM microsoft/aspnet:1.0.0-rc1-update1-coreclr
RUN mkdir /app
COPY project.json ./NuGet.Config* /app/
WORKDIR /app
RUN ["dnu", "restore"]
COPY . /app
EXPOSE 5000/tcp
ENTRYPOINT ["dnx", "-p", "project.json", "web", "--server.urls", "http://*:5000"]
EOF

# We need to add a custom proxy.conf file to NGINX to ensure that Connection: keep-alive is properly forwarded.
NGINX_PROXY_CONF_FILE = <<-EOF
# HTTP 1.1 support
proxy_http_version 1.1;
proxy_buffering off;
proxy_set_header Host \$http_host;
proxy_set_header Upgrade \$http_upgrade;
proxy_set_header Connection keep-alive;
proxy_set_header X-Real-IP \$remote_addr;
proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto \$proxy_x_forwarded_proto;
EOF

DOCKER_BUILD_SCRIPT = <<-EOF
# Check for .dockerignore
[ ! -f "/vagrant/.dockerignore" ] && \
	echo "You don't seem to have a .dockerignore file. You can speed up\nbuild times by creating one." 1>&2
			
docker ps -q -f label=com.rchowe.vagrant.killable=true \
	| xargs --no-run-if-empty docker rm -f > /dev/null

cat > /tmp/nginx-proxy.conf <<"ZZPROXYFILE"
#{NGINX_PROXY_CONF_FILE}
ZZPROXYFILE

[ ! -f "/vagrant/Dockerfile.Mono" ] && cat > /vagrant/Dockerfile.Mono <<"ZZMONODOCKERFILE"
#{DOCKERFILE_MONO}
ZZMONODOCKERFILE

[ ! -f "/vagrant/Dockerfile.CoreCLR" ] && cat > /vagrant/Dockerfile.CoreCLR <<"ZZCORECLRDOCKERFILE"
#{DOCKERFILE_CORECLR}
ZZCORECLRDOCKERFILE

echo "\n*** Building Mono version ***"
docker build -t app-mono -f /vagrant/Dockerfile.Mono /vagrant | grep -v "Sending build context to Docker daemon"
if [ -f "/vagrant/Dockerfile.CoreCLR" ]; then
	echo "\n*** Building CoreCLR version ***"
	docker build -t app-coreclr -f /vagrant/Dockerfile.CoreCLR /vagrant \
		| grep -v "Sending build context to Docker daemon"
fi

docker run -d --label com.rchowe.vagrant.killable=true \
	-e DEFAULT_HOST=mono.linux.web.dev -p 80:80 \
	-v /tmp/nginx-proxy.conf:/etc/nginx/proxy.conf:ro \
	-v /var/run/docker.sock:/tmp/docker.sock:ro \
	jwilder/nginx-proxy > /dev/null
docker run -d --label com.rchowe.vagrant.killable=true \
	-e VIRTUAL_HOST=mono.linux.web.dev app-mono > /dev/null
[ -f "/vagrant/Dockerfile.CoreCLR" ] && \
	docker run -d --label com.rchowe.vagrant.killable=true \
		-e VIRTUAL_HOST=coreclr.linux.web.dev app-coreclr > /dev/null
EOF

#------------------------------------------------------------------------------
# Windows Build Scripts

INSTALL_IIS_SCRIPT = <<-'EOF'
$ErrorActionPreference = "Stop"
# Enable IIS in windows configuration.
Write-Host " *** Installing IIS *** "
Import-Module ServerManager | Out-Null
Add-WindowsFeature Web-Server, Web-WebServer, Web-Net-Ext | Out-Null
Add-WindowsFeature Web-Mgmt-Console | Out-Null

# Install HttpPlatformHandler
# Even though it says it's an MSI, it actually gets run like an EXE.
Write-Host " *** Installing HttpPlatformHandler ***"
Invoke-WebRequest 'http://go.microsoft.com/fwlink/?LinkId=690721' -OutFile "$ENV:TEMP\httpPlatformHandler_amd64.msi" | Out-Null
& "$ENV:TEMP\httpPlatformHandler_amd64.msi" /q

# Unlock the handlers section for the HttpPlatformHandler
Set-WebConfiguration //handlers machine/webroot/apphost -Metadata overrideMode -Value Allow

# Remove the default website.
Write-Host " *** Removing the default website ***"
Remove-Website 'Default Web Site' | Out-Null

# Create our website.
Write-Host " *** Creating Application website *** "
New-Website 'clr.windows.web.dev' -PhysicalPath 'C:\Websites\clr.windows.web.dev\wwwroot' -Force | Out-Null
New-Website 'coreclr.windows.web.dev' -PhysicalPath 'C:\Websites\coreclr.windows.web.dev\wwwroot' -Force | Out-Null
Remove-WebBinding

# Create web bindings
New-ItemProperty -Path "IIS:\Sites\clr.windows.web.dev" -Name Bindings -Value @{protocol="http"; bindingInformation="*:80:clr.windows.web.dev"}
New-ItemProperty -Path "IIS:\Sites\clr.windows.web.dev" -Name Bindings -Value @{protocol="http"; bindingInformation="*:80:*"}
New-ItemProperty -Path "IIS:\Sites\coreclr.windows.web.dev" -Name Bindings -Value @{protocol="http"; bindingInformation="*:80:coreclr.windows.web.dev"}
New-Item -Type Directory -Force -Path "C:\Websites\clr.windows.web.dev" | Out-Null
New-Item -Type Directory -Force -Path "C:\Websites\coreclr.windows.web.dev" | Out-Null
EOF

INSTALL_ASPNET_SCRIPT = <<-'EOF'
$ErrorActionPreference = "Stop"

# Install Git, a dependency for some Bower assets.
Write-Host " *** Installing Git ***"
New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\Git_is1" | Out-Null
New-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\Git_is1" `
	-Name "Inno Setup CodeFile: Path Option" -Value "Cmd" -PropertyType "String" -Force | Out-Null
Invoke-WebRequest 'https://github.com/git-for-windows/git/releases/download/v2.7.1.windows.1/Git-2.7.1-64-bit.exe' `
	-OutFile "$ENV:TEMP\git-2.7.1-64-bit.exe" | Out-Null
& "$ENV:TEMP\Git-2.7.1-64-bit.exe" /SILENT

# Install node.js, a dependency for the client-side assets.
Write-Host " *** Installing node.js ***"
Invoke-WebRequest 'https://nodejs.org/download/release/latest/node-v5.5.0-x64.msi' -OutFile "$ENV:TEMP\node-v5.5.0-x64.msi" | Out-Null
msiexec /q /i "$ENV:TEMP\node-v5.5.0-x64.msi"

# Install ASP.NET
Write-Host " *** Installing DNVM ***"
&{Invoke-Expression ((New-Object Net.WebClient).DownloadString('https://dist.asp.net/dnvm/dnvminstall.ps1'))} | Out-Null
cmd /c "C:\Users\vagrant\.dnx\bin\dnvm upgrade -a x64 2>&1" | Out-Null
EOF

BUILD_ASPNET_SCRIPT = <<-'EOF'
$ErrorActionPreference = "Stop"
Stop-Website 'clr.windows.web.dev'
Stop-Website 'coreclr.windows.web.dev'

# Do this to ensure that the web apps are shut down (prevent file conflicts).
iisreset | Out-Null

$SourceDirectory = 'C:\Source'

if ( Test-Path $SourceDirectory ) {
	Remove-Item -Recurse -Force $SourceDirectory
}
Copy-Item -Recurse -Path 'C:\Vagrant' -Destination "$SourceDirectory" | Out-Null

# Install the runtime, if not already present.
cmd /c "dnvm install 1.0.0-rc1-update1 -r clr -a x64 2>&1" | Out-Null
cmd /c "dnvm install 1.0.0-rc1-update1 -r coreclr -a x64 2>&1" | Out-Null

# Build the app
Push-Location $SourceDirectory
	if ( Test-Path "C:\Websites\clr.windows.web.dev" )     { Remove-Item -Force -Recurse -Path "C:\Websites\clr.windows.web.dev" | Out-Null }
	if ( Test-Path "C:\Websites\coreclr.windows.web.dev" ) { Remove-Item -Force -Recurse -Path "C:\Websites\coreclr.windows.web.dev" | Out-Null }
	New-Item -Type Directory -Force -Path "C:\Websites\clr.windows.web.dev" | Out-Null
	New-Item -Type Directory -Force -Path "C:\Websites\coreclr.windows.web.dev" | Out-Null
	& "C:\Program Files\nodejs\npm.cmd" install -g bower gulp | Out-Null
	& "C:\Program Files\nodejs\npm.cmd" install | Out-Null
	dnu restore --quiet | Out-Null
	dnu publish --runtime dnx-clr-win-x64.1.0.0-rc1-update1     --configuration Release --no-source --out "C:\Websites\clr.windows.web.dev"     --quiet | Out-Null
	dnu publish --runtime dnx-coreclr-win-x64.1.0.0-rc1-update1 --configuration Release --no-source --out "C:\Websites\coreclr.windows.web.dev" --quiet | Out-Null
Pop-Location

Start-Website 'clr.windows.web.dev'
Start-Website 'coreclr.windows.web.dev'
EOF
