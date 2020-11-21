---
title: "A custom streamdeck<br/>Part 2: Existing solutions"
date: 2020-11-21T21:00:00-00:00
categories:
  - blog
  - Arduino
  - Firmware
  - Hardware
  - Streaming
tags:
  - Elgato
  - MIDI
  - Twitch
  - OBS
---

## Introduction
Last time I discussed the results of a survey which my streamer friend [James](https://www.twitch.tv/gameswithjames01){:target="_blank"} put out to his Twitter followers. These results are enough to get us started on the first prototype, but I would be remiss not to do some additional research on the streamdeck market before heating up the soldering iron. 

The benefits of this are twofold. Firstly, it allows us to see what products we could be competing against and whether they have any shortcomings we can address in our design. And secondly, it gives an insight into any existing tools that are available to us. 'Build on the shoulder of giants' is one of my favourite design mantras, and it's easier than ever nowadays thanks to the popularity of open source.

## Elgato
Elgato is a German/US company who design equipment for streamers. Their catalogue includes 3 hardware streamdecks - in mini, regular and XL flavours - and an app based streamdeck for mobile. The standout feature of their hardware streamdecks are the buttons with integral LCD screens, which allow streamers to customise the interface to their specific needs. Elgato's streamdecks offer almost limitless control thanks to a vast library of plugins, which is maintained by members of the streaming community along with Elgato themselves.

The 3 hardware streamdecks are currently available for £80, £140 and £210 on Amazon. A large portion of this cost is likely down to the impressive 'plug and play' software features which make them accessible to even the most ardent luddite.

![](/assets/images/elgato.jpg)

## Mechanical keyboards
The humble keyboard is the streamdeck everyone starts off with. Modern life would be more or less impossible without keyboards, so the industry has diversified to suit the needs of a growing number of users. These days you can not only choose the colour of your keyboard (and specific keys) but the feel of it too, with switches tailored for individual gaming and typing preferences. Custom layouts are also possible, and this is of particular interest to us with regards to streamdecks. LEDs are commonplace on many modern keyboards as a means of aiding usability in low light conditions as well as providing an aesthetic boost. 

Suppliers such as [Mechboards UK](https://mechboards.co.uk){:target="_blank"} sell all the components you need to build your own customised keyboard for a low price. A streamdeck built from such components - mounted in a nice 3D printed enclosure - is shown below (credit to [Parts Not Included](https://www.partsnotincluded.com/diy-stream-deck-mini-macro-keyboard){:target="_blank"}).

![](/assets/images/StreamCheap.jpg)

## MIDI controllers
The MIDI standard was first introduced back in the 80's, and has changed surprisingly little since then. MIDI allows different musical devices to talk to eachother in a common language, giving musicians the flexibility to mix and match hardware to their heart's content. A MIDI controller is typically used to send MIDI messages to music software such as a DAW (Digital Audio Workstation) or DJ software. However, so long as the necessary drivers are installed, there's no reason why these messages can't be used to control other software such as OBS.

The switches in MIDI controllers are typically tactile in nature; they have shorter travel and higher actuation force than those found in mechanical keyboards. This is useful in a performance environment, as timing of button presses can be more tightly controlled. Some MIDI controllers also feature knobs and sliders for adjusting parameters. As MIDI controllers often receive messages as well as send them, LEDs can be used to give visual feedback to the user.

![](/assets/images/akai.jpg)

## LioranBoard
This is a free software based streamdeck for OBS. While not as polished as Elgato's offering, LioranBoard is a logical first step for streamers looking to improve their workflow.

## Next steps
The time is right to get our first prototype up and running. As with all prototypes, getting something working as quickly as possible is of utmost importance, as this minimise the time between design iterations. Building a purely software based prototype would be outside my comfort zone - and somewhat redundant given the existence of LioranBoard - so hardware is where we'll focus our initial efforts. Although I do have some experience in building MIDI controllers, my concern is this would not be as 'plug and play' as a comparable HID compatible device. Therefore, my first design will be based on a custom mechanical keyboard, using cheap and readily available components.

