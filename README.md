# Sermatec documentation
This repository contains all the information related to the Sermatec products.

## Introduction
Sermatec makes a wide range of solar inverters, for small plants there are 5 kW and 10 kW hybrid models.

*Update 2024: As of now, the Sermatec stopped manufacturing and distributing residential solar inverters. Pages about the inverters are no longer present on official website and according to information of my local distributor, Sermatec no longer makes home inverters. I wouldn't be suprised if the decision to stop production was caused by some security incident...*

These inverters contain WiFi module, which is officially used to connect to manufacturer's server to provide data for official application. Also, there is a Modbus interface present, which allows to access all inverter's registers and thus complete control over the device.

## WLAN (WiFi) communication
The hybrid inverters contains the USR-WIFI232-B2 UART-TCP converter, which supports 802.11 b/g/n and it is the main method of communication with the inverter.
The API protocol used is called OSIM (Optical Storage Integrated Machine).

It supports two modes:
- station mode: connects to the home AP, if there is an Internet access, it attempts to connect to the cloud server with a defined IP.
- AP mode: local connection mode in the official app, used mainly for maintenance, setting parameters and service (firmware upgrade). Setting parameters require a password. Luckily it is hardcoded, here is a list of discovered passwords:
    - `Sermatec2021`
    - `sermatec2021` - confirmed working in the app version 1.7.5
    - `Sermatec2015`
    - `Sermatec2019@Gsstes2019` - used as a RC4 key

There are several ports open:
| Port | Usage |
| ---- | ---- |
| 23/tcp | telnet |
| 80/tcp | USR-WIFI configuration page (login and password: `admin`) |
| 8000/tcp | unknown |
| 8899/tcp | API (see [script](https://github.com/sermatec-opensource/sermatec-inverter)) |

### Security considerations
⚠️⚠️⚠️⚠️ The WLAN communication poses several security risks:
- the default AP password is well-known (`gsstes123456`) and if it is not changed (which is not done even by professional installation companies), anybody can connect to the inverter, and:
    - can potentionally destroy the inverter and connected batteries by setting up improper configuration,
    - can find the home AP login details and connect to the inverter owner's home network.
- the inverter connects to the remote server and can be controlled by the Sermatec or the installation company remotely; the Sermatec company itself as the owner of the servers, or any attacker which gain access to the aforementioned server can possibly destroy the connected inverters.

The not-so-fun thing is that if the AP password is changed via the USR-WIFI configuration page, it will be changed back to default on next restart by the inverter's microcontroller (which probably deploys USR-WIFI setup on every boot).

The fact that the inverters can be potentionally abused remotely means not only risk of destroying the inverter/batteries and gaining the home AP credentials but also a possibility of causing a blackout if the area relies solely on the solar power.

⚠️ The WebUI password length is 15 chars max.

### Alternative to official application
The safest way of securing the inverter is connecting it to a network with no access to the Internet and then using either local connection mode or tools developed by our community:
- [Python script](https://github.com/sermatec-opensource/sermatec-inverter), which can be integrated in your own projects,
- [Home Assistant integration](https://github.com/sermatec-opensource/homeassistant-sermatec-inverter).

### Protocol documentation
The protocol used on this interface is different from the Modbus, it does not even use similar addresses or structure. This protocol is not public and was not available even for official distributors. All information about the protocol comes from community's reverse-engineering efforts. Documentation of the protocol is in the `PROTOCOL.md` file.

## Modbus communication
There is also a Modbus interface. This is a manufacturer's offical way of integrating the inverter with other devices (e.g. inverter distributor's control module), it allows to read all data and set all inverter's parameters. It is the most powerful but also the most dangerous way of interfacing with the inverter.

Modbus parameters:
- method: rtu
- stopbits: 1
- bytesize: 8
- parity: none
- baudrate: 9600
- unit: 1
- type: Read Holding Registers (function code 03)

Sadly, whole documentation of available registers cannot be published -- it is not public and was made available only for distributors. The best option is to ask your distributor or wait for some leak. Only some of the addresses were published, which are documented here:

| Address | Name | Unit | Data type |
| ------- | ---- | ---- | --------- |
| 0x3000 | battery voltage | 0.1 V | U16 |
| 0x3001 | battery current | 0.1 A | S16 |
| 0x3002 | battery temperature | 0.1 °C | S16
| 0x3003 | battery SOC | 1 % | U16 |
| 0x4000 | PV1 voltage | 0.1 V | U16 |
| 0x4001 | PV1 current | 0.1 A | S16 |
| 0x4002 | PV1 power | 1 W | S16 |
| 0x4003 | PV2 voltage | 0.1 V | U16 |
| 0x4004 | PV2 current | 0.1 A | S16 |
| 0x4005 | PV2 power | 1 W | S16 |
| 0x5509 | Remote load power (on-grid mode; from meter) | 1 W | U16 |
| 0x4035 | Load active power (off-grid mode) | 1 W | S16 |
| 0x4017 | Net side active power (negative = import from grid, positive = export to grid) | 1 W | S16 |
