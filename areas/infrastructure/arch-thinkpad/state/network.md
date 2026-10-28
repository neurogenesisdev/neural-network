## interfaces
enp2s0f0         DOWN           <MAC_enp2s0f0> <NO-CARRIER,BROADCAST,MULTICAST,UP> 
enp4s0           DOWN           <MAC_enp4s0> <NO-CARRIER,BROADCAST,MULTICAST,UP> 
enp6s0f3u1u1u2   DOWN           <MAC_enp6s0f3u1u1u2> <NO-CARRIER,BROADCAST,MULTICAST,UP> 
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP> 
tailscale0       UNKNOWN        <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> 
wlp3s0           UP             <MAC_RANDOMISED_wlp3s0> <BROADCAST,MULTICAST,UP,LOWER_UP> 

## permanent MACs
enp2s0f0 permaddr=
enp4s0 permaddr=
enp6s0f3u1u1u2 permaddr=
tailscale0 permaddr=
wlp3s0 permaddr=<PERMADDR_wlp3s0>

## nftables
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		ct state invalid drop
		ct state { established, related } accept
		iif "lo" accept
		ip protocol icmp limit rate 10/second burst 5 packets accept
		ip6 nexthdr ipv6-icmp limit rate 10/second burst 5 packets accept
		tcp dport 22000 accept
		udp dport { 21027, 22000 } accept
		iifname "tailscale0" accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
	}

	chain output {
		type filter hook output priority filter; policy accept;
	}
}

