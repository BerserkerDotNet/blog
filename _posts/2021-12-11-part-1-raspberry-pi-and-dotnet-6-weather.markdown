---
layout: post
title: Part 1. Building Home Weather Station with .NET 6 and Raspberry Pi 4. The Setup
date: '2021-12-25T12:00:00.000-07:00'
author: Andrii Snihyr
tags:
- .Net
- C#
- dotnet
- Pi
- Raspberry Pi
- sensors
- SunFunder
- IoT
modified_time: '2020-05-03T12:00:00.000-07:00'
---
I was looking for a fun project to do this holiday season and thought that it's about time to put my [Raspberry Pi][Pi] to good use. I've decided to build a device that will measure temperature, humidity, and barometric pressure in an apartment. Then, send this data to Azure and have an app written in .NET MAUI to view this data.
I'll split this in the series of articles on how to do it with .NET 6 and Raspberry Pi 4. In this part, I'll describe the setup of Raspberry Pi for .NET 6 development and write a basic "Hello World" app. Let's get started!
<!--more-->
### Goals

1. Setup Raspberry Pi to work from VS Code on Windows.
1. Install .NET 6 on Raspberry Pi.
1. Write an app that will control a simple diode.

### Components

1. [Raspberry Pi 4][Pi] with Raspbian OS installed and ready to be used. See ["Installing the Operating System"][PiImager] for details. Version 3 will also work as the GPIO pins are compatible between 4 and 3.
1. LED diode with pins to connect to the Pi. I'm using [Dual-color LED module][LED] from [SunFounder modules kit][SunFounder].
1. [Breadboard][BB] to connect the diode to Pi. [SunFounder modules kit][SunFounder] includes both the Breadboard[BB] and a [GPIO Breakout Expansion Board][PiExt] to connect to Pi. GPIO board is optional and can be replaced by jumper wires.
1. Jumper wires.

### Connect Remotely to Raspberry Pi From VS Code

Visual Studio Code is a great lightweight editor that has a plugin for pretty much anything, and it has an integrated terminal that can be used to communicate with Pi. There are a few prerequisites for VS Code to remotely connect to Pi.

First, enable SSH on Pi. For that, log in to the Pi directly and go to "Preferences -> Pi Configuration" in the main menu. Switch to "Interfaces" section and make sure that SSH is enabled. For detailed instructions refer to ["Setting up an SSH Server"][PiSSH]

Second, install [Remote SSH][RemoteSSH] extension for VS Code that enables remote development. When installed, a remote connection can be established using Pi's IP address. For detailed instructions see the "Getting Started" section of [Remote SSH][RemoteSSH].

Make sure that everything works as expected by opening a Linux terminal in VS Code instance that is remotely connected to Pi and issuing a `whoami` command. It should output `pi`.

### Install .NET 6 on Pi
To compile and run .NET apps on the Raspberry Pi a .NET SDK has to be installed, as it contains all the tools necessary to write an app.
Since Raspberry Pi CPU architecture is ARM, a version of SDK has to be compatible with [ARM architecture][ARM]. Fortunately, the official .NET SDK download page lists ARM32 and ARM64 versions of SDK available. I'm using a 32-bit version of the Raspbian OS, so I'd need to install a 32-bit version of SDK. Choose the appropriate version based on your OS.

SDK is shipped as the TAR package compressed in a GZip archive (`tar.gz`). It needs to be downloaded, extracted into a folder, and added to the PATH environment variable. Let's go through each step.

First, open a terminal with a remote SSH connection to the Pi and issue a `wget` command to download the SDK package. 
```bash
wget https://download.visualstudio.microsoft.com/download/pr/72888385-910d-4ef3-bae2-c08c28e42af0/59be90572fdcc10766f1baf5ac39529a/dotnet-sdk-6.0.101-linux-arm.tar.gz -P $HOME

```
Parameter `-P $HOME` indicates that the file will be downloaded into the user's home directory, which is typically `/home/pi/` for Raspberry Pi.

