# NI-GPIB-USB-HS-PyVisa-install-on-Raspberry-Pi-4
Tutorial on installing USB drivers for the NI-GPIB-USB-HS and PyVisa onto a Raspberry Pi 4 to enable the use of the odin-gpib adapter

Before starting this tutorial it is assumed you have a fresh raspberry pi OS install, in this case I am using "2022-09-22-raspios-bullseye-arm64" https://downloads.raspberrypi.org/raspios_arm64/images/raspios_arm64-2022-09-26/2022-09-22-raspios-bullseye-arm64.img.xz


## Linux-GPIB-drivers 

### Install dependencies:

```bash
sudo apt-get install tk-dev build-essential texinfo texi2html libcwidget-dev libncurses5-dev libx11-dev binutils-dev bison flex libusb-1.0-0 libusb-dev libmpfr-dev libexpat1-dev tofrodos subversion autoconf automake libtool mercurial
```

### Checkout the LINUX-GPIB to home directory:

```bash
cd ~
svn checkout svn://svn.code.sf.net/p/linux-gpib/code/trunk linux-gpib-code
```

## Use rpi-source to add modules to the kernel:


### Check current kernel version

```bash
uname -r 
```
Expected output:
Example `5.15.74-v8+` 

### Navigate to kernel modules directory

```bash
cd /lib/modules/5.15.74-v8+/
```

### Install dependencies for rpi-source

```bash
sudo apt install git bc bison flex libssl-dev
sudo apt install python2
```

### Download rpi-source script

```bash
Sudo wget https://raw.githubusercontent.com/RPi-Distro/rpi-source/ceee03b5fb9cee65dd933f4784bb455cfa872a76/rpi-source
```

### Use nano to edit source of script

```bash
sudo nano rpi-source
```
Change top line of "#!/usr/bin/env python" to "#!/usr/bin/env python2"

ctrl-x, y, enter to exit and save


### Move the edited file

```bash
sudo mv rpi-source  /usr/local/bin/
```

### Make the file exectuable 

```bash
sudo chmod +x /usr/local/bin/rpi-source && /usr/local/bin/rpi-source -q --tag-update
```

### Run rpi-source

```bash
rpi-source --architecture 1
```

Make sure to include the parameter as this will install the 64 bit version which contains dependent libraries


## Generate keys 

### Move directory

```bash
cd /lib/modules/5.15.74-v8+/build/certs 
```

### Create keygen file

```bash
- sudo tee x509.genkey > /dev/null << 'EOF'
[ req ]
default_bits = 4096
distinguished_name = req_distinguished_name
prompt = no
string_mask = utf8only
x509_extensions = myexts
[ req_distinguished_name ]
CN = Modules
[ myexts ]
basicConstraints=critical,CA:FALSE
keyUsage=digitalSignature
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid
EOF
```

```bash
- cat x509.genkey 
[ req ]
default_bits = 4096
distinguished_name = req_distinguished_name
prompt = no
string_mask = utf8only
x509_extensions = myexts
[ req_distinguished_name ]
CN = Modules
[ myexts ]
basicConstraints=critical,CA:FALSE
keyUsage=digitalSignature
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid
```

```bash
sudo openssl req -new -nodes -utf8 -sha512 -days 36500 -batch -x509 -config x509.genkey -outform DER -out signing_key.x509 -keyout signing_key.pem
```

### Install linux-gpib kernel part

```bash
cd ~/linux-gpib-code/linux-gpib-kernel
make
sudo make install
```


### Install linux-gpib user part

