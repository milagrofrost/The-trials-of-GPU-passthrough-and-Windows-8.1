Extracting this out of VMware communities, just in case.
https://community.broadcom.com/vmware-cloud-foundation/viewdocument/the-trials-of-gpu-passthrough-and-w?CommunityKey=185ca20b-b56d-41b3-a4b8-aad095f0d970&tab=librarydocuments

The trials of GPU passthrough and Windows 8.1 (using ESXI 5.5 and View 5.3) 
 
Jan 16, 2014 03:56 AM

milagrofrost

This is documentation for a project I'm doing right now.  The project is to create a gaming VM and using View with vDGA graphics to stream these games to any capable PC/thin client.  It comes in four pieces:

Modding a GT 640 to become a Grid K1
Enabling passthrough
Creating the View Pool
Getting Windows 8.1 to work with the passthrough GPU
Modding a GT 640 to become a Grid K1

--------------------------------------------------------------------------

When you passthrough a GPU to your VM, you're able to utilize fully dedicated graphics (vDGA) on your VM.  Only problem is that the supported GPUs are pretty expensive with the cheapest card being the Quadro 4000. You might be able to get that card for $300 if you're lucky.  Even then, it could be a little more power than you need.  The GRID K1s are super expensive and have a ton of power, which is an absolute over-kill for a single user/VM.  With that in mind, we can obtain a (lesser-powered) GRID K1 by modding a much cheaper GT 640 (which go for ~$75). 

References for the GFX card mod:

- http://blog.vmpress.org/2013/08/vmware-view.html?showComment=1387510439973 (Google translate that) I used this guide the most

- http://www.eevblog.com/forum/chat/hacking-nvidia-cards-into-their-professional-counterparts/ (the original discovery)

This mod requires some soldering skills.  For the GT 640 you have to remove 2 resistors and add 2 different ones in.  I won't go into great detail about it as the first link mentioned above gives pretty good detail about it.  The replacement resistors can be found at element14.com.  Here are the two resistors for this mod:

40k resistors
15k resistors
These are 0402, surface mount, thick film resistors.  I'd recommend buying at least 5 of each as you might lose one eventually when unpackaging.

When unsoldering and soldering the resistors, make sure you don't apply too much heat as you could ruin the GFX card.  This video gives you a basic idea of how to solder these tiny resistors.  When the soldering job is complete I would test the card on a regular desktop to confirm that the card is now being seen as a GRID K1 and not a GT 640 (which requires installing the NVIDIA drivers).

Once you've confirmed the card is now modded, you can then install the GFX card on your ESXi host. 



Enabling Passthrough

-------------------------------------------

Read this VMware article on vDGA and vSGA.  Page 18 is where the vDGA install process begins, but feel free to read the differences on vDGA and vSGA in the intro.

About the vDGA GPU passthrough installation process on page 18:

It mentions running the command on your ESXi host *esxcfg-module –l | grep <module_name>* to make sure VT-d is enabled.  My HP ML150 G6 had VT-d enabled, but the VT-d module wasn't running, but the install still worked.  So don't freak out if this happens to you as well.
Follow the other steps down to a T, especially things like memory reservation and allowing over 2 GB of RAM (if you've configured that on your VM).
When it says install the Nvidia driver, you'll want to download this for ESXi 5.5.  Extract the downloaded zip and copy the *bundle.zip to your ESXi host.
Then run the command *Esxcli software vib install –d [the bundle zip file]*   Then wait a while.  Don't get impatient.
You can append *--maintenance-mode* if you don't want to put your host in, you guessed it, maintenance mode.
Here's a couple of screenshots of what you're supposed to be seeing in the web client once you've completed the vDGA article.








Creating the View Pool

-------------------------------------------

On to setting up the manual pool for this VM.  This is all screenshots!









You should choose 1 for Max number of monitors.  Supposedly, passthrough GPUs don't support 2.  Do not enable 3D rendering, that's setting does not apply to your graphics card.  Don't be fooled!





Don't forget to entitle yourself!  You deserve it.



Pool has been created, so lets move on to the next arduous task.



Getting Windows 8.1 to work with the passthrough GPU

----------------------------------------------------------------------------------------------------------

This was (and still is) a pain in the butt.

Let me start by describing the process to passthrough GPU for Windows 7.

Log into VM with View client
The VM has two display adapters installed
the SVGA adapter (installed by View Agent)
and the passthrough GPU (for my project it's the GRID K1)
By default the VM boots with the SVGA adapter as the main display
No bueno.  We want the hardware GPU.
Each time your VM reboots you have to fix the display adapters by:
going into devmgmt and disabling the SVGA adapter
When SVGA is disabled, the passthrough GPU kicks in and runs normally
Sometimes though, when your VM reboots, the SVGA adapter is disabled, but the Passthrough GPU never kicks in, but you still have the desktop screen.  You're basically in display adapter limbo.
you have to re-enable SVGA again (this might require a reboot)
then disable the SVGA adapter in order for the GPU to kick in
That's annoying
So to remedy this, I created a local group policy and utilized the "devcon" program for startup and shutdown to fix this annoyance
Startup script uses "devcon [dev&venID for SVGA adapter] disable"
Shutdown script uses "devcon [dev&venID for SVGA adapter] enable"
this workaround worked wonderfully
But then I got curious and wanted to do all of this on Windows 8.1
Oh man.

Process for Windows 8.1.  It hasn't been perfected yet.

Log into your VM with View Client
Disabling SVGA adapter in devmgmt does nothing
You can also disable your passthrough GPU as well and your screen does not go blank.
why is that?  both of your display adapters are disabled, how can you still have video?
WARP and Microsoft's Basic Display Driver using DWM.exe
WARP, AFAIK, basically allows you to have graphics without a GPU and just utilizing the CPU to do all the work.
In Windows 8, WARP is causing issues in getting a passthrough GFX card to enable.
If you disable the SVGA adapter, Windows 8 assumes there are no other active display adapters available and enables it's Basic Display Driver.
The only way AFAIK to get the passthrough GPU to show as an active video card is to plug a separate monitor into the card itself.  Once you plug in a monitor the GFX card immediately enables in Windows 8.1
But this causes a number of issues.
The monitor you use for View Client has to have the same resolution capabilities as the monitor plugged into the GFX card in the ESXi server.  What a waste of resources!  Setting up two mirrored monitors n order to see one.
You can use 2 different monitors, but you'll run into full screen issues when streaming video games.  Plus the image quality on the view client side looks a little worse.
So far the only solution I can come up with is to buy a VDI EDID adapter.  This would plug into the passthrough GFX card and allow you to customize the screen resolution to whatever you desire.
The next plan is to get Zero Client and test with that.  Considering the complications of this whole project I have been successful with Windows 7 and 8.1 in streaming PC games with View Client.  The end result is like using Onlive or Steambox, just with ESXi.  Sorry for the terseness in this guide.  I'm really just trying to compliment the guides that are already out here and filling a few holes that I haven't seen filled when it comes to passthrough complications.
