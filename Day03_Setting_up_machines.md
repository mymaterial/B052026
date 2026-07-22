### Set-up machines

1. Download lab01.zip to any folder.
2. Extract it, will create lab01 folder
4. Open the folder, right click on lab01.vmx file with vmware workstation
5. If asked, choose "I copied it"

### How to clone existing vm?

1. stop existing vm.
2. copy folder lab01 to lab02
3. open lab02 folder, open vmx file with notepad
	-> Edit "Display_name=lab02"
	-> Save and Close
   Again, open vmx file with vmware
4. If asked, choose "I copied it"
5. Since its a cloned copy, change the hostname to lab02
    -> hostnamectl set-hostname lab02
	-> reboot.

`username: root` `password: rac123`

Note: Repeat the process to create more clones.