```bash
cd ~/linux-gpib-code/linux-gpib-user
./bootstrap
./configure --sysconfdir=/etc
make
sudo make install

### Check NI-GPIB-USB-HS is detected 

```bash
lsusb | grep GPIB 
```

Expected output: `Bus 001 Device 005: ID 3923:709b National Instruments Corp. GPIB-USB-HS`

### Load kernel module

```bash
sudo modprobe ni_usb_gpib
```

### Update cache for linker

```bash
sudo ldconfig
```

### Check if kernel module loaded

```bash
lsmod | grep gpib
```
Expected outcome: 
```
ni_usb_gpib            36864  0
gpib_common            45056  1 ni_usb_gpib
```

### Add config file

```bash
cd /etc/
```
If gpib.conf is in directory then replace contents with the following:
```
interface {
        minor = 0
        board_type = "ni_usb_b"
        pad = 0
        master = yes
}
```

If gpib.conf is not in directory then `touch gpib.conf` and open with nano `sudo nano gpib.conf`
and add the above contents

### Test GPIB config

```bash
sudo gpib_config
```

Expected outcome: No output, no error

### Test communication with the device:

```bash
sudo ibtest
```

```
- d 
- Enter device address ex 24
- w
- *IDN?
- r 
- 100
```
Expected outcome: ```received string: 'KEITHLEY INSTRUMENTS INC.,MODEL 2410,4110494,C33   Mar 31 2015 09:32:39/A02  /J/K
'```

If you get an output similar to this, then the linux-gpib driver is successfully installed and you can talk to GPIB instruments


## Python 3 dependencies

### Install Python and modules

```bash
sudo apt install python3.9
```

Make sure to install no less than python version 3.9 as this is the lowest version tested to be working with the odin-gpib adapter

```bash 
python
```

Expected outcome:
```
Python 3.9.2 (default, Feb 28 2021, 17:03:44) 
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```

Navigate to a suitable folder location and create a python virtual enviroment 

```bash
cd ~/develop/projects/virtual-envs
```

Create a new venv for the odin-gpib adapter

```bash
python3 -m venv odin-gpib-3.9
```

Load the virtual enviroment

```bash
source odin-gpib-3.9/bin/activate 
```

Git clone odin-control into this directory

```bash
git clone https://github.com/odin-detector/odin-control.git
```

Install odin-control

```bash
cd odin-control 
python setup.py develop
cd ..
```

Making sure you are back in `~/develop/projects/virtual-envs` and git clone the following repo also

```bash
git clone https://github.com/JoshHarris2108/odin-gpib.git
```

Install odin-gpib

```bash
cd odin-gpib
python setup.py develop
cd ..
```

Install pyvisa and gpib-ctypes

```bash
pip3 install pyvisa
pip3 install pyvisa-py
pip3 install gpib-ctypes
```

Performing a `pip3 list` should produce something very similar to the following
```
Package           Version  Location
----------------- -------- ------------------------------------------------------------
future            0.18.2
gpib-ctypes       0.3.0
odin-control      1.3.0    /home/gpib-pi/develop/projects/virtual-envs/odin-control/src
odin-gpib         0.0.0    /home/gpib-pi/develop/projects/virtual-envs/odin-gpib/src
pip               20.3.4
pkg-resources     0.0.0
psutil            5.9.4
PyVISA            1.12.0
PyVISA-py         0.5.3
pyzmq             25.0.0b1
setuptools        44.1.1
tornado           6.2
typing-extensions 4.4.0
```

We can now run a test script to make sure that pyvisa can also talk to the instruments

`~/test.py`
```python
import pyvisa
resources = pyvisa.ResourceManager('@py')
instrument = resources.open_resource('GPIB::X::INSTR')
# Sub X for the address of the instrument
print(instrument.query('*IDN?'))
```

```bash
python3 ~/test.py
```

Expected outcome: `KEITHLEY INSTRUMENTS INC.,MODEL 2410,4110494,C33   Mar 31 2015 09:32:39/A02  /J/K`

### Running the adapter

Navigate to ~/develop/projects/virtual-envs/odin-gpib/test 

```bash
cd ~/develop/projects/virtual-envs/odin-gpib/test
odin_control --config config/gpib_test.cfg --logging=debug --debug_mode=1
```

You may need to edit the gpib_test.cfg to make the server broadcast on the correct IP for your setup

```
[server]
debug_mode = 1
http_port  = 8888
http_addr  = 0.0.0.0
static_path = ./static
adapters   = system_info, gpib
[tornado]
logging = debug

# [adapter.gpib]
# module = odin_gpib.adapter.GpibK2410Adapter
# background_task_enable = 0

[adapter.gpib]
module = odin_gpib.adapter.GpibAdapter
background_task_enable = 0

[adapter.system_info]
module = odin.adapters.system_info.SystemInfoAdapter
```


