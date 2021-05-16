Quick Evaluation
================

If you simply want to see how SecureDrop works, from the source and the journalist perspective, please visit the `SecureDrop Demo <https://demo.securedrop.org/>`__

If you are planning to setup SecureDrop for your organization or you are an
aspiring SecureDrop administrator or you are just curious to see what it
takes to setup and run a SecureDrop system you are probably looking for a quick way to have a fully functional SecureDrop system for evaluation purposes without the hassle coming from a physical installation.

Then this page is for you and describes, step by step, all you need to run a virtual SecureDrop system with similar features to a real one.

.. note:: This virtual SecureDrop system should never be used in real life applications under any circumstances. **It's only for evaluation purposes**.

.. warning:: If you think that none, beside you, should know that you have an interest in SecureDrop and that this can put you in danger **DO NOT FOLLOW THIS GUIDE** and get in touch with the SecureDrop team for further help. TODO expand this

.. note:: Developers and contributors should follow the virtualization guides published in the Devepoler Documentation section instead of following this tutorial.


Requirements
------------

You need:

* a Debian based computer. You can use a different distro but this tutorial focuses on Debian
* 16GB of ram
* about 100GB free space
* 2 USB pen drives
* a working gmail account like "g.white@gmail.com" which will be used to send emails
* a working email account like "j.brown@example.co" which will be used as recepient for sent emails
* a valid PGP/GPG keypair and its fingerprint
* an email client already configured to encrypt/decrypt PGP/GPG emails for the email account above
* an android phone with andOTP installed

Before starting
---------------

SecureDrop is an orchestration of physical computers, each with a different role and purpose, configured with specific software and designed to provide a reporting system accessible via Tor network. The final goal is to offer a reliable, secure and anonymous communication channel between sources and journalists.

.. note:: Please read **now** the Overview page to understand all the parts involved and come back here.

.. note:: Please read **now** the Glossary to avoid confusion or misunderstanding and come back here.

This guide assumes that you are working from a Linux computer (HOST) running
Debian stable and will lead to the production of these KVM virtual machines:

* *Admin Workstation*
* *Application Server*
* *Monitor Server*
* *Journalist Workstation*
* *Secure Viewing Station*

**What do they do?**

* The *Admin Workstation* is used to configure the two servers.
* The *Application Server* is where sources are dropping their leaked information (payload)
* The *Monitor Server* checks if any payload has been dropped in the *Application Server* and notifies journalists
* The *Journalist Workstation* downloads the payload from the *Application Server*
* The *Secure Viewing Station* discloses the payload content to the journalist


The installation steps will be:

1. configure your HOST to run KVM and Vagrant (which is to create the two servers)
2. create the two servers
3. create the *Admin Workstation* using Tails operating system
4. configure SSH on *Admin Workstation* and the two servers
5. create the *Secure Viewing Station*
6. install SecureDrop


.. _step1:

Step 1 - Configure your host
----------------------------

This step will configure your computer (HOST) to create, run and manage KVM
virtual machines.

Vagrant is used to automatically deploy the two servers as "half-baked" KVM
virtual machines without the need of manually install the operating system or
manually configure software.

.. code:: sh

   sudo apt-get update
   sudo apt-get install -y vagrant vagrant-libvirt libvirt-daemon-system qemu-kvm virt-manager
   sudo apt-get install -y ansible rsync
   vagrant plugin install vagrant-libvirt
   vagrant plugin install vagrant-mutate
   sudo usermod -a -G libvirt $USER
   sudo systemctl restart libvirtd

Add your localhost user to the kvm group to give it permission to run KVM:

.. code:: sh

   sudo usermod -a -G kvm $USER
   sudo rmmod kvm_intel
   sudo rmmod kvm
   sudo modprobe kvm
   sudo modprobe kvm_intel

Verify that libvirt is installed and that your system supports KVM:

