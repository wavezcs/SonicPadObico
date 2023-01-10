# Obico for Klipper on the Creality Sonic Pad
I managed to get Obico running on the Sonic Pad so below are instructions to get it going. You will need to gain root in order to complete these steps. Please be careful as anytime you are using root, there is a chance you could brick the device, if you can't boot into it. Creality has not provided a process to completely flash the device.


**This guide was written for folks with prior experience using linux. Only follow this guide if you are comfortable editing files, navigating linux filesystems, and can do your own troubleshooting. This worked for me, but I can't guarantee it will work for you. Please make sure you are on the same firmware before running. This guide assumes you have 1 printer. I have not tested this on a Pad connected to multiple printers. I am using a Ender 5 S1 so I can't guarentee this will work for you and other printers.**


>If you do run into some trouble, you can restore the device by running the following command:
>>/usr/share/script/recovery.sh all

This guide was setup for Sonic Pad firmware `V1.0.6.35.154  02 Dec. 2022`

At some point I'll look to automate this into a script. This is all hard coded for now.
The following are the high level steps:

1. SSH and get root
2. Download and configure Obico
3. Install depedencies
4. Link printer
5. Create service

Let's do it!

## 1. SSH and get root

  ### 1.1 Login to Sonic Pad
  ```ssh -oHostKeyAlgorithms=+ssh-rsa creality@<your-ip/hostname>```
  >Password: creality
  
  ### 1.2 Get root
  credit to [smwoodward](https://github.com/smwoodward/Sonic-Pad-Updates/blob/main/root_access/Root) for root access method

  Edit moonraker init to inject root password update
  `vi /usr/share/moonraker/moonraker/components/machine.py`

  After Line 115 enter the following:
  ```await self._execute_cmd("sed -i '/root:$1$kADTkVT0$czwdHve48Tc33myUPXAD/croot:$1$quuqrAVq$XQKBnFkq5J7bJ4AAeJaYg0:19277:0:99999:7:::' /etc/shadow")```

  ### 1.3 Restart your Sonic Pad

  ### 1.4 SSH in as root
  ```ssh -oHostKeyAlgorithms=+ssh-rsa root@<your-ip/hostname>```
  >Password: creality

## 2. Download and configure Obico

  ### 2.1 Get Obico client from github.
  ```
  cd /usr/share
  git clone https://github.com/TheSpaghettiDetective/moonraker-obico.git
  ```

  ### 2.2 Create moonraker-obico.cfg
  ```
  cp moonraker-obico.cfg.sample moonraker-obico.cfg
  ```
  >Edit config you copied and change logging path to: /mnt/UDISK/printer_logs/moonraker-obico.log
  
  >If you are using a local server version of obico, edit the server url



## 3. Install depedencies

  ### 3.1 Setup python virtual environment to prevent version collisons
  ```pip3 install virtualenv
     cd /usr/share/moonraker-obico
     virtualenv env
     source env/bin/activate
  ```
  
  Verify you are running in the virtual environment.
  
  ```which python3```
          
  >Output should be /usr/share/moonraker-obico/env/bin/python3.


  ### 3.2 Install required modules from the obico installation kit
 **Ignore the failure of psutil wheel build. We'll fix that.**
 
 
 ```pip3 install --require-virtualenv -r requirements.txt```

  For some reason, I still got errors from module inclusion despite the above so run the following as well:
  ```
  pip3 install --require-virtualenv requests
  pip3 install --require-virtualenv backoff
  pip3 install --require-virtualenv sentry_sdk
  pip3 install --require-virtualenv bson
  pip3 install --require-virtualenv pathvalidate
  ```


  ### 3.3 Turns out python3-psutil is already installed from opkg. Copy in the psutil module to local env
  ```cp -R /usr/lib/python3.7/site-packages/psutil* /usr/share/moonraker-obico/env/lib/python3.7/site-packages/```
 
 
  At this point obico should be ready to roll. Let's now get a printer connected.


## 4. Link printer

  ### 4.1 Retrieve linking code from obico website (or your own server)
  
  Go to [obico.io](https://app.obico.io/printers/wizard/setup/) and get a code to link your printer.
 
  ### 4.2 Run linking client and enter the code from 4.1
  Execute the link app                  
  ```python3 -m moonraker_obico.link -c /usr/share/moonraker-obico/moonraker-obico.cfg```


  ### 4.3 Run obico
  Before running obico as a service, make sure it runs successfully. Watch for any errors.
  
  Ignore errors on API hit rate limits if using obico.io cloud service
  
  To end obico, use ctrl-c
  
  ```/usr/share/moonraker-obico/env/bin/python3 -m moonraker_obico.app -c /usr/share/moonraker-obico/moonraker-obico.cfg```


## 5. Create service

  Now that everything should be running, lets setup Obico to run as a system service at boot

  ### 5.1 Create procd service
  Edit and put the following in `/etc/init.d/moonraker_obico_service`

  ```
  #!/bin/sh /etc/rc.common

  START=95
  STOP=1
  DEPEND=fstab
  USE_PROCD=1

  start_service() {
      procd_open_instance
      procd_set_param stdout 1
      procd_set_param stderr 1
      procd_set_param env PYTHONPATH=/usr/share/moonraker-obico
      procd_set_param command /usr/share/moonraker-obico/env/bin/python3 -m moonraker_obico.app -c /usr/share/moonraker-obico/moonraker-obico.cfg
      procd_close_instance
  }
  ```
  
  
  ### 5.2 Enable and run service
  ```
  /etc/init.d/moonraker_obico_service enable
  /etc/init.d/moonraker_obico_service start
  ```

there you have it. you should be up and running with obico!
