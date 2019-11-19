WriteUp for Troopers20 Student Challenge UniComm
=================================================

This is the first WriteUp for the Student Challenges hosted by ERNW to make students able to visit the Troopers conference in Heidelberg, Germany. To apply for a student ticket you need to solve three challenges and write a letter of motivation. 

The first challenge is named UniComm and upon visiting its startpage you are presented with a few notices. The main text reads:

     UniComm is a propriatary protocol used by our appliances to communicate among each other. 

"The facts" tell you the following: 

    The protocol is TCP based, connect to the port using for example nc.
    
    It is a non-ascii protocol. Don't try to decode it as such.
    
    We cannot provide any documentation about the protocol.
    
    You can send any payloads to the service and use whatever information you can gather for other assignments.
    
    Do not perform any denial of service attacks.

So the most useful information might be, that it is a TCP based protocol, that does not send ASCII data. All the data transmitted and recieved was also captured using WireShark to be able to examine it afterwards.  When connecting with nc we get a greeting emoji as the answer:

    $ nc unicomm-master.fshbwl.ru 31337
    🙋

After some time we get an alarm clock emoji: ⏰ At first I looked at the timings between those two mesaages using the captured traffic and it seems like you get the clock emoji after 5 seconds. So let's try to send some data. For the first few tries I decided to go for just some strings. 

    $ nc unicomm-master.fshbwl.ru 31337
    🙋
    a
    🤔 a

    $ nc unicomm-master.fshbwl.ru 31337
    🙋
    data
    🧐 data

While messing around I thought about the name of the challeng "UniComm" and that it does not use ASCII. Maybe the "Uni" stands for Unicode. Depending on what you send the server replies with one of two emojis: 🤔 or 🧐. After the emoji the sent data is repeated and the connection closes without getting the clock emoji. Since the server greets us with a greeting emoji we can try to send back the same. 

    $ nc unicomm-master.fshbwl.ru 31337
    🙋
    🙋
    🙆
    9‍⃣️0‍⃣️1‍⃣️✖2‍⃣️6‍⃣️4‍⃣️
    ⏰

So after greeting the server back it sends another emoji and some unicode. The sent unicode seems to represent a math challenge. After getting some challenges it seems like it always consists of 2 numbers with 1-3 digits each and an operand between, that can be either +, - oder *. Since you have only 5 seconds to solve those I could hope for an easy one, I am able to solve in 5 seconds or automate the process of solving it. Since I was not patient enough end expected more challenges to follow after the first one, I decided to go for a python script. The majority of numbers were 3 digits long so I went for solvig only challenges with 2 numbers that consist of 3 digits. One thing to note is, that the digits are in boxes created by unicode encoding, but the digit itself is also contained in the data. The first byte is the digit itself and then 9 bytes follow to represent the box. 

``` python
    TCP_IP = '116.203.229.37'   # unicomm-master.fshbwl.ru
    TCP_PORT = 31337            # port given
    BUFFER_SIZE = 2048          # should be enough
    MESSAGE = b'\xf0\x9f\x99\x8b\n' # the greeting emoji to get to the challenge

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((TCP_IP, TCP_PORT))
    s.send(MESSAGE)

    time.sleep(1)                               # wait a second until we can be sure to have got the greeting emoji
    data = s.recv(BUFFER_SIZE)
    print("Whole answer from server:\n", data)
    print("🙋 from Server:\n", data[:5])         # first emoji
    print("🙆 from Server:\n", data[5:10])       # second emoji
    print("Challenge from Server encoded:\n", data[10:])   # raw challenge
    print("Challenge from Server decoded:\n", data[10:].decode("UTF-8")) # challenge itsefl
    print("first letter: ", data[10:11])            # extracting the digits of the first number
    print("second letter: ", data[20:21])
    print("third letter: ", data[30:31])
    FIRST_NUM = int(data[10:11])*100 + int(data[20:21]) *10 + int(data[30:31]) #calculating the first number
    print("First number: ", FIRST_NUM)

    print("operation to perform: ", data[40:43])
    OPERATOR = data[40:43]

    print("first number: ", data[-31:-30]) # getting the next digits from the end
    print("second number: ", data[-21:-20])
    print("third number: ", data[-11:-10])
    SECOND_NUM = int(data[-31:-30])*100 + int(data[-21:-20]) *10 + int(data[-11:-10]) # caclulating the second number
    print("Second number: ", SECOND_NUM)
    # Check which operator is meant to be used and calculating the result
    if OPERATOR == b'\xe2\x9c\x96':
        print("OP: *")
        RES = FIRST_NUM * SECOND_NUM
    elif OPERATOR == b'\xe2\x9e\x95':
        print("OP: +")
        RES = FIRST_NUM + SECOND_NUM
    elif OPERATOR == b'\xe2\x9e\x96':
        print("OP: -")
        RES= FIRST_NUM - SECOND_NUM
    else:
        print("Unknown Operator")       # meybe we get a surprise operator
        exit(1)
    print("Result: ", RES)

    s.send(str(RES).encode("UTF-8")+b'\n') # encode the result in UTF-8 and send it back
    data = s.recv(BUFFER_SIZE)

    print(data.decode("UTF-8"))             #print what comes back

    s.close()
```
After executing we get the following output:

    Challenge from Server decoded:
    5‍⃣️5‍⃣️4‍⃣️➕4‍⃣️6‍⃣️6‍⃣️

    first letter:  b'5'
    second letter:  b'5'
    third letter:  b'4'
    First number:  554
    operation to perform:  b'\xe2\x9e\x95'
    first number:  b'4'
    second number:  b'6'
    third number:  b'6'
    Second number:  466
    OP: +
    Result:  1020
    -----BEGIN FISHBOWL FLAG-----
    Pz7N5MBbV2PNZcsOAsvKxFiCPvZukzgO2jJVFzTwGCy5OEFb
    H6JVLK887XwZjNhHEVqRfcX5TGkX+4LFEGnWNHjWPL+7nZyX
    c/D8JbBXpyBuVePjyA9SK+A6TYaCcT5cxcsX0FJPcoK7jDsG
    5G7DO59YSMc=
    -----END FISHBOWL FLAG-----
    🙃

Okay seems like we already got the flag and do not need to solve other challenges. 