.. code:: sh

   sudo libvirtd --version
   [ `egrep -c 'flags\s*:.*(vmx|svm)' /proc/cpuinfo` -gt 0 ] &&  \
   echo "KVM supported!" || echo "KVM not supported..."

Now set the default Vagrant provider to ``libvirt``

.. code:: sh

   echo 'export VAGRANT_DEFAULT_PROVIDER=libvirt' >> ~/.bashrc
   source ~/.bashrc

.. note:: If you don't want to set the default provider you will need to explicitly pass to ``vagrant`` the parameter ``--provider=libvirt`` whenever you execute it. Example: ``vagrant up --provider=libvirt ...`` . This tutorial assumes that you've set the default provider as described above.

You are all set to download the VirtualBox image for Ubuntu Focal:

.. code:: sh

   vagrant box add --provider virtualbox bento/ubuntu-20.04

and convert it from the ``virtualbox`` to the ``libvirt`` format:

.. code:: sh

   vagrant mutate bento/ubuntu-20.04 libvirt


.. _step2:

Step 2 - Create the servers
----------------------------

Now that your HOST is configured, you can create the two KVM virtual machines which will act as servers.

From your HOST terminal clone the SecureDrop project:

.. code:: sh

  git clone https://github.com/freedomofpress/securedrop.git

Vagrant will read instructions from the ``Vagrantfile`` hosted in the root of the ``securedrop`` folder. Deploy the two KVM servers with this command:

.. code:: sh

  cd securedrop
  vagrant up --no-provision /prod/

This is what you are expected to see:

.. code:: rst

  Bringing machine 'mon-prod' up with 'libvirt' provider...
  Bringing machine 'app-prod' up with 'libvirt' provider...
  ==> app-prod: Checking if box 'bento/ubuntu-20.04' version '202012.23.0' is up to date...
  ==> mon-prod: Checking if box 'bento/ubuntu-20.04' version '202012.23.0' is up to date...
  ==> mon-prod: Creating image (snapshot of base box volume).
  ==> mon-prod: Creating domain with the following settings...
  ==> mon-prod:  -- Name:              securedrop_mon-prod
  ==> mon-prod:  -- Domain type:       kvm
  ==> mon-prod:  -- Cpus:              1
  ==> mon-prod:  -- Feature:           acpi
  ==> mon-prod:  -- Feature:           apic
  ==> mon-prod:  -- Feature:           pae
  ==> mon-prod:  -- Memory:            512M
  ==> mon-prod:  -- Management MAC:
  ==> mon-prod:  -- Loader:
  ==> mon-prod:  -- Nvram:
  ==> mon-prod:  -- Base box:          bento/ubuntu-20.04
  ==> mon-prod:  -- Storage pool:      default
  ==> mon-prod:  -- Image:             /var/lib/libvirt/images/securedrop_mon-prod.img (64G)
  ==> mon-prod:  -- Volume Cache:      default
  ==> mon-prod:  -- Kernel:
  ==> mon-prod:  -- Initrd:
  ==> mon-prod:  -- Graphics Type:     vnc
  ==> mon-prod:  -- Graphics Port:     -1
  ==> mon-prod:  -- Graphics IP:       127.0.0.1
  ==> mon-prod:  -- Graphics Password: Not defined
  ==> mon-prod:  -- Video Type:        virtio
  ==> mon-prod:  -- Video VRAM:        9216
  ==> mon-prod:  -- Sound Type:
  ==> mon-prod:  -- Keymap:            en-us
  ==> mon-prod:  -- TPM Path:
  ==> mon-prod:  -- INPUT:             type=mouse, bus=ps2
  ==> mon-prod: Creating shared folders metadata...
  ==> mon-prod: Starting domain.
  ==> mon-prod: Waiting for domain to get an IP address...
  ==> app-prod: Creating image (snapshot of base box volume).
  ==> app-prod: Creating domain with the following settings...
  ==> app-prod:  -- Name:              securedrop_app-prod
  ==> app-prod:  -- Domain type:       kvm
  ==> app-prod:  -- Cpus:              1
  ==> app-prod:  -- Feature:           acpi
  ==> app-prod:  -- Feature:           apic
  ==> app-prod:  -- Feature:           pae
  ==> app-prod:  -- Memory:            1024M
  ==> app-prod:  -- Management MAC:
  ==> app-prod:  -- Loader:
  ==> app-prod:  -- Nvram:
  ==> app-prod:  -- Base box:          bento/ubuntu-20.04
  ==> app-prod:  -- Storage pool:      default
  ==> app-prod:  -- Image:             /var/lib/libvirt/images/securedrop_app-prod.img (64G)
  ==> app-prod:  -- Volume Cache:      default
  ==> app-prod:  -- Kernel:
  ==> app-prod:  -- Initrd:
  ==> app-prod:  -- Graphics Type:     vnc
  ==> app-prod:  -- Graphics Port:     -1
  ==> app-prod:  -- Graphics IP:       127.0.0.1
  ==> app-prod:  -- Graphics Password: Not defined
  ==> app-prod:  -- Video Type:        virtio
  ==> app-prod:  -- Video VRAM:        9216
  ==> app-prod:  -- Sound Type:
  ==> app-prod:  -- Keymap:            en-us
  ==> app-prod:  -- TPM Path:
  ==> app-prod:  -- INPUT:             type=mouse, bus=ps2
  ==> app-prod: Creating shared folders metadata...
  ==> app-prod: Starting domain.
  ==> app-prod: Waiting for domain to get an IP address...
  ==> mon-prod: Waiting for SSH to become available...
  ==> app-prod: Waiting for SSH to become available...
  ==> mon-prod: Setting hostname...
  ==> app-prod: Setting hostname...
  ==> mon-prod: Configuring and enabling network interfaces...
  ==> app-prod: Configuring and enabling network interfaces...
  ==> mon-prod: Machine not provisioned because `--no-provision` is specified.
  ==> app-prod: Machine not provisioned because `--no-provision` is specified.


