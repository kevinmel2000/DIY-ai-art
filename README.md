This guide goes through all the steps to build an AI-art installation, using a 
Nvidia Jetson Xavier NX and a Samsung The Frame 32". It includes pre-designed CAD-files,
how to set up the computer to run an art-kiosk (with code), how to build and 
assemble the control box and button etc.

## Table of content
1. [Build control box](#build-the-control-box)
    1. [Hand-cut parts](#hand-cut-parts)
    2. [Cut cable slots](#cut-cable-slots)
    3. [Cut wood biscuits holes](#cut-wood-biscuits-holes)
    4. [Glue parts together](#glue-parts-together)
    5. [Spackling paste and sanding](#spackling-paste-and-sanding)
    6. [Add hinges](#add-hinges)
    7. [Add magnetic lock](#add-magnetic-lock)
2. [Build button box]()
    1. ...
    2. ...
3. [Set up Nvidia Jetson Xavier NX]()
    1. ...
    2. ...
4. [Assemble art installation]()
    1. ...
    2. ...

## Set up Nvidia Jetson Xavier NX Dev Kit
The Nvidia Jetson Xavier NX module (https://developer.nvidia.com/embedded/jetson-xavier-nx)
is an embedded system-on-module developed by Nvidia for running computationally demanding tasks
on edge.

The Nvidia Jetson Xavier NX Development Kit 
(https://www.nvidia.com/en-us/autonomous-machines/embedded-systems/jetson-xavier-nx/) is a single-board computer
with an integrated Nvidia Jetson Xavier NX module. Similar to the Raspberry Pi, it has 40 GPIO pins that you can
interact with.

The development kit includes:
* x1 Nvidia Jetson Xavier NX board
* x1 19.0V/2.37A power adapter
* x2 Power cables:
    * Plug type I -> C5
    * Plug type B -> C5
* Quick start / Support guide

![xavier_1](./tutorial_images/setup_computer/xavier_1.jpg)

![xavier_2](./tutorial_images/setup_computer/xavier_2.jpg)

### Install operating system
As Raspberry Pi, Jetson Xavier is using a micro-SD card as its hard drive. As far as I know, there's only one supported
OS image (Ubuntu) provided by Nvidia.

To install the OS, you'll need to use a second computer. 

Start of by downloading the OS image. 
To do this, you need to sign up for a `NVIDIA Developer Program Membership`. It's free and quite useful as you'll get 
access to the Nvidia Developer forum. When you've created the account, you can download the image here: 
https://developer.nvidia.com/jetson-nx-developer-kit-sd-card-image

After you've downloaded it, unzip it. 

To flash the OS image to the micro-SD card, start of by inserting the micro-SD card into the second computer and list 
the available disks. Find the disk name of the micro-SD card you just inserted. In my case, it's `/dev/disk2`:

![xavier_4](./tutorial_images/setup_computer/xavier_4.svg)

When you've found the name of the micro-SD card, un-mount it. Be careful, you definitely don't want to unmount any of 
the other disks!

![xavier_5](./tutorial_images/setup_computer/xavier_5.svg)

Now, change your current directory to where you downloaded and un-zipped the image (usually ~/Downloads).

![xavier_6](./tutorial_images/setup_computer/xavier_6.svg)

To flash the micro-SD card with the OS image, run the command below. Replace `/dev/disk2` with the disk name of your 
micro-SD card and replace `sd-blob.img` with the name of the un-zipped image you downloaded. I've sped up the 
animation, flashing the card usually takes quite a long time (it took ~55 min @ ~4.6 MB/s for me).

![xavier_7](./tutorial_images/setup_computer/xavier_7.svg)

When you're done flashing the micro-SD card with the image, you're ready to boot up the Jetson! Remove the SD-card 
from the second computer and insert it into the Jetson computer. The SD-slot is found under the Xavier NX Module.

![xavier_8](./tutorial_images/setup_computer/xavier_8.jpg)

There's no power button, it will boot when you plug in the power cable. After booting and filling in the initial 
system configuration, you should see the Ubuntu desktop.

![xavier_9](./tutorial_images/setup_computer/xavier_9.gif)

If you get stuck during boot-up with an output as below, try to reboot the machine.
```bash
[ *** ] (1 of 2) A start job is running for End-user configuration after initial OEM installation...
```

Full instruction from Nvidia can be found here: 
https://developer.nvidia.com/embedded/learn/get-started-jetson-xavier-nx-devkit

### Install base dependencies
Before we clone the repository and install the project dependencies, we need to install some base 
dependencies. These are good to install regardless if you're setting up an AI-installation or will
do something else with the Xavier NX Dev Kit.

#### Update and upgrade apt-get
```
sudo apt-get update
sudo apt-get upgrade
```

If asked to choose between `gdm3` and `lightdm`, choose `gdm3`.

Lets reboot the system before we continue:
```bash
sudo reboot
```

#### Install pip
```bash
sudo apt install python3-pip
```

#### Install, create and activate virtual environment
Install virtual environment:
```bash
sudo apt install -y python3-venv
```

Create a virtual environment called `aiart`:
```bash
python3 -m venv ~/venvs/aiart
```

Activate virtual environment:
```bash
source ~/venvs/aiart/bin/activate
```

#### Install python wheel
```bash
pip3 install wheel
```

#### GPIO access
Jetson.GPIO is a Python package that works in the same way as RPi.GPIO, but for the Jetson
family of computers. It enables us to, through Python code, interact with the GPIO pinouts
on the Xavier.

First, install the Jetson.GPIO package into your virtual environment:
```bash
pip3 install Jetson.GPIO
```

Then, we need to set up user permissions to be able to access the GPIOs. Create new GPIO user group (remember to change 
`your_user_name`):

```bash
sudo groupadd -f -r gpio
sudo usermod -a -G gpio your_user_name
```

Copy custom GPIO rules (remember to change `pythonNN` with your Python version):
```bash
sudo cp venvs/aiart/lib/pythonNN/site-packages/Jetson/GPIO/99-gpio.rules /etc/udev/rules.d/
```

#### Install Jetson stats (optional)
[Jetson stats](https://github.com/rbonghi/jetson_stats) is a really nice open-source package to monitor and control the 
Jetson. It enables you to track CPU/GPU usage, check temperatures etc.

To install Jetson stats:
```bash
sudo -H pip install -U jetson-stats
```

You need to reboot the machine before you can use it.

TODO: ADD SVG ANIMATION

### Set up art kiosk
The program running the art kiosk is writting in `Python`. The program is running as 4 different
processes (`Kiosk`, `ArtButton`, `PIRSensorScreensaver` and `GANEventHandler`), seen in the diagram below.

![screen_saver_installation_1](./tutorial_images/install_art_kiosk/art_kiosk_diagram.png)

The `Kiosk` process handles all the GUI, toggling (<F11>) and ending (<Escape>) fullscreen, listens to change
of active artwork to be displayed etc.

The `ArtButton` process listens to a GPIO pinout (defined in config.yaml) connected to a button (see how to solder and
connect the button under #.....). When triggered, it replaces the active artwork (active_artwork.jpg) with a random 
image sampled from the image directory (defined in config.yaml, default `/images`).

The `PIRSensorScreensaver` process listens to a GPIO pinout (defined in config.yaml) connected to a PIR sensor (see how 
to solder and connect the PIR sensor under #.....). When no motion has triggered the PIR sensor within a predefined 
threshold (defined in config.yaml), the computer's screensaver is activated. When motion is detected, it is deactivated.

The `GANEventHandler` process is listening to deleted items in the image directory. When an image is deleted (replacing
the active artwork), the process checks how many images that are left in the image directory. If the number of images 
are below a predefined threshold (defined in config.yaml), a new process is spawned, generating new images using the GAN
network.

#### Clone this repository
```bash
git clone https://github.com/maxvfischer/Arthur.git
```

### Install dependencies
```bash
pip3 install -r requirements.txt
```

### Install xscreensaver
To reduce the risk of burn-in when displaying static art on the screen, a PIR (passive infrared) sensor was integrated. 
When no movement has been registered around the art installation, a screen saver is triggered.

The default screen saver on Ubuntu is `gnome-screensaver`. It's not a screen saver in the traditional sense. Instead of showing moving images, it blanks the screen,
basically shuts down the HDMI signals to the screen, enabling the screen to fall into low energy mode.

The screen used in this project is a Samsung The Frame 32" (2020). When the screen is set to HDMI (1/2) and no HDMI signal is provided, it shows a static image telling the user that no HDMI signal is found. This is what happened when using `gnome-screensaver`. This is a unwanted behaviour in this set up, as we either wants the screen to go blank, or show some kind of a moving image, to reduce the risk of burn-in. We do not want to see a static screen telling us that no hdmi signal is found.

To solve this problem, `xscreensaver` was installed instead. It's an alternative screen saver that allows for moving images. Also, it seems like `xscreensaver's`
blank screen mode works differently than `gnome-screensaver`. When `xscreensaver's` blank screen is triggered, it doesn't seems to shut down the HDMI signal,
but rather turn the screen black. This is the behaviour we want in this installation. 

Follow these steps to uninstall `gnome-screensaver` and install `xscreensaver`:

```bash
sudo apt-get remove gnome-screensaver
sudo apt-get install xscreensaver xscreensaver-data-extra xscreensaver-gl-extra
```
After uninstalling `gnome-screensaver` and installing `xscreensaver`, we need to add it to `Startup Applications` for it to start on boot:

![screen_saver_installation_1](./tutorial_images/setup_computer/screen_saver_installation_1.png)

![screen_saver_installation_2](./tutorial_images/setup_computer/screen_saver_installation_2.png)

Full installation guide: https://askubuntu.com/questions/292995/configure-screensaver-in-ubuntu

### Add AI-model checkpoint
Copy the model checkpoint into `arthur/ml/checkpoint`:

    ├── arthur
         ├── ml
             ├── StyleGAN.model-XXXXXXX.data-00000-of-00001
             ├── StyleGAN.model-XXXXXXX.index
             └── StyleGAN.model-XXXXXXX.meta

### Add initial active artwork
Add an initial active artwork image by copying an image here: `arthur/active_artwork.jpg`

### Adjust config.yaml
The config.yaml contains all the settings.

```
active_artwork_file_path: 'active_artwork.jpg'  # Path and name of active artwork

aiartbutton:
  GPIO_mode: 'BOARD'  # GPIO mode
  GPIO_button: 15  # GPIO pinout used for the button
  image_directory: 'images'  # Directory to copy new images from
  button_sleep: 1.0  # Timeout in seconds after button has been pressed

ml_model:
  batch_size: 1  # Latent batch size used when generating images
  img_size: 1024  # Size of generated image (img_size, img_size)
  test_num: 20  # Number of images generated when model is triggered
  checkpoint_directory: 'ml/checkpoint'  # Checkpoint directory
  image_directory: 'images'  # Output directory of generated images
  lower_limit_num_images: 200  # Trigger model if number of images in image_directory is below this value
```

## Build the control box
To get a nice looking installation with as few visible cables as possible, a control box 
was built to encapsulate the Nvidia computer, power adapters, Samsung One Connect box etc.

### Hand-cut parts
The control box was build using 12mm (0.472") MDF. MDF is quite simple to work with and
looks good when painted. A disadvantage is that it produces very fine-graned
dust when cut or sanded.

![raw_mdf](./tutorial_images/build_control_box/raw_mdf.jpg)

A vertical panel saw was used to cut down the MDF into smaller pieces. A table saw was 
used to cut out the final pieces.

![vertical_panel_saw](./tutorial_images/build_control_box/vertical_panel_saw.jpg)

![table_saw](./tutorial_images/build_control_box/table_saw.jpg)

| Piece              | Dimensions (width, height)    | Sketch                                                                         |
|--------------------|-------------------------------|--------------------------------------------------------------------------------|
| Bottom base panel  | 320mm x 235mm                 | ![table_saw](./tutorial_images/build_control_box/bottom_base_panel_sketch.png) |
| Top lid panel      | 344mm x 259mm                 | ![table_saw](./tutorial_images/build_control_box/top_lid_panel_sketch.png)     |
| Left side panel    | 235mm x 57mm                  | ![table_saw](./tutorial_images/build_control_box/left_side_panel_sketch.png)   |
| Right side panel   | 235mm x 57mm                  | ![table_saw](./tutorial_images/build_control_box/right_side_panel_sketch.png)  |
| Top side panel     | 344mm x 57mm                  | ![table_saw](./tutorial_images/build_control_box/top_side_panel_sketch.png)    |
| Bottom side panel  | 344mm x 57mm                  | ![table_saw](./tutorial_images/build_control_box/bottom_side_panel_sketch.png) |

![raw_pieces](./tutorial_images/build_control_box/raw_pieces.jpg)

![raw_pieces_with_lid](./tutorial_images/build_control_box/raw_pieces_with_lid.jpg)

### Cut cable slots
To enable the cables to go in and out of the box, two cable slots were cut out:

1. One cable slot in the top side panel for the One Connect cable and button cables.
2. One cable slot in the bottom side panel for the electrical cable.

A caliper was used to measure the diameter of the cables. An extra ~1mm was then added to the slots for
the cables to fit nicely.

![caliper](./tutorial_images/build_control_box/caliper.jpg)

The slots were then outlined at the center of the panels.

![cable_slot_1](./tutorial_images/build_control_box/cable_slot_1.jpg)

![cable_slot_2](./tutorial_images/build_control_box/cable_slot_2.jpg)

A jigsaw was used to cut out the slots.

![jigsaw_1](./tutorial_images/build_control_box/jigsaw_1.jpg)

![jigsaw_2](./tutorial_images/build_control_box/jigsaw_2.jpg)

A small chisel and a hammer was used to remove the cut out piece.

![chisel_1](./tutorial_images/build_control_box/chisel_1.jpg)

![chisel_2](./tutorial_images/build_control_box/chisel_2.jpg)

![chisel_3](./tutorial_images/build_control_box/chisel_3.jpg)

![cable_slot_3](./tutorial_images/build_control_box/cable_slot_3.jpg)

### Cut wood biscuits holes
To make the control box robust, wood biscuits were used to glue the parts together. By using wood biscuits, 
no screws were needed, thus giving a nice finish without visible screw heads. It also helps to aligning the
pieces when gluing.

When using the wood biscuit cutter, it's important that the holes end up at the correct place at the 
aligning panels. One simple way of solving this is to align your panels and then draw a line on both 
panels at the center of where you want the biscuit to be. If you do this, the holes will end up at the 
right place.

![wood_biscuit_align](./tutorial_images/build_control_box/wood_biscuit_align.jpg)

![wood_biscuit_machine_1](./tutorial_images/build_control_box/wood_biscuit_machine_1.jpg)

![wood_biscuit_machine_2](./tutorial_images/build_control_box/wood_biscuit_machine_2.jpg)

![wood_biscuit_all_pieces](./tutorial_images/build_control_box/wood_biscuit_all_pieces.jpg)

Before gluing the pieces together, check that the connecting holes are correctly aligning and that all
wood biscuits fit nicely (they can somethings vary a bit in size).

![wood_biscuit_asseble](./tutorial_images/build_control_box/wood_biscuit_asseble.jpg)

### Glue parts together
When gluing the parts together, you'll need to be fairly quick and structured. Prepare by placing the 
aligning panels next to each other and have all the wood biscuits ready.

![gluing_1](./tutorial_images/build_control_box/gluing_1.jpg)

![gluing_2](./tutorial_images/build_control_box/gluing_2.jpg)

Start of by adding the glue in the wood biscuit holes.

![gluing_3](./tutorial_images/build_control_box/gluing_3.jpg)

Press down the wood biscuits into the holes and apply wood glue along all the connecting parts.

![gluing_4](./tutorial_images/build_control_box/gluing_4.jpg)

Now, assemble all the connecting parts together and apply force using clamps. You should see
glue seeping out between the panels.

![gluing_5](./tutorial_images/build_control_box/gluing_5.jpg)

Use an engineer's square to check that you have 90 degrees in each corner of the box.

![gluing_6](./tutorial_images/build_control_box/gluing_6.jpg)

Finally, remove all the visible redundant glue with a wet paper tissue.

![gluing_7](./tutorial_images/build_control_box/gluing_7.jpg)

![gluing_8](./tutorial_images/build_control_box/gluing_8.jpg)

### Spackling paste and sanding
After removing the clamps, there were some visible gaps and cracks that needed to be filled.

![spackling_1](./tutorial_images/build_control_box/spackling_1.jpg)

![spackling_2](./tutorial_images/build_control_box/spackling_2.jpg)

![spackling_3](./tutorial_images/build_control_box/spackling_3.jpg)

I used plastic padding (a two component plastic spackling paste) to cover up the gaps and cracks.

![spackling_4](./tutorial_images/build_control_box/spackling_4.jpg)

Be careful with how much hardener you add, as it will dry very quickly if adding to much.

![spackling_5](./tutorial_images/build_control_box/spackling_5.jpg)

![spackling_6](./tutorial_images/build_control_box/spackling_6.jpg)

![spackling_7](./tutorial_images/build_control_box/spackling_7.jpg)

When everything had dried, an electric sander was used to remove redundant plastic padding.
The inside of the box was smoothed by manual sanding. As a rule of thumb, if you can
feel an edge or a crack, it will be visible when you paint it.

![spackling_8](./tutorial_images/build_control_box/spackling_8.jpg)

![spackling_9](./tutorial_images/build_control_box/spackling_9.jpg)

### Add hinges
The hinges were first added to the lid. It made it easier to align the lid on the box 
later on.

The hinge mortises were measured and outlined. An electric multicutter tool was then used 
to cut out a grid with the same depth as the hinges. The material was then removed using 
a chisel and a hammer. The mortises were then smoothed
by manual sanding.

![hinge_1](./tutorial_images/build_control_box/hinge_1.jpg)

![hinge_2](./tutorial_images/build_control_box/hinge_2.jpg)

![hinge_3](./tutorial_images/build_control_box/hinge_3.jpg)

![hinge_4](./tutorial_images/build_control_box/hinge_4.jpg)

![hinge_5](./tutorial_images/build_control_box/hinge_5.jpg)

![hinge_6](./tutorial_images/build_control_box/hinge_6.jpg)

The hinges were aligned and a bradawl was used to mark the centers of the holes. MDF is a 
very dense material, therefore it's important to pre-drill before screwing the hinges in 
place. If you don't do this, there's a risk that the material will crack.

![hinge_7](./tutorial_images/build_control_box/hinge_7.jpg)

![hinge_8](./tutorial_images/build_control_box/hinge_8.jpg)

The depth of the screws were measured and adhesive tape was used to mark the depth
on the drill head. 

![hinge_9](./tutorial_images/build_control_box/hinge_9.jpg)

![hinge_10](./tutorial_images/build_control_box/hinge_10.jpg)

![hinge_11](./tutorial_images/build_control_box/hinge_11.jpg)

Before aligning the hinges on the box, make sure to add some support under the lid,
it should be able to rest at the same level as the box. Double-coated adhesive tape 
was then attached to each hinge and the lid was aligned on top of the box. When the 
lid was correctly aligned, I applied pressure to make the adhesive tape stick.

![hinge_12](./tutorial_images/build_control_box/hinge_12.jpg)

![hinge_13](./tutorial_images/build_control_box/hinge_13.jpg)

![hinge_14](./tutorial_images/build_control_box/hinge_14.jpg)

The hinge holes and the mortises were drilled and cut out in the same way as on
the lid.

![hinge_15](./tutorial_images/build_control_box/hinge_15.jpg)

![hinge_16](./tutorial_images/build_control_box/hinge_16.jpg)

![hinge_17](./tutorial_images/build_control_box/hinge_17.jpg)

![hinge_18](./tutorial_images/build_control_box/hinge_18.jpg)

![hinge_19](./tutorial_images/build_control_box/hinge_19.jpg)

![hinge_20](./tutorial_images/build_control_box/hinge_20.jpg)

![hinge_21](./tutorial_images/build_control_box/hinge_21.jpg)

![hinge_22](./tutorial_images/build_control_box/hinge_22.jpg)

![hinge_23](./tutorial_images/build_control_box/hinge_23.jpg)

![hinge_24](./tutorial_images/build_control_box/hinge_24.jpg)

### Add magnetic lock
A standard magnetic lock was used to keep the lid in place.

![magnetic_lock_1](./tutorial_images/build_control_box/magnetic_lock_1.jpg)

![magnetic_lock_2](./tutorial_images/build_control_box/magnetic_lock_2.jpg)

![magnetic_lock_3](./tutorial_images/build_control_box/magnetic_lock_3.jpg)

![magnetic_lock_4](./tutorial_images/build_control_box/magnetic_lock_4.jpg)

![magnetic_lock_5](./tutorial_images/build_control_box/magnetic_lock_5.jpg)

![magnetic_lock_6](./tutorial_images/build_control_box/magnetic_lock_6.jpg)

![magnetic_lock_7](./tutorial_images/build_control_box/magnetic_lock_7.jpg)

![magnetic_lock_8](./tutorial_images/build_control_box/magnetic_lock_8.jpg)

![magnetic_lock_9](./tutorial_images/build_control_box/magnetic_lock_9.jpg)

### Milling edges
To give a nice finish, all the edges were milled.

![milling_1](./tutorial_images/build_control_box/milling_1.jpg)

![milling_2](./tutorial_images/build_control_box/milling_2.jpg)

![milling_3](./tutorial_images/build_control_box/milling_3.jpg)
