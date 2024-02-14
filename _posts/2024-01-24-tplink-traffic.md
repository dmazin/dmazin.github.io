Recently, I bought some TPLink WiFi lightbulbs. I know that WiFi is probably not the best medium (todo: is that the right word?) for smart devices: I will probably switch to Zigbee soon.

Why is WiFi not great? One reason is that it's not designed for IoT communications. For example, Zigbee is able to use lower power (and thus cause less interference, even though it uses the crowded 2.4 Ghz band [todo: is band the right word]) and transmits less data (todo: how? by avoiding TCP?).

One more reason I don't want IoT devices on my WiFi network is because it adds to network traffic. Does it actually add a significant amount, though? That's what I'll try and measure today.

Lately, I've been reading a great book about networking -- (link to textbook) -- it's very slightly outdated, but it explains networking in a way I really understand, and goes into just the  right amount of detail.

What I've learned is that, originally, Ethernet was a shared medium. That is, each host connected to the same coax cable, and each message was delivered to each host (if the message wasn't for you, you ignored it). These days, that's not how we use Ethernet -- we use it point-to-point, each device connecting to a switch, and so a device doesn't get packets not intended for it.

The original, shared, way of using Ethernet sounds a lot like WiFi, though. Does every packet sent by the access point get delivered to every device?

Actually, I'm not sure. I put the wireless interface on a Raspberry Pi into promiscuous mode (todo: needs to be explained), and ran tshark (the command-line version of Wireshark), but I only saw packets intended for that interface. A little reading showed that the driver might already filter the packets, or, perhaps, there is some feature of WiFi or my router that prevents devices from getting packets not intended for them (though I'm not sure how that'd work).

But, all is not lost. Even given the above, WiFi is still a shared medium, just like Ethernet over shared coax. Some wireless cards can be put into "monitor" mode, where they capture all the packets flying around in the air -- and this works even if you have not joined the WiFi network.

I am a little surprised that this is a thing, because I imagined that wireless devices are, like, sending/receiving signals, but these signals are... like, "incomprehensible" streams of bits, that only make sense if you have joined the wireless network.

But, thinking about it, it makes sense. There's a structure to all networking. The things sent wirelessly from device-to-device are not random: they are encapsulated, just like the frames sent over Ethernet. At least, this is the case for WiFi. (I wonder if Bluetooth and Zigbee are the same?)

And so, if you have an antenna capable of picking up the wireless signal, you can capture those messages. And so that's what I'll do now.

# Notes
- [ ] Update filename (don't mention tplink unless it's relevant)
- [ ] Spelling and grammar check