Now you can ask Vagrant to return some precious information regarding the new virtual machines.

Ask Vagrant to list the available virtual machines:

.. code:: sh

  vagrant global-status

which outputs something like:

.. code:: rst

  id       name     provider state     directory
  ---------------------------------------------------------------------------
  c61ef02  mon-prod libvirt preparing /home/user/projects/others/securedrop
  e427318  app-prod libvirt preparing /home/user/projects/others/securedrop

As you can see the stase is **preparing**. When vagrant will finish the setup the same command will output:

.. code:: rst

  id       name     provider state   directory
  -------------------------------------------------------------------------
  c61ef02  mon-prod libvirt running /home/user/projects/others/securedrop
  e427318  app-prod libvirt running /home/user/projects/others/securedrop

Notice the change in state to **running**.

Now ask vagrant to display the ssh settings for each virtual machine:

.. code:: sh

  vagrant ssh-config c61ef02

which, in this example, outputs:

.. code:: rst

  Host mon-prod
    HostName 192.168.121.73
    User vagrant
    Port 22
    UserKnownHostsFile /dev/null
    StrictHostKeyChecking no
    PasswordAuthentication no
    IdentityFile /home/user/.vagrant.d/insecure_private_key
    IdentitiesOnly yes
    LogLevel FATAL


.. code:: sh

  vagrant ssh-config e427318

which, in this example, outputs:

.. code:: rst

  Host app-prod
    HostName 192.168.121.204
    User vagrant
    Port 22
    UserKnownHostsFile /dev/null
    StrictHostKeyChecking no
    PasswordAuthentication no
    IdentityFile /home/user/.vagrant.d/insecure_private_key
    IdentitiesOnly yes
    LogLevel FATAL

So,

* the *Monitor Server* has IP: 192.168.121.73
* the *Application Server* has IP: 192.168.121.204

