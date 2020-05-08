# The Mystery of iOS Background Service Advertising

How does Apple's proprietary technique for background GATT service advertising work?

According to Apple's documentation, when an iOS app using CoreBluetooth to implement a BLE peripheral is in the background, service UUIDs are no longer advertised, and instead are put on a special "overflow area":


> Any service UUIDs contained in the value of the CBAdvertisementDataServiceUUIDsKey key that don’t fit in the allotted space go to a special “overflow” area. These services are discoverable only by an iOS device explicitly scanning for them.
> While your app is in the background, the local name isn’t advertised and all service UUIDs are in the overflow area.

[Reference](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager/1393252-startadvertising)

But what is this "overflow area"?  How does it work?

**UPDATE:  The procedure below failed to find the answer.  But I finally figured it out another way.  See [here](http://www.davidgyoungtech.com/2020/05/07/hacking-the-overflow-area).**

## Test App Behavior

To find out, I built an iOS app [here](https://github.com/davidgyoung/background-advertiser-ios) that advertises GATT Service UUID `2F234454-CF6D-4A0F-ADF2-F4911BA9FFA6`.   The app runs both a BLE central and peripheral.  The app's peripheral code advertises that service, and the app's central code scans for it and connects to it if found.

When in the foreground, the app advertises the service UUID as expected in a type 07 BLE packet that contains the service UUID in little endian order:

```
4-14 22:12:57.509 15890 6A:9F:24:DE:23:61 02011a1107a6ffa91b91f4f2ad0f4a6dcf5444232f0000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

When in the background, the advertisement changes to a manufacturer advertisment that looks like this:

```
04-14 22:12:57.580 15890 6A:9F:24:DE:23:61 02011a14ff4c0001000000000000000000000800000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

Yet somehow other iOS devices in the foreground specifically scanning for service UUID `2F234454-CF6D-4A0F-ADF2-F4911BA9FFA6` will discover the peripheral advertising the above manufacturer advertisement.  How is this possible?  Clearly there must be other communication involved as the advertisement does not contain the service UUID.  Presumably this communication is over BLE.

## Traffic Sniffing

To investigate further I set up a BLE sniffer.   I used an [Ubertooth device](https://github.com/greatscottgadgets/ubertooth/blob/master/host/README.btle.md) in follower mode.  It listens on a single BLE advertising channel logging all traffic for a specific MAC address, and then if it sees a connection attempt on that channel, it starts switching channels to follow the traffic. 

Using this tool, I can log traffic from this exchange.  But because there are multipel BLE advertising channels, and the Ubertooth can only be on one channel at a time, you have to get lucky to capture the proper exchange.

So I ran a test over and over using two iOS devices running the app described above and logged the results.

## Test Setup

For my test, I used two iOS devices with the same app described above.  One app was kept in the background, the other was in the foreground.

iPhone 6, iOS 11 (Background)  Rotating MAC: 4a:0d:a8:1f:84:6f
iPod Touch 6th Generation, iOS 12 (Foreground)

I found the Rotating MAC of the iPhone 6 by putting the Ubertooth right next to the phone and looking for the detected device with the highest RSSI (about -40 dBm which is very strong.)   That device had MAC 4a:0d:a8:1f:84:6f.    I then started a capture logging traffic to and from that MAC address:

`$  ubertooth-btle -t 4a:0d:a8:1f:84:6f`
`$  ubertooth-btle -f`

I then ran this capture for as long as I continued to get results.  This lasted 480 seconds.  It likely stopped after that becasue iOS rotated the MAC of the iPhone 6.

While the capture was running, I repeatedly rebooted the iPod Touch, and launched the test app right after boot.  I let the test app run for about 10 seconds then rebooted and restarted again.  The reason I kept rebooting is that I wanted to capture multiple attempts by the foregrounded app on teh iPod touch to detect the background advertisement from the iPhone 6, starting from a clean slate each time.  

I did 10 boot cycles during this test.  And while I did not confim that a discovery/connection was successful each time, I know from previous tests that this process is highly reliable, so I am confident that there were indeed 10 discovery and connections over the course of the log.

To be clear, these were not the only two BLE devices in the vicinity.  There were other phones and laptops in BLE range.  So some traffic logged was from other devices not part of this controlled test.

## Results

The [logs from Ubertooth](ios-discovery-of-gatt-service-from-background-advert-multiple-attempts.txt) captured 6 connection requests.  Here's one:

```
systime=1588434626 freq=2402 addr=8e89bed6 delta_t=0.470 ms rssi=-62
c5 22 f3 c0 b9 9d a6 64 6f 84 1f a8 0d 4a 6a 95 9a af 34 4a 87 03 0b 00 18 00 00 00 48 00 ff ff ff ff 1f aa 40 ad b0
Advertising / AA 8e89bed6 (valid)/ 34 bytes
    Channel Index: 37
    Type:  CONNECT_REQ
    InitA: 64:a6:9d:b9:c0:f3 (random)
    AdvA:  4a:0d:a8:1f:84:6f (random)
    AA:    af9a956a
    CRCInit: 874a34
    WinSize: 03 (3)
    WinOffset: 000b (11)
    Interval: 0018 (24)
    Latency: 0000 (0)
    Timeout: 0048 (72)
    ChM: ff ff ff ff 1f
    Hop: 10
    SCA: 5, 31 ppm to 50 ppm

    Data:  f3 c0 b9 9d a6 64 6f 84 1f a8 0d 4a 6a 95 9a af 34 4a 87 03 0b 00 18 00 00 00 48 00 ff ff ff ff 1f aa
    CRC:   40 ad b0
```

It is possible that some of these connection requests may not have been from the iPod Touch device in the test.   But most of not all probably were.  While looking at the logs in real time during testing, the connecton traffic was noticable, and it typically showed up shortly after launching the app on the iPod Touch.

What the logs don't capture is any byte sequence that matches the Service UUID of `2F234454-CF6D-4A0F-ADF2-F4911BA9FFA6`.   If you search the log for either "2f 23" or "23 2f" (little endian and big endian ordering) you get nothing.  The service UUID that is in this mysterious "overflow area" was not in any of the captured packets.

## Conclusions

This test failed to find any packets that included the advertised GATT Service UUID.  Leaving the mystery of how the "overflow area" works unsolved.

A few possible explanations:

* Perhaps the Service UUID was communicated on a different  BLE channel so the Ubertooth didn't pick it up even though the iPod Touch did.  
* Perhaps the Service UUID was communicated in a packet that didn't match the MAC address filter.
* Perhaps the Service UUID was not communicated over BLE at all -- it may have somehow been in cache on the iPod Touch.








