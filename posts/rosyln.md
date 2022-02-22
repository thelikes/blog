# Compile .NET Framework / C# in Linux

Quick notes and proof of concept to get up and running with building .net framework projects from linux.

## Install mono-roslyn

Install updated mono, including roslyn:

```
apt install gnupg ca-certificates -y 
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF
echo "deb https://download.mono-project.com/repo/ubuntu stable-bionic main" | sudo tee /etc/apt/sources.list.d/mono-official-stable.list
apt update -y
apt install nuget mono-roslyn -y
```

### Resources
- [Mono Download](https://www.mono-project.com/download/stable/#download-lin)
- [How to install Monodevelop in 20.04 and get it to build something?](https://askubuntu.com/questions/1229982/how-to-install-monodevelop-in-20-04-and-get-it-to-build-something)
- [Build .NET 4.5 on Linux in 5 minutes](https://hudsonmendes.medium.com/build-net-4-5-on-linux-in-5-minutes-and-see-what-it-is-like-848ea45fc667)

## Install .NET

Install .NET 6:

```
mkdir /opt/install-microsoft_prod && cd /opt/install-microsoft_prod
wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
dpkg -i packages-microsoft-prod.deb
apt-get update -y
apt install apt-transport-https -y
apt install dotnet-sdk-6.0 aspnetcore-runtime-6.0 dotnet-runtime-6.0 -y
```

Test:

```
dotnet --list-sdks
dotnet --list-runtimes
```

### Resources
- [How to Install .Net Framework 5 on Ubuntu 20.04 LTS](https://linuxways.net/ubuntu/how-to-install-net-framework-5-on-ubuntu-20-04-lts/)

## PoC

Simple HelloWorld project for testing. Contains no Nugets or other complications.

```
git clone https://github.com/thelikes/HelloWorld
cd HelloWorld
nuget update -self
msbuild -p:Configuration=Release -p:Platform=x64
```