.. note:: As a future reference and **only in case of need/troubleshooting** you can connect to these two virtual machines via ssh from your host with this command: ``ssh -o "IdentitiesOnly=yes" -i ~/.vagrant.d/insecure_private_key vagrant@<vm-ip>``













.. _step3:

Step 3 - Create admin workstation
---------------------------------

The *Admin Workstation* operating system is **Tails** which is a Linux distribution running from a USB drive meant to leave no traces about user's activity.

The *Admin Workstation* is used to install SecureDrop in the two servers and your task, now, is to virtualize it by following meticulously the `official guide <https://tails.boum.org/doc/advanced_topics/virtualization/virt-manager/index.en.html#index4h1>`__ . Please consider the below notes and warnings as fundamental addendum to the official guide.

.. note:: After truncating the Tails .img file, rename it in something meaningful, like ``tails-amd64-4.18_admin_workstation.img```

.. note:: Right before creating the virtual machine, the wizard asks you to name the virtual machine. Choose something meaningful like ``securedrop_admin-workstation``

.. warning:: After choosing "Customize configuration before install" also click on the "Network selection" dropdown list and select **Virtual network 'vagrant-libvirt': NAT**. This forces the 3 virtual machines to be in the same network so that they can "talk" to each other.

.. warning:: When you will specify the operating system from the dropdown list you will find "Debian 10" instead of "Debian Buster".

.. warning:: For decent performance you are advised to allow at least 3 cpus and 4096MB of ram

.. warning:: You will need to reuse the Tails .img file so make a copy of it right after downloading and verifying it.

Quickly the *Admin Workstation* virtual machine will be running and show a "Welcome to Tails!" screen:

* click the "+" button in the "Additional settings" area
* click on "Administration password"
* add an arbitrary password
* eventually set your "Language & Region"
* click on the button "Start Tails"

When you see the desktop, follow the instructions to `configure the Persistent Storage <https://tails.boum.org/doc/first_steps/persistence/index.en.html>`__ which allows to maintain the content of ``~/Persistent/`` after rebooting the virtual machine.

.. warning:: When the Persistence Wizard asks you to "Specify the files that will be saved in the persisent volume" turn all the options ON

Be sure to restart your *Admin Workstation* after you have configured the Persistent Storage and **set again the arbitrary Administration Password** at the "Welcome to Tails!" screen. From now on the password will be remembered because it's saved in the Persistent Storage.

Click "Start Tails" and check if the basic network connectivity works:

* open Tor Browser (**Applications->Favorites->Tor Browser**)
* wait until it loads and click on the "Tor check" green button.
* you should be able to see: *"Congratulations. This browser is configured to use Tor."*.

Test if you can ping the two servers. Open a terminal (*Applications->System Tools->Terminal*):

.. code::

  ping -c 3 192.168.121.73
  ping -c 3 192.168.121.204

Clone the SecureDrop repository in the persistent directory:

.. code:: sh

  cd ~/Persistent/
  git clone https://github.com/freedomofpress/securedrop.git

.. _step4:










Step 4 - Configure SSH
------------------------------------

The two servers you have just created are only running the basic operating system and they lack all the SecureDrop software.

SecureDrop is installed from the *Admin Workstation* via SSH, therefore you need to configure all the 3 virtual machines to allow SSH access.

Start with generating the keypair for the user *amnesia* in *Admin Workstation*. Open a terminal and type:

.. code:: sh

  ssh-keygen -t rsa -b 4096

