# Overview

## Hardware Outputs

The Ni lab computer control system is a flexible control system for controlling all the key components of an AMO Physics experiment. The key categories of control signals include **RF outputs**, **Analog outputs** and **Digital outputs**. 

- **RF outputs**: These are output signals with a certain frequency (typically tens to hundreds of MHz) and an amplitude. They are used mostly for driving acousto-optic/electro-optic modulators and providing reference RF signals for laser locking. 

- **Analog outputs**: These are relatively low frequency output signals (one update every 2 to 2.5 microseconds) that can take arbitrary shapes in time. They are used as control signals for various PID control circuits in the lab including those that stabilize the current through magnetic field coils or laser intensity. 

- **Analog outputs (faster)**: This is in its own category, since it utilizes its own device, which is an arbitrary waveform generator. The AWG updates so that one sample can be output every 1.6 ns or even faster. We use this for creating multiple tweezers (i.e. generating multiple RF tones at once). 

- **Digital outputs**: These are signals that take two discrete values, low and high. They are used for interfacing with external hardware including the triggering of external function generators, scientific cameras, laser shutters or controlling the route of RF signals. When setting up a new piece of equipment in the lab, utilizing these existing RF, analog and digital outputs (usually digital) are the easiest ways to get started quickly!

Note that our FPGA is the master time reference for the entire sequence. It directly interfaces with RF outputs (Analog Devices DDS chips) with physical connections and also directly provides the TTL outputs. It clocks our slow analog output (National Instruments DAC card) and also provides the start trigger for our fast analog output (Spectrum Instrumentation AWG)

## Control Software

For the purpose of control, we require a flexible and easy-to-use system. Our main goals are to allow the user to do anything they can think of, and also allow the more curious to explore and add new devices and functionality as easily as possible without compromising quality. To this end, here is a non-exhaustive list of features and design considerations of our system

- Script-based frontend: We use a script-based frontend (currently MATLAB) for constructing and running sequences. Scripts can take advantage of all the features of a given programming language, especially control flow logic. For instance, it is easy to write loops and also conditional statements. It is easy to incorporate other software and also to interweave with analysis code and use for feedback. A graphical interface can be built on top of the script interface.

- Sequence compilation: After creation of a sequence, the sequence is compiled to a device-independent intermediate representation, doing as much work as possible before running the sequence. This includes compiling any ramp functions that are used in the experiment.

- Device abstraction: The device-independent intermediate representation of the sequence can then be used in a device-specific implementation. These device classes implement what should be done during a sequence run or what is done prior to a sequence being run. Thus, adding a new device to the control system on the control software side simply requires following this template. 

- Web-based control interface: Our control suite includes a web-based control interface, where currently our digital and RF outputs can be flipped on and off and parameters changed. Our web-based control interface has a simple API to communicate *independently* (from our sequence runner) with devices, making new device additions straightforward.

- Flexible parameter scanning: Our sequence construction API allows for easily scannable parameters in various configurations (one can scan multiple parameters together, or across different dimensions). Default values are also supported to allow for easily testing parameters, while keeping default parameters to use if no change is needed.

- Dynamic in-sequence variables: Steps in a sequence can be controlled by dynamic variables which can be changed at the millisecond timescale. These dynamic variables represent all concepts including the time, amplitude or even ramp type of a pulse. These dynamic variables are changed during atom rearrangement. 

More details about this entire system can be found elsewhere on this guide, but hopefully this will get you started thinking about the requirements of our control system and the features we support

## What actually happens when you run a sequence?

Let's follow the "lifespan" of a sequence from the time you hit the `run` button (or more appropriately run the `run` function). First, the MATLAB sequence will be serialized into a string of bytes that will then be reconstructed into a different representation in C++ (via Python). The C++ version of the sequence will then be further processed and compiled, ultimately into a list of "abstract commands" for all devices. It is up to each device to define what an abstract command means and how an abstract command should be interpreted. For instance, for the FPGA, these commands consist of setting a DDS amplitude, frequency or phase as well as changing a TTL output. For the slow analog output, we simply give the device a list of values to set. For the AWG, commands change the amplitude, frequency and phase of a particular RF tone being generated by the FPGA. These "abstract commands" are abstract in the sense that their actual values may be changed by the so-called dynamic in-sequence variables in the previous [section](./index.md#control-software). During the actual sequence, after the dynamic in-sequence variables are determined, "actual commands" (once again determined independently by each device) are created and sent off to each device to prepare to run. Then, once all devices are ready to go, the FPGA is told to run and the sequence is now run. 