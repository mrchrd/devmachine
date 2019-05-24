# -*- mode: ruby -*-
# vi: set ft=ruby :

require "yaml"
current_dir = File.dirname(File.expand_path(__FILE__))
config_yaml = YAML.load_file("#{current_dir}/config.yml")
home_disk = "#{current_dir}/home.vmdk"

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.vm.box = config_yaml["vagrant"]["box"]

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # NOTE: This will enable public access to the opened port
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine and only allow access
  # via 127.0.0.1 to disable public access
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  # config.vm.network "private_network", ip: "192.168.33.10"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  config.vm.synced_folder "C:/", "/mnt/c"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  config.vm.provider "virtualbox" do |vb|
    # Display the VirtualBox GUI when booting the machine
    vb.gui = config_yaml["vagrant"]["gui"]

    # Customize the number of CPUs on the VM:
    vb.cpus = config_yaml["vagrant"]["cpus"]

    # Customize the amount of memory on the VM:
    vb.memory = config_yaml["vagrant"]["memory"]

    # Customize the VM:
    vb.customize [ "modifyvm", :id, "--clipboard", "bidirectional" ]
    vb.customize [ "modifyvm", :id, "--draganddrop", "bidirectional" ]
    vb.customize [ "modifyvm", :id, "--monitorcount", config_yaml["vagrant"]["monitors"] ]
    vb.customize [ "modifyvm", :id, "--mouse", "usbtablet" ]
    vb.customize [ "modifyvm", :id, "--rtcuseutc", "on" ]
    vb.customize [ "modifyvm", :id, "--vram", "128" ]
    vb.customize [ "setextradata", :id, "GUI/ShowMiniToolBar", "false" ]
    vb.customize [ "setextradata", :id, "GUI/Fullscreen", config_yaml["vagrant"]["fullscreen"] ]
  end

  # Install plugins
  required_plugins = %w(
    vagrant-disksize
    vagrant-persistent-storage
    vagrant-scp
    vagrant-vbguest
  )
  _retry = false
  required_plugins.each do |plugin|
    unless Vagrant.has_plugin? plugin
      system "vagrant plugin install #{plugin}"
      _retry=true
    end
  end
  if (_retry)
    exec "vagrant " + ARGV.join(' ')
  end

  # Set disk size
  config.disksize.size = "20GB"

  # Add persistent storage
  config.persistent_storage.diskdevice = '/dev/sdc'
  config.persistent_storage.enabled = true
  config.persistent_storage.filesystem = 'xfs'
  config.persistent_storage.location = home_disk
  config.persistent_storage.mountname = config_yaml["user"]["name"]
  config.persistent_storage.mountpoint = '/home/' + config_yaml["user"]["name"]
  config.persistent_storage.size = config_yaml["user"]["home_size"]
  config.persistent_storage.volgroupname = 'home'

  # Install VirtualBox Guest Additions
  config.vbguest.auto_reboot = true
  config.vbguest.auto_update = true

  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  config.vm.provision "upgrade",
    type: "shell",
    run: config_yaml["run"]["upgrade"],
    inline: <<-SHELL
	#!/bin/sh
	export APT_LISTCHANGES_FRONTEND=none
	export DEBIAN_FRONTEND=noninteractive
	export GOPATH=/usr/local
	export LANG=en_US.UTF-8
	apt-get update -y

	# Upgrade APT packages
	apt-get dist-upgrade -y
	apt-get autoremove -y --purge

	# Upgrade Snap packages
 	snap refresh
	eval "$(snap list --all | awk '/disabled/ {print "snap remove",$1,"--revision",$3}')"

	# Update Go Packages
	if which go; then
	  go get -u all
	fi
  SHELL
  config.vm.provision "base",
    type: "shell",
    run: "once",
    env: {
      "VAGRANT_FULLNAME" => config_yaml["user"]["full_name"],
      "VAGRANT_PASSWORD" => config_yaml["user"]["password"],
      "VAGRANT_TIMEZONE" => config_yaml["user"]["timezone"],
      "VAGRANT_USER" => config_yaml["user"]["name"],
    },
    inline: <<-SHELL
	#!/bin/sh
	export APT_LISTCHANGES_FRONTEND=none
	export DEBIAN_FRONTEND=noninteractive
	export LANG=en_US.UTF-8
	apt-get update -y

	# Allow secure APT repositories
	apt-get install -y apt-transport-https

	# Disable multicast DNS
	apt-get autoremove -y --purge avahi-daemon

	# Have .local TLD resolved by DNS servers
	ln -sf ../run/systemd/resolve/resolv.conf /etc/resolv.conf

	# Set timezone
	timedatectl set-timezone "${VAGRANT_TIMEZONE}"

	# Create User
	if ! getent passwd "${VAGRANT_USER}"; then
	  useradd -M \
	    -c "${VAGRANT_FULLNAME}" \
	    -G sudo \
	    -s /bin/bash \
	    "${VAGRANT_USER}"
	  echo "${VAGRANT_USER}:${VAGRANT_PASSWORD}" | chpasswd
	  chown "${VAGRANT_USER}:${VAGRANT_USER}" "/home/${VAGRANT_USER}"
	fi
  SHELL
  config.vm.provision "cli-dev",
    type: "shell",
    run: config_yaml["run"]["cli-dev"],
    env: {
      "VAGRANT_USER" => config_yaml["user"]["name"],
    },
    inline: <<-SHELL
	#!/bin/sh
	export APT_LISTCHANGES_FRONTEND=none
	export DEBIAN_FRONTEND=noninteractive
	export GOPATH=/usr/local
	export LANG=en_US.UTF-8
	apt-get update -y

	# Go
	snap install --classic go

	# AWS CLI
	apt-get install -y awscli
	go get -u -v github.com/kubernetes-sigs/aws-iam-authenticator/cmd/aws-iam-authenticator

	# ChefDK
	echo "deb https://packages.chef.io/repos/apt/stable bionic main" > /etc/apt/sources.list.d/chef.list
	curl https://packages.chef.io/chef.asc | sudo apt-key add -
	apt-get update
	apt-get install -y chefdk

	# CircleCI
	snap install circleci

	# Docker
	apt-get install -y docker.io
	usermod -aG docker vagrant
	usermod -aG docker "${VAGRANT_USER}"

	# gcloud
	echo "deb http://packages.cloud.google.com/apt cloud-sdk-bionic main" > /etc/apt/sources.list.d/google-cloud-sdk.list
	curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
	apt-get update
	apt-get install -y google-cloud-sdk

	# git
	apt-get install -y \
	  git \
	  git-crypt

	# GitHub
	snap install --classic hub

	# Gomplate
	go get -u -v github.com/hairyhenderson/gomplate/cmd/gomplate

	# Helm
	snap install --classic helm

	# json tools
	apt-get install -y jq

	# Kubernetes
	snap install --classic \
	  kubectl \
	  microk8s

	# Terraform
	snap install terraform
	go get -u -v github.com/wata727/tflint

	# Vagrant
	apt-get install -y \
	  vagrant \
	  virtualbox \
	  virtualbox-guest-additions-iso
	vagrant plugin install \
	  vagrant-scp \
	  vagrant-persistent-storage \
	  vagrant-vbguest

	# yaml tools
	apt-get install -y yamllint
  SHELL
  config.vm.provision "desktop",
    type: "shell",
    run: config_yaml["run"]["desktop"],
    inline: <<-SHELL
	#!/bin/sh
	export APT_LISTCHANGES_FRONTEND=none
	export DEBIAN_FRONTEND=noninteractive
	export LANG=en_US.UTF-8
	apt-get update -y

	# Ubuntu Desktop
	apt-get install -y \
	  devilspie \
	  gnome-shell-extensions \
	  gnome-tweaks \
	  materia-gtk-theme \
	  moka-icon-theme \
	  ubuntu-desktop \
	  ubuntu-restricted-extras \
	  virtualbox-guest-dkms
	systemctl start gdm

	# Remove unwanted applications
	apt-get autoremove -y --purge \
	  aisleriot \
	  firefox \
	  gnome-mahjongg \
	  gnome-mines \
	  gnome-sudoku \
	  ubuntu-web-launchers

	# Language Packs
	apt-get install -y \
	  hunspell-fr \
	  hunspell-fr-classical \
	  hyphen-fr \
	  language-pack-fr \
	  language-pack-fr-base \
	  language-pack-gnome-fr \
	  language-pack-gnome-fr-base \
	  libreoffice-help-fr \
	  libreoffice-l10n-fr \
	  mythes-fr \
	  thunderbird-locale-fr

	# GIMP
	snap install gimp

	# Google Chrome
	wget -q https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
	dpkg -i google-chrome-stable_current_amd64.deb
	rm -f google-chrome-stable_current_amd64.deb
	apt-get install -y chrome-gnome-shell

	# Inkscape
	snap install inkscape

	# Modem Manager
	apt-get install -y modem-manager-gui

	# Pidgin
	apt-get install -y pidgin

	# Spotify
	snap install spotify

	# Thunderbird
	apt-get install -y xul-ext-lightning

	# VLC
	snap install vlc
  SHELL
  config.vm.provision "desktop-dev",
    type: "shell",
    run: config_yaml["run"]["desktop-dev"],
    inline: <<-SHELL
	#!/bin/sh
	export APT_LISTCHANGES_FRONTEND=none
	export DEBIAN_FRONTEND=noninteractive
	export LANG=en_US.UTF-8
	apt-get update -y

	# Insomnia
	snap install insomnia

	# Postman
	snap install postman

	# Slack
	snap install --classic slack

	# Visual Studio Code
	snap install --classic code
  SHELL
  config.vm.provision "userinit",
    type: "shell",
    run: config_yaml["run"]["userinit"],
    env: {
      "VAGRANT_EMAIL" => config_yaml["user"]["email"],
      "VAGRANT_FULLNAME" => config_yaml["user"]["full_name"],
      "VAGRANT_LOCALE" => config_yaml["user"]["locale"],
      "VAGRANT_PASSWORD" => config_yaml["user"]["password"],
    },
    privileged: false,
    inline: <<-SHELL
	#!/bin/sh

	# Gnome Settings
	if which gsettings; then
	  dbus-launch gsettings set org.gnome.desktop.background picture-uri 'file:///usr/share/backgrounds/Manhattan_Sunset_by_Giacomo_Ferroni.jpg'
	  dbus-launch gsettings set org.gnome.desktop.input-sources sources "[('xkb', 'ca'), ('xkb', 'us')]"
	  dbus-launch gsettings set org.gnome.desktop.interface clock-show-date true
	  dbus-launch gsettings set org.gnome.desktop.interface gtk-theme 'Materia-compact'
	  dbus-launch gsettings set org.gnome.desktop.interface icon-theme 'Moka'
	  dbus-launch gsettings set org.gnome.desktop.peripherals.touchpad speed -0.4
	  dbus-launch gsettings set org.gnome.desktop.peripherals.touchpad tap-to-click false
	  dbus-launch gsettings set org.gnome.desktop.peripherals.touchpad two-finger-scrolling-enabled false
	  dbus-launch gsettings set org.gnome.desktop.screensaver picture-uri 'file:///usr/share/backgrounds/Manhattan_Sunset_by_Giacomo_Ferroni.jpg'
	  dbus-launch gsettings set org.gnome.mutter dynamic-workspaces false
	  dbus-launch gsettings set org.gnome.mutter workspaces-only-on-primary false
	  dbus-launch gsettings set org.gnome.settings-daemon.plugins.color night-light-enabled true
	  dbus-launch gsettings set org.gnome.settings-daemon.plugins.xsettings overrides "{'Gtk/ShellShowsAppMenu': <0>}"
	  dbus-launch gsettings set org.gnome.shell enabled-extensions "['user-theme@gnome-shell-extensions.gcampax.github.com']"
	  dbus-launch gsettings set org.gnome.shell.extensions.user-theme name 'Materia-transparent-compact'
	  dbus-launch gsettings set org.gnome.system.locale region "${VAGRANT_LOCALE}"
	  dconf write /org/gnome/terminal/legacy/profiles:/:$(gsettings get org.gnome.Terminal.ProfilesList default | tr -d \\')/background-color "'rgb(0,0,0)'"
	  dconf write /org/gnome/terminal/legacy/profiles:/:$(gsettings get org.gnome.Terminal.ProfilesList default | tr -d \\')/foreground-color "'rgb(170,170,170)'"
	  dconf write /org/gnome/terminal/legacy/profiles:/:$(gsettings get org.gnome.Terminal.ProfilesList default | tr -d \\')/use-theme-colors "false"
	fi

	# GPG Key
	if ! gpg -k "${VAGRANT_EMAIL}"; then
	  gpg \
	    --batch \
	    --gen-key \
	    <<-EOF
	      Key-Type: RSA
	      Key-Length: 4096
	      Subkey-Type: RSA
	      Subkey-Length: 4096
	      Name-Real: "${VAGRANT_FULLNAME}"
	      Name-Email: "${VAGRANT_EMAIL}"
	      Passphrase: "${VAGRANT_PASSWORD}"
	      Expire-Date: 0
	EOF
	fi

	# SSH Key
	if [ ! -f ~/.ssh/id_rsa ]; then
	  ssh-keygen \
	    -b 4096 \
	    -C "${VAGRANT_EMAIL}" \
	    -f ~/.ssh/id_rsa \
	    -N "${VAGRANT_PASSWORD}" \
	    -t rsa
	fi

	# Visual Studio Code Settings
	if which code; then
	  code --install-extension humao.rest-client
	  code --install-extension mauve.terraform
	  code --install-extension ms-vscode.Go
	  code --install-extension redhat.vscode-yaml
	  code --install-extension vscodevim.vim
	fi
  SHELL
end