Press *enter* at each of the 3 prompts (don't set a passphrase). The keypair will be stored in ``~/.ssh/``.

The ssh parameters provided by vagrant about the two servers in Step 2 show that it's not possible to login via SSH using username and password (PasswordAuthentication no). Instead the private key generated by vagrant and included in ``/home/user/.vagrant.d/ insecure_private_key`` (in your HOST) must be used.

The only option you have to copy *amnesia*'s public key to both server is to connect first via the insecure key. So you need to copy it from your HOST to the *Admin Workstation*.

Open a terminal in your HOST:

.. code:: sh

  cat /home/user/.vagrant.d/ insecure_private_key

Select and copy (CTRL+SHIFT+C) the output, then in your *Admin Workstation* terminal type:

.. code:: sh

  gedit ~/insecure_private_key

paste (CTRL+V) in it the key, save it and close the application.

Now check that you are able to login into each server by providing the insecure key:

.. code:: sh

  # monitor server
  ssh -o "IdentitiesOnly=yes" -i ~/insecure_private_key vagrant@192.168.121.73 "echo 'connection to monitor server: OK' && exit"


  # application server
  ssh -o "IdentitiesOnly=yes" -i ~/insecure_private_key vagrant@192.168.121.204
  "echo 'connection to application server: OK' && exit"

You should be able to see:

.. code:: rst

  connection to monitor server: OK
  connection to application server: OK

Thanks to the insecure_private_key you now add *amnesia*'s public key to the authorized keys of each server and disable the insecure key:

.. code:: sh

  cat ~/.ssh/id_rsa.pub | ssh -o "IdentitiesOnly=yes" -i ~/insecure_private_key vagrant@192.168.121.73 "cat > ~/.ssh/authorized_keys"
  cat ~/.ssh/id_rsa.pub | ssh -o "IdentitiesOnly=yes" -i ~/insecure_private_key vagrant@192.168.121.204 "cat > ~/.ssh/authorized_keys"

From now on you can only access the two servers with the keypair you generated, like this:

.. code:: sh

  ssh vagrant@192.168.121.73
  ssh vagrant@192.168.121.204









.. _step5:

Step 5 - Create Secure Viewing
------------------------------

When a document or message is submitted to SecureDrop by a source, it is automatically encrypted with the Submission Key. The private part of this key is only stored on the Secure Viewing Station which is never connected to the Internet. SecureDrop submissions can only be decrypted and read on the Secure Viewing Station.

Similarly to step 3, you are tasked with the creation of a virtualized Tails.

Create a copy and enlarge the original Tails .img file, for example:

.. code:: sh

  cp tails-amd64-4.18.img tails-amd64-4.18_secure-viewing.img
  truncate -s 7200M tails-amd64-4.18_secure-viewing.img

You will use ``tails-amd64-4.18_secure-viewing.img`` as "existing disk image".

Following meticulously the `official guide <https://tails.boum.org/doc/advanced_topics/virtualization/virt-manager/index.en.html#index4h1>`__ . Please consider the below notes and warnings as fundamental addendum to the official guide.

.. note:: Right before creating the virtual machine, the wizard asks you to name the virtual machine. Choose something meaningful like ``securedrop_secure-viewing``

.. warning:: For decent performance you are advised to allow at least 2 cpus and 2048MB of ram

.. warning:: When you will specify the operating system from the dropdown list you will find "Debian 10" instead of "Debian Buster".

.. warning:: After setting "Disk bus: USB" as described in the official Tails guide, right click on the NIC item you see in the left menu and remove it: the *Secure Viewing Workstation* works airgapped. Then "Begin Installation"

When you see the desktop, follow the instructions to `configure the Persistent Storage <https://tails.boum.org/doc/first_steps/persistence/index.en.html>`__ which allows to maintain the content of ``~/Persistent/`` after rebooting the virtual machine.

.. warning:: When the Persistence Wizard asks you to "Specify the files that will be saved in the persisent volume" turn all the options ON

Be sure to restart your *Admin Workstation* after you have configured the Persistent Storage and **set again the arbitrary Administration Password** at the "Welcome to Tails!" screen. From now on the password will be remembered because it's saved in the Persistent Storage.

Now that the *Secure Viewing Workstation* is up and running follow the :doc:`instructions <generate_submission_key>` for generating the Submission Key and come back here.

In order to proceed to step 6, you need to copy the SecureDrop.asc file to one of your  USB pen drive which will then become the *Export Device*.

The *Export Device* is the physical media (e.g., designated USB drive) used to transfer decrypted documents from the Secure Viewing Station to a journalist’s everyday workstation, or to another computer for additional processing.

.. warning:: *Export Device* is particularly important and, in a regular SecureDrop setup, it's protected with strong encryption but for the sake of this tutorial and its evaluation purpose, you will just need a clean regular pendrive.

Label your USB pendrive as "Export" with a sticker and plug it into your HOST pc.

Your target is to connect this pendrive to the *Secure Viewing Workstation* so that you can copy the Submission Key on it.

Open ``virt-manager``. Double-click on ``securedrop_secure-viewing`` and click the blue "i" button in the popup window. Click the "Add Hardware" button, click on "USB Host Device" and selected the pendrive.

From the *Secure Viewing Workstation* click on **Places->Documents**. You should see your pendrive listed on the left menu bar: click on it to mount it.

Navigate to the folder where you have saved the ``SecureDrop.asc`` file. Copy it in the pendrive.

Eject the pendrive by clicking on the "up-arrow" button next to its name in the left menu bar and physically unplug it from your HOST.








.. _step6:

Step 6 - Setup SecureDrop
-------------------------

You are now ready to setup the two servers with SecureDrop.

Open a terminal:

.. code:: sh

  cd ~/Persistent/securedrop

.. code:: sh

  ./securedrop-admin setup

.. note:: If the process gets stuck or something goes wrong, interrupt the process, ``rm -R admin/.venv3`` and launch the process again

.. code:: sh

  ./securedrop-admin update

Connect your *Export Device* to the *Admin Workstation* in the same way you did in step 5 for the *Secure Viewing Workstation*.

Click on **Places->Documents** and copy the ``SecureDrop.asc`` file to ``/home/amnesia/Persistent/securedrop/install_files/ansible-base/``

Find the fingerprint of the GPG keypar mentioned in the requirements at the beginning of this tutorial, open a terminal, download and import its public key from the ubuntu keyserver, for example:

.. code:: sh

  gpg --keyserver hkps://keyserver.ubuntu.com --recv-key "AC1310B9324AB5771D6C23F9404861EC9B405F44"

  gpg --export -a "AC1310B9324AB5771D6C23F9404861EC9B405F44" > /home/amnesia/Persistent/securedrop/install_files/ansible-base/ossec.pub


.. code:: sh

  cd ~/Persistent/securedrop
  ./securedrop-admin sdconfig


You will be asked several questions:

.. code:: rst

  Username for SSH access to the servers: vagrant
  Daily reboot time of the server (24-hour clock): 4
  > Local IPv4 address for the Application Server: 192.168.121.204
  > Local IPv4 address for the Monitor Server: 192.168.121.73
  Hostname for Application Server: app-prod
  Hostname for Monitor Server: mon-prod
  DNS server(s): 8.8.8.8 8.8.4.4
  Local filepath to public key for SecureDrop Application GPG public key: SecureDrop.asc
  Whether HTTPS should be enabled on Source Interface (requires EV cert): no
  > Full fingerprint for the SecureDrop Application GPG Key: 32FF05B3891AF462643D529975AEA3636807A891
  Local filepath to OSSEC alerts GPG public key: ossec.pub
  > Full fingerprint for the OSSEC alerts GPG public key: AC1310B9324AB5771D6C23F9404861EC9B405F44
  Admin email address for receiving OSSEC alerts: j.brown@example.com
  Local filepath to journalist alerts GPG public key (optional):
  SMTP relay for sending OSSEC alerts: smtp.gmail.com
  SMTP port for sending OSSEC alerts: 587
  SASL domain for sending OSSEC alerts: gmail.com
  > SASL username for sending OSSEC alerts: g.white
  > SASL password for sending OSSEC alerts: <g.white-gmail-password>
  Enable SSH over Tor (recommended, disables SSH over LAN). If you respond no, SSH will be available over LAN only
  : yes
  Space separated list of additional locales to support (is ru nl ar es_ES sk tr ro en_US sv zh_Hans de_DE ca hi n
  b_NO zh_Hant it_IT pt_BR fr_FR el cs): en_US
  WARNING: v2 onion services cannot be installed on servers running Ubuntu 20.04. Do you want to enable v2 onion s
  ervices?: no
  Do you want to enable v3 onion services (recommended)?: yes

.. note:: In the list below, the lines starting with ">" should be updated with your setup specific information

.. note:: *Full fingerprint for the SecureDrop Application GPG Key* is the fingerprint of the Submission Key you have created in step 5. Write it with no spaces.

.. code:: sh

  ./securedrop-admin install

.. note:: The sudo password for the ``app-prod`` and ``mon-prod`` servers is by
          default ``vagrant``.

After install you can configure your Admin Workstation to SSH into each VM via:

.. code:: sh

  ./securedrop-admin tailsconfig

This last comments terminates with a message like this:

.. code:: rst

  ok: [localhost] => {
      "msg": "Successfully configured Tor and set up desktop bookmarks for SecureDrop! You will see a notification appear on your screen when Tor is ready.\nThe Journalist Interface's Tor onion URL is: http://zn3gdzvx2j3rwakh3nh9hw51iqsl7tvqxujnsfmifpflah7ezr2xhwid.onion The Source Interfaces's Tor onion URL is: http://4lkgh26axo54hygygjubawy1dybz7thbi3gpayd711jqndb2xdlhxyid.onion "
  }


which means that both the *Source Interface* and the *Journalist Interface* are running as Tor services and displays the URLs needed to access each of them.

.. note:: From now on, when you want to access via SSH the app or the monitor server you'll type ``ssh app` or ``ssh mon`` from the *Admin Workstation*. The SSH connection will be established over the TOR network and might be a bit slow.

In order to be able to connect to the *Journalist Interface* Ms Brown needs to have an account configured so you should create one. Open a terminal from the *Admin Workstation*:

.. code:: sh

  ssh app
  sudo -u www-data bash
  cd /var/www/securedrop
  ./manage.py add-journalist

Add the input information:

.. code:: rst

  Username: j.brown
  First name: Jennifer
  Last name: Brown
  Note: Passwords are now autogenerated.
  This user's password is: sixth repulsive photo lark diligence yapping providing
  Will this user be using a YubiKey [HOTP]? (y/N): n
  User "j.brown" successfully added

  Scan the QR code below with FreeOTP:

Ask Ms Brown to open her andOTP application from her own phone and scan the QR-code displayed in the terminal.

---

.. note:: Write down the user's password

At this point the configuration is completed.

Anyone accessing the *Source Interface* at http://4lkgh26axo54hygygjubawy1dybz7thbi3gpayd711jqndb2xdlhxyid.onion via Tor Browser can send any kind of document and, when that happens, an email notification will be sent to j.brown@example.com.

Ms Brown, thanks to the notification, will know that someones has sent something via SecureDrop and connects to the *Journalist Interface* using a Tails usb pendrive (in this tutorial you will use the *Admin Workstation* for semplicity).

She will then connect to the *Journalist Interface* and login with the password above and the OTP code generated by andOTP (on her phone). From there she will  download the received file(s) on the *Transfer Device* (i.e. another pendrive) which will be used to carry the .gpg file to the *Secure Viewing Workstation*.

From the *Secure Viewing Workstation*, Ms Brown will take the file from the *Transfer Device*, decrypt it via GPG and finally have access to the leaked file.

--

**Final note**

This tutorial is meant to help you hit the ground running and figure out, hands on, how the whole architecture works. However there is much more to know about SecureDrop as you can tell from the extensive documentation present in this website. Hopefully, now that you have the whole picture, it will be easier to navigate the rest of the documentation.