Example snippet of successful startup (GPIB board errors are to be expected, caused by the pyvisa interrogation of all the GPIB bus lanes, the main thing to look for is that is there are no errors with the boards you have attached, in this case only one board is attached and so it is GPIB board 0, and as the log shows below there are no errors with that board)
```
[D 221108 15:20:57 system_info:37] SystemInfoAdapter loaded
[D 221108 15:20:57 api:145] Registered API adapter class SystemInfoAdapter from module odin.adapters.system_info for path system_info
[D 221108 15:20:57 highlevel:49] SerialSession was not imported No module named 'serial'.
[D 221108 15:20:57 highlevel:56] USBSession and USBRawSession were not imported No module named 'usb'.
[D 221108 15:20:57 highlevel:61] TCPIPSession was correctly imported.
[D 221108 15:20:57 highlevel:68] GPIBSession was correctly imported.
[D 221108 15:20:57 highlevel:201] Created library wrapper for py
[D 221108 15:20:57 highlevel:3019] Created ResourceManager with session 5981893
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 1 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 2 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 3 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 4 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 5 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 6 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 7 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 8 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 9 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 10 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 11 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 12 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 13 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 14 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 15 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 1 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 2 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 3 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 4 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 5 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 6 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 7 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 8 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 9 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 10 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 11 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 12 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 13 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 14 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
libgpib: invalid descriptor
libgpib: invalid descriptor
[D 221108 15:20:57 gpib:76] GPIB board 15 error in _find_boards(): GpibError('ask() error: Iberr 4, bad argument to function call')
[D 221108 15:20:57 resource:274] GPIB0::15::INSTR - opening ...
[D 221108 15:20:57 resource:299] GPIB0::15::INSTR - is open with session 8550876
GPIBInstrument at GPIB0::15::INSTR from adapter.py
[D 221108 15:20:57 gpib:395] GPIB.write b'*IDN?\r\n'
[D 221108 15:20:57 messagebased:436] GPIB0::15::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
[D 221108 15:20:57 adapter:118] model 2510 check
[D 221108 15:20:57 resource:274] GPIB0::24::INSTR - opening ...
[D 221108 15:20:57 resource:299] GPIB0::24::INSTR - is open with session 1875893
GPIBInstrument at GPIB0::24::INSTR from adapter.py
[D 221108 15:20:57 gpib:395] GPIB.write b'*IDN?\r\n'
[D 221108 15:20:57 messagebased:436] GPIB0::24::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
[D 221108 15:20:57 adapter:114] model 2410 check
[D 221108 15:20:57 selector_events:59] Using selector: EpollSelector
[D 221108 15:20:57 adapter:204] LOOPINGTrue
[D 221108 15:20:57 gpib:395] GPIB.write b'*IDN?\r\n'
[D 221108 15:20:57 api:145] Registered API adapter class GpibAdapter from module odin_gpib.adapter for path gpib
[D 221108 15:20:57 default:40] Static path for default handler is ./static
[D 221108 15:20:57 messagebased:436] GPIB0::15::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
[I 221108 15:20:57 main:58] HTTP server listening on 0.0.0.0:8888
KEITHLEY INSTRUMENTS INC.,MODEL 2510,0831353,A08   Sep 11 2001 08:57:44/A02  /?
 Sent from keithley2510.py
GPIBInstrument at GPIB0::15::INSTR from k2510
[D 221108 15:20:57 adapter:204] LOOPINGTrue
GPIBInstrument at GPIB0::24::INSTR inside device.py 1
[D 221108 15:20:57 gpib:395] GPIB.write b':SENS:AVER:STAT?\r\n'
[D 221108 15:20:57 messagebased:436] GPIB0::24::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
GPIBInstrument at GPIB0::24::INSTR inside device.py 1
[D 221108 15:20:57 gpib:395] GPIB.write b':SENS:AVER:COUN?\r\n'
[D 221108 15:20:57 messagebased:436] GPIB0::24::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
GPIBInstrument at GPIB0::24::INSTR inside device.py 1
[D 221108 15:20:57 gpib:395] GPIB.write b':SENS:AVER:TCON?\r\n'
[D 221108 15:20:57 messagebased:436] GPIB0::24::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
GPIBInstrument at GPIB0::24::INSTR inside device.py 1
[D 221108 15:20:57 gpib:395] GPIB.write b':MEAS:VOLT?\r\n'
[D 221108 15:20:57 messagebased:436] GPIB0::24::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
[D 221108 15:20:57 keithley2410:75] Voltage from K2410_24 = 16.00013
GPIBInstrument at GPIB0::24::INSTR inside device.py 1
[D 221108 15:20:57 gpib:395] GPIB.write b':SOUR:VOLT:RANG?\r\n'
[D 221108 15:20:57 messagebased:436] GPIB0::24::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
GPIBInstrument at GPIB0::24::INSTR inside device.py 1
[D 221108 15:20:57 gpib:395] GPIB.write b':SENS:CURR:PROT?\r\n'
[D 221108 15:20:57 messagebased:436] GPIB0::24::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
GPIBInstrument at GPIB0::24::INSTR inside device.py 1
[D 221108 15:20:57 gpib:395] GPIB.write b':MEAS:CURR?\r\n'
[D 221108 15:20:57 messagebased:436] GPIB0::24::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
[D 221108 15:20:58 server:95] 304 GET /api/0.1/gpib/devices/K2410_24/filter (192.168.0.7) 2.30ms
[D 221108 15:20:58 server:95] 200 GET /api/0.1/gpib/devices/K2410_24/voltage (192.168.0.7) 3.35ms
[D 221108 15:20:58 server:95] 200 GET /api/0.1/gpib/devices/K2410_24/current (192.168.0.7) 3.15ms
[D 221108 15:20:58 server:95] 304 GET /api/0.1/gpib/devices/K2410_24/device_control_state (192.168.0.7) 3.27ms
[D 221108 15:20:58 server:95] 304 GET /api/0.1/gpib/devices/K2410_24/filter (192.168.0.7) 1.29ms
[D 221108 15:20:58 server:95] 304 GET /api/0.1/gpib/devices/K2410_24/voltage (192.168.0.7) 1.15ms
[D 221108 15:20:58 server:95] 304 GET /api/0.1/gpib/devices/K2410_24/current (192.168.0.7) 1.17ms
[D 221108 15:20:58 server:95] 304 GET /api/0.1/gpib/devices/K2410_24/device_control_state (192.168.0.7) 1.16ms
[D 221108 15:20:59 adapter:204] LOOPINGTrue
[D 221108 15:20:59 gpib:395] GPIB.write b'*IDN?\r\n'
[D 221108 15:20:59 messagebased:436] GPIB0::15::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
KEITHLEY INSTRUMENTS INC.,MODEL 2510,0831353,A08   Sep 11 2001 08:57:44/A02  /?
 Sent from keithley2510.py
GPIBInstrument at GPIB0::15::INSTR from k2510
[D 221108 15:20:59 adapter:204] LOOPINGTrue
GPIBInstrument at GPIB0::24::INSTR inside device.py 1
[D 221108 15:20:59 gpib:395] GPIB.write b':SENS:AVER:STAT?\r\n'
[D 221108 15:20:59 messagebased:436] GPIB0::24::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
GPIBInstrument at GPIB0::24::INSTR inside device.py 1
[D 221108 15:20:59 gpib:395] GPIB.write b':SENS:AVER:COUN?\r\n'
[D 221108 15:20:59 messagebased:436] GPIB0::24::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
GPIBInstrument at GPIB0::24::INSTR inside device.py 1
[D 221108 15:20:59 gpib:395] GPIB.write b':SENS:AVER:TCON?\r\n'
[D 221108 15:20:59 messagebased:436] GPIB0::24::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
GPIBInstrument at GPIB0::24::INSTR inside device.py 1
[D 221108 15:20:59 gpib:395] GPIB.write b':MEAS:VOLT?\r\n'
[D 221108 15:20:59 messagebased:436] GPIB0::24::INSTR - reading 20480 bytes (last status <StatusCode.success_max_count_read: 1073676294>)
```
Interface loaded and ready to use 
![image](https://user-images.githubusercontent.com/114068472/200605855-9c7bbd7a-c51b-4c52-a0f3-4b3acff41dab.png)