Second, extract the file using a combination of `mkdir` and `tar` commands.
```bash
mkdir -p $HOME/dotnet && tar zxf dotnet-sdk-6.0.101-linux-arm.tar.gz -C $HOME/dotnet
```
The above commands create a folder named dotnet in the user home directory and extract the SDK archive into that folder. To verify that SDK is extracted, navigate into that folder using `cd $HOME/dotnet` and run the `ls` command to list the content of the folder. It should be similar to:
![dotnet folder content]({{ "images/pi-weather-station-part-1/contents-of-dotnet.png" | relative_url }} "Contents of dotnet SDK folder")

Third, make .NET SDK available on the Bash. Add the following lines to the end of the `~/.bashrc` file.
```bash
export DOTNET_ROOT=$HOME/dotnet
export PATH=$PATH:$HOME/dotnet
```
Open `~/.bashrc` file from the terminal via `nano ~/.bashrc` or from VS Code directly (my preferred option) by using "File" -> "Open File" menu and navigating to the user home folder.
Save the file and restart the Pi by issuing `sudo reboot` command.

After reboot, verify that dotnet SDK is properly installed and available on the terminal by running `dotnet --info` command. It should output something similar to:

![dotnet info output]({{ "images/pi-weather-station-part-1/dotnet-info.png" | relative_url }} "dotnet info output")

That is it, now all is ready to write and run .NET apps on the Pi just as on any other platform.

### Setting Up the Breadboard

For the diode to work, it needs to be connected to the Pi. There are multiple ways to do this. In the following example I'll be using a breadboard.
Insert the [GPIO Breakout Expansion Board][PiExt] into the breadboard and connect it to the Pi via a ribbon cable.
The result looks like the following:

![Pi connected to the breadboard]({{ "images/pi-weather-station-part-1/breadboard_and_pi.png" | relative_url }} "Pi connected to the breadboard")

Next, connect power and ground (GND) to the breadboard. On the breadboard, the powerline is marked as `+` (pink-ish line) and the ground is marked as `-` (blue-ish line).
Connect them by using two short male-to-male jumper wires. I'll use red wire for the powerline and black wire for GND. The red wire goes into the 3V3 port, and GND goes into the GND port.

![Breadboard power]({{ "images/pi-weather-station-part-1/breadboard_power.png" | relative_url }} "Breadboard power")

Now, the diode from the [SunFounder][SunFounder] kit has 3 wires coming out of it: one for red color (marked R), one for green color (marked G), and the ground (marked GND). `R` and `G` wires will go into GPIO 17 and 18 respectively, GND will go to the ground line on the board. Here is how it looks when connected:

![Connecting diode]({{ "images/pi-weather-station-part-1/connect_diode.png" | relative_url }} "Connecting diode to breadboard")

Note: GPIO pins 17 and 18 are chosen randomly, so you can use any GPIO pins to connect the diode.

The breadboard setup is complete. Time to write some code.

### "Hello World" from .NET on Raspberry Pi
Let's create a simple console app that will make a diode blink "Hello World" in morse code. This is purely for demonstration purposes to see that our setup works. Part 2 will demonstrate a real app to collect temperature, pressure, and humidity.

Create an app named `HelloPi` using the `console` template by issuing the following command in the terminal.
```bash
dotnet new console -o HelloPi
cd HelloPi
```
Let's see if it runs by executing `dotnet run` in the `~/HelloPi` folder. "Hello, World!" should be printed in the console.

#### Let There Be Light

Since VS Code is already connected remotely to Pi, open newly created project by going to "File" -> "Open Folder". Then open the `HelloPi` folder.
The LED is connected to GPIO pins, so `System.Device.Gpio` library should be used to work with GPIO pins from .NET. Add the following item group to the `csproj` file to include the `System.Device.Gpio` library to the project.
<p>
<script src="https://gist.github.com/BerserkerDotNet/7893110f03a203700965e538eba1bf1f.js"></script>
    <noscript>
        <pre>
  <ItemGroup>
    <PackageReference Include="System.Device.Gpio" Version="1.5.0" />
  </ItemGroup>
        </pre>
    </noscript>
</p>

`System.Device.Gpio` exposes an API called `GpioController` that allows working with GPIO pins. Here is a sample of how to open a GPIO pin and write a signal to it.

<p>
<script src="https://gist.github.com/BerserkerDotNet/a91c02b0a164c223cee9730b64aa99a1.js"></script>
    <noscript>
        <pre>
using System.Device.Gpio;

const int timeUnit = 500;
const int greenPin = 18;

