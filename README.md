# Cisco-HDLC

ds: Linux kernel module for Cisco HDLC encapsulation

---

PDS Linux Kernel Device Driver is a driver for E1 devices connected via
Ethernet. Currently, it is assumed that only one E1 device is connected to one
Ethernet port: the driver sends messages via broadcast. (Adding the ability to
work via unicast is not difficult, just no one needs it at the moment.)

The driver allows you to work in the so-called fake mode, in which it still
returns a positive status without receiving a response from the device. (In
configuration mode, data transmission is stateless.) In this mode, the PDS can
be used as a virtual E1 tunnel: for example, it is useful for automated testing
of E1/DAHDI configuration system.

As an add-on, there is also a simple implementation of the Cisco HDLC Ethernet
encapsulation module. Keepalive messages are not supported.