## other tables present (Tailscale-managed, contents volatile)
table inet filter {
table ip filter {
table ip mangle {
table ip nat {
table ip6 filter {
table ip6 mangle {
table ip6 nat {

## listening sockets
tcp *:22000 users:(("syncthing"))
tcp 0.0.0.0:<ephemeral> users:(("spotify"))
tcp 0.0.0.0:<ephemeral> users:(("spotify"))
tcp <TS_IPV4>:22 users:(("sshd"))
tcp <TS_IPV4>:<ephemeral> users:(("tailscaled"))
tcp 127.0.0.1:8384 users:(("syncthing"))
tcp [<TS_IPV6>]:<ephemeral> users:(("tailscaled"))
udp *:22000 users:(("syncthing"))
udp 0.0.0.0:1900 users:(("spotify"))
udp 0.0.0.0:21027 users:(("syncthing"))
udp 0.0.0.0:41641 users:(("tailscaled"))
udp 0.0.0.0:5353 users:(("spotify"))
udp 0.0.0.0:<ephemeral> users:(("spotify"))
udp 0.0.0.0:<ephemeral> users:(("spotify"))
udp 0.0.0.0:<ephemeral> users:(("syncthing"))
udp 224.0.0.251:5353 users:(("spotify"))
udp 224.0.0.251:5353 users:(("spotify"))
udp [::]:21027 users:(("syncthing"))
udp [::]:41641 users:(("tailscaled"))
udp [::]:5353 users:(("spotify"))
udp [::]:5353 users:(("spotify"))
udp [::]:<ephemeral> users:(("syncthing"))

## NetworkManager conf.d
/etc/NetworkManager/conf.d/00-macrandomize.conf:
/etc/NetworkManager/conf.d/00-macrandomize.conf:[connection]
/etc/NetworkManager/conf.d/00-macrandomize.conf:[device]
/etc/NetworkManager/conf.d/00-macrandomize.conf:connection.stable-id=${CONNECTION}/${BOOT}
/etc/NetworkManager/conf.d/00-macrandomize.conf:ethernet.cloned-mac-address=stable
/etc/NetworkManager/conf.d/00-macrandomize.conf:wifi.cloned-mac-address=stable
/etc/NetworkManager/conf.d/00-macrandomize.conf:wifi.scan-rand-mac-address=yes

## resolv.conf owner
-rw-r--r-- 1 root root 257 Oct  3 19:31 /etc/resolv.conf

## usbguard
allow id 046d:c52b serial "" name "USB Receiver" hash "djeL7wNsJBQMuBiqUyWflgupndhsbPkbOih8g3L6OeA=" parent-hash "+0s5mKAEDBjZasKfFR9ExKfjpMr/J4C4yq4bgYdLJSM=" with-interface { 03:01:01 03:01:02 03:00:00 } with-connect-type "hotplug"
allow id 0bc2:231a serial "<USB_SERIAL_NA8J>" name "Expansion" hash "3IrVg6/E0tp6laUJDXoh7M5J+5WkaARHukSrjJMnfYc=" parent-hash "cVFMyGdwqzSlQbDVPkNvf7ngduLu3eE8/7E8dBmNJHc=" with-interface { 08:06:50 08:06:62 } with-connect-type "unknown"
allow id 0bda:4852 serial "<USB_SERIAL_00e0>" name "Bluetooth Radio" hash "62+eL3fnbawNeK3D4H4qSHmmND3Mmt0X//9gfbgjfE4=" parent-hash "TaKoMrgQrk94nyzpOQk+iNVB0H+ZSnYN/X7lY+QzAn0=" with-interface { e0:01:01 e0:01:01 e0:01:01 e0:01:01 e0:01:01 e0:01:01 e0:01:01 } with-connect-type "not used"
allow id 13d3:5415 serial "" name "Integrated Camera" hash "6QINoKEN0ocK/xHenraLlMwujxcx0VrI6v4vMInJrFw=" parent-hash "vL+E5pTms/14TMT1FF36bNufC5tB/6+/dKHVKIZVAMI=" with-interface { 0e:01:00 0e:02:00 0e:02:00 0e:02:00 0e:02:00 0e:02:00 0e:02:00 0e:02:00 } with-connect-type "not used"
allow id 17ef:307f serial "<USB_SERIAL_0000>" name "USB3.1 Hub             " hash "cVFMyGdwqzSlQbDVPkNvf7ngduLu3eE8/7E8dBmNJHc=" parent-hash "ej9rWyOKtN32uAkJVjY+KzTM3N0TbXLv9atySDfqNgg=" with-interface 09:00:00 with-connect-type "unknown"
allow id 17ef:307f serial "<USB_SERIAL_0000>" name "USB3.1 Hub             " hash "ej9rWyOKtN32uAkJVjY+KzTM3N0TbXLv9atySDfqNgg=" parent-hash "mY33/owHioZsFcJ98I14TeQ+B9tqBjX0nE9Ogy2f4Hk=" with-interface 09:00:00 with-connect-type "hotplug"
allow id 17ef:3080 serial "<USB_SERIAL_0000>" name "USB2.0 Hub             " hash "IhXEiXbbpR+z7Cy47Z/FdbP+FoIlhkRonjmZPugTDL4=" parent-hash "kEmk59jhT66dAfeKLu268+WxYaPI+cIvXH9FBKv/sYw=" with-interface { 09:00:01 09:00:02 } with-connect-type "unknown"
allow id 17ef:3080 serial "<USB_SERIAL_0000>" name "USB2.0 Hub             " hash "kEmk59jhT66dAfeKLu268+WxYaPI+cIvXH9FBKv/sYw=" parent-hash "+0s5mKAEDBjZasKfFR9ExKfjpMr/J4C4yq4bgYdLJSM=" with-interface { 09:00:01 09:00:02 } with-connect-type "hotplug"
allow id 17ef:3081 serial "" name "USB2.0 HUB" hash "Mk7mxc+fBrn8jKHjmxkLj7sx7DOy1LyiCce5rDoPo1Y=" parent-hash "IhXEiXbbpR+z7Cy47Z/FdbP+FoIlhkRonjmZPugTDL4=" with-interface { 09:00:01 09:00:02 } with-connect-type "unknown"
allow id 17ef:3082 serial "<USB_SERIAL_1015>" name "ThinkPad TBT 3 Dock" hash "jsTYaYKQrgjRGcRPzf0qsP3Yk834jxqF2HZSog8OFao=" parent-hash "cVFMyGdwqzSlQbDVPkNvf7ngduLu3eE8/7E8dBmNJHc=" with-interface { ff:ff:00 02:06:00 0a:00:00 0a:00:00 } with-connect-type "unknown"
allow id 17ef:3083 serial "<USB_SERIAL_0000>" name "ThinkPad Thunderbolt 3 Dock USB Audio" hash "XT6jBJjHa2bRyukq1Zt51btac2MnmqOVBGR7sWh3aUQ=" parent-hash "Mk7mxc+fBrn8jKHjmxkLj7sx7DOy1LyiCce5rDoPo1Y=" with-interface { 01:01:00 01:02:00 01:02:00 01:02:00 01:02:00 03:00:00 } with-connect-type "unknown"
allow id 1d6b:0002 serial "0000:02:00.4" name "EHCI Host Controller" hash "g4C+/9Czivl3COs7n5IrgCn8aGKQyA1nbVaXFO674/I=" parent-hash "R/6eYWpJE+6YUXeuG6yQIR13+fLXJHFSlqUyBUoo8So=" with-interface 09:00:00 with-connect-type ""
allow id 1d6b:0002 serial "0000:05:00.0" name "xHCI Host Controller" hash "vL+E5pTms/14TMT1FF36bNufC5tB/6+/dKHVKIZVAMI=" parent-hash "ILG3rVl27a2RDaMkj6U6IcS918L/EcKOC8V5M/MeZpA=" with-interface 09:00:00 with-connect-type ""
allow id 1d6b:0002 serial "0000:06:00.3" name "xHCI Host Controller" hash "+0s5mKAEDBjZasKfFR9ExKfjpMr/J4C4yq4bgYdLJSM=" parent-hash "KTj0i1ONjkGo2CJx42BsIwl+RMi6YVks67qrDYNrwPo=" with-interface 09:00:00 with-connect-type ""
allow id 1d6b:0002 serial "0000:06:00.4" name "xHCI Host Controller" hash "TaKoMrgQrk94nyzpOQk+iNVB0H+ZSnYN/X7lY+QzAn0=" parent-hash "Uk+Btxo+I+tLT7+0bnE1NSmtnqPXUzmSqOXpyVHnU+o=" with-interface 09:00:00 with-connect-type ""
allow id 1d6b:0003 serial "0000:05:00.0" name "xHCI Host Controller" hash "s7Z4BYiDDvaR6/+iivMSASVOogbt2HrWrWeUpnCj2yU=" parent-hash "ILG3rVl27a2RDaMkj6U6IcS918L/EcKOC8V5M/MeZpA=" with-interface 09:00:00 with-connect-type ""
allow id 1d6b:0003 serial "0000:06:00.3" name "xHCI Host Controller" hash "mY33/owHioZsFcJ98I14TeQ+B9tqBjX0nE9Ogy2f4Hk=" parent-hash "KTj0i1ONjkGo2CJx42BsIwl+RMi6YVks67qrDYNrwPo=" with-interface 09:00:00 with-connect-type ""
allow id 1d6b:0003 serial "0000:06:00.4" name "xHCI Host Controller" hash "loJYlkuv5WE4gGiX77MET1+gKtJiS8m4G4M0xhh1V6M=" parent-hash "Uk+Btxo+I+tLT7+0bnE1NSmtnqPXUzmSqOXpyVHnU+o=" with-interface 09:00:00 with-connect-type ""
allow id 2109:8887 serial "<USB_SERIAL_40AN>" name "40AN" hash "GOS88lswv7jzMBGTBZ6Zc9p9R5JekR7h+vC3e7NIkVs=" parent-hash "kEmk59jhT66dAfeKLu268+WxYaPI+cIvXH9FBKv/sYw=" with-interface 11:00:00 with-connect-type "unknown"
block id 04e8:61f5 serial "<USB_SERIAL_1234>" name "Portable SSD T5" hash "RuWEil/5aPaCu5Z1zvVWE7CkIzYarhnPR/qBkjfIorM=" parent-hash "kEmk59jhT66dAfeKLu268+WxYaPI+cIvXH9FBKv/sYw=" with-interface { 08:06:50 08:06:62 } with-connect-type "unknown"
block id 0781:5581 serial "<USB_SERIAL_0401>" name " SanDisk 3.2Gen1" hash "lNW0pcXeKiDEFudrtvgbgo7xwPIfuhbO1oDDMBVs4Jk=" parent-hash "loJYlkuv5WE4gGiX77MET1+gKtJiS8m4G4M0xhh1V6M=" with-interface 08:06:50 with-connect-type "hotplug"