using GpioController controller = new();
controller.OpenPin(greenPin, PinMode.Output);
controller.Write(greenPin, PinValue.High);
Thread.Sleep(timeUnit);
controller.Write(greenPin, PinValue.Low);
controller.ClosePin(greenPin);
        </pre>
    </noscript>
</p>

The code above includes the GPIO namespace, instantiates an instance of the `GpioController`, opens a green pin 18 for a write operation, writes a high value to it to make the diode glow, waits for 500ms, and writes a low signal to turn off the diode. Finally, the pin is closed and the program exits.

#### Morse Code "Hello World"
To make it more fun then just blink a diode on a timer, let's encode "Hello World" into diode blinking.

The rules for Morse Code are as follows:
1. The length of a dot is 1 time unit.
1. A dash is 3 time units.
1. The space between symbols (dots and dashes) of the same letter is 1 time unit.
1. The space between letters is 3 time units.
1. The space between words is 7 time units.

The full alphabet of the international Morse Code can be found on [Wikipedia][MorseCode].
The `Hello World` in the Morse Code is represented as `.... . .-.. .-.. ---/.-- --- .-. .-.. -..`, where `/` signifies the end of the word.
For our diode example it means we need to define a unit time and switch the diode on/off based on the rules above.

When put together, the final code looks like this:
<p>
<script src="https://gist.github.com/BerserkerDotNet/eb5feab41eaa0594315874013a27beca.js"></script>
    <noscript>
        <pre>
using System.Device.Gpio;

const int timeUnit = 500;
const int greenPin = 18;

using GpioController controller = new();
controller.OpenPin(greenPin, PinMode.Output);

var helloWorldInMorseCode = ".... . .-.. .-.. ---/.-- --- .-. .-.. -..";
foreach (var symbol in helloWorldInMorseCode)
{
    switch (symbol) 
    {
        case '.':
            PartOfLetterEnd();
            Dot();
            break;
        case '-':
            PartOfLetterEnd();
            Dash();
            break;
        case ' ':
            LetterEnd();
            break;
        case '/':
            WordEnd();
            break;
    }
}

PartOfLetterEnd();

controller.ClosePin(greenPin);

void Dot()
{
    controller.Write(greenPin, PinValue.High);
    Thread.Sleep(timeUnit);
}

void Dash()
{
    controller.Write(greenPin, PinValue.High);
    Thread.Sleep(timeUnit * 3);
}

void PartOfLetterEnd()
{
    controller.Write(greenPin, PinValue.Low);
    Thread.Sleep(timeUnit);
}

void LetterEnd()
{
    controller.Write(greenPin, PinValue.Low);
    Thread.Sleep(timeUnit * 3);
}

void WordEnd()
{
    controller.Write(greenPin, PinValue.Low);
    Thread.Sleep(timeUnit * 7);
}
        </pre>
    </noscript>
</p>

### Conclusions
In this part, Raspberry Pi was set up to work with VS Code remotely. .NET 6 SDK was installed on the Raspberry Pi and made visible from the bash terminal. Also, a small silly program was written to transmit “Hello World” in morse code using GPIO pins and the LED. This demo program made it clear that there is an easy way to work with Raspberry Pi GPIO pins from .NET. 

In part 2, the diode will be replaced with the actual sensors to measure temperature, humidity, and pressure. In addition, I'll show how to work with I2C devices using .NET on Raspberry Pi.

See you there, and happy coding!

[Pi]: https://www.Raspberry Pi.org/
[SunFounder]: https://www.amazon.com/dp/B014PF05ZA?psc=1&ref=ppx_yo2_dt_b_product_details
[LED]: https://www.sunfounder.com/products/dual-color-led-module
[BB]: https://en.wikipedia.org/wiki/Breadboard
[PiExt]: https://www.sunfounder.com/products/breakout-expansion-board?_pos=1&_sid=b9c3d170d&_ss=r
[PiImager]: https://www.raspberrypi.com/documentation/computers/getting-started.html#installing-the-operating-system
[RemoteSSH]: https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh
[PiSSH]: https://www.raspberrypi.com/documentation/computers/remote-access.html#ssh
[ARM]: [https://en.wikipedia.org/wiki/ARM_architecture]
[MorseCode]: [https://en.wikipedia.org/wiki/Morse_code]