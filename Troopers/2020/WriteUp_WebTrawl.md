WriteUp for Troopers20 Student Challenge UniComm
=================================================

After solving the first two challenges it was time for me to jump into the third one. At first I looked at the provided description.

    In times of large-scale cyber attacks and cloud-first applications you need something to protect your data from evil forces. To better understand past and ongoing attacks, WebTrawl provides you with highly sophisticated filters, over the top performance and best security.†

As this does not give us useful information, let's look at "The Facts"


    Identify malicious actors and effectively block the threat
    WebTrawl is easily extendable to accomodate for new threats.
    WebTrawl is optimized for speed, minimizing delays between receiving and processing requests.
    Our high performance cache engine (Magikarp) guarantees lightning fast responses for frequently requested resources.
    WebTrawl gives you fine-grained control over what to filter.
    You will be given a download link for WebTrawl.
    You may reverse the program given to you to identify issues.
    Do not perform any denial of service attacks or attack different ports.

Okay so we are given the software itself and should be able to reverse it. The service is hosted on webtrawl-test.fshbwl.ru Port 443. Also, a caching engine called "Magikarp" is mentioned. After downloading the app, we see a list of files with the pyc extension. 

    -rw-r--r-- 1 dennis dennis 4.0K Sep 27 08:55 cache.pyc
    -rw-r--r-- 1 dennis dennis 1.2K Sep 27 08:55 filter.pyc
    drwxr-xr-x 2 dennis dennis 4.0K Sep 27 08:55 http
    -rw-r--r-- 1 dennis dennis 2.9K Sep 27 08:55 __init__.pyc
    -rw-r--r-- 1 dennis dennis  918 Sep 27 08:55 loghandler.pyc
    -rw-r--r-- 1 dennis dennis  561 Sep 27 08:55 __main__.pyc
    -rw-r--r-- 1 dennis dennis  408 Sep 27 08:55 utils.pyc


Also there is a folder called http, which contains the following files:

    -rw-r--r-- 1 dennis dennis 2.0K Sep 27 08:55 exceptions.pyc
    -rw-r--r-- 1 dennis dennis 2.3K Sep 27 08:55 headers.pyc
    -rw-r--r-- 1 dennis dennis  112 Sep 27 08:55 __init__.pyc
    -rw-r--r-- 1 dennis dennis 2.4K Sep 27 08:55 query.pyc
    -rw-r--r-- 1 dennis dennis 5.8K Sep 27 08:55 request.pyc
    -rw-r--r-- 1 dennis dennis 2.4K Sep 27 08:55 response.pyc
    -rw-r--r-- 1 dennis dennis 3.2K Sep 27 08:55 status.pyc
    -rw-r--r-- 1 dennis dennis 1009 Sep 27 08:55 url.pyc

After some googling I found, that pyc files contain bytecode compiled by Python, which is executed by Python's VM. Opening the files with an editor let's us read the strings, but the bytecode is of course not readable. Searching for a tool do decompile these files I found uncompyle6, which can easily be installed via pip. After installing it and trying to decompile all the pyc files, I saw, that it failed on two of those. One beeing *__init__.pyc* in the root folder and the other one *request.py* in the *http* folder. So first, I looked at all the files, I was able to decompile. *main.py* does not seem like it contains any valuable information. The only thing I noticed was, that there was a typo in webtrawl (written as webtral). Not being able to look at *__init__.py*, I looked at *filter.py* next. It contains two unimplemented functions in the class *Filter* and a class named *BlacklistFilter*, that seems to check items from a request against a given blacklist. Not so interesting either. *utils.py* only contains a function that looks at a string and tries to translate it into a bool. If the string contains *'yes', 'true', 't', 'y'*, or *'1'* it returns *True*. If it contains *'no', 'false', 'f', 'n'* or *'1'* it returns *False*. If none of the above match, it raises an exception. *loghandler.py* defines how logs are named and how the rotation is handled. The naming scheme might be interesting later on. *cache.py* contains a class named *cache*. From a first view it reads a persistent file and saves hits. Whenever a hit is detected, it saves something called *items*, counts the number of hits and saves the time of the last hit using *datetime.now()*. Also an item might be able to be permanent. 

Next, let's look at the *http* folder. *exceptions.py* simply contains a list of possible HTTP status codes and other possible exceptions. *headers.py* seems to have something to do with HTTP headers. It describes the content type, a timestamp and some other things. *query.py* seems to take parameters from a request and split them in key value pairs. *response.py* assembles the headers for a response and the body and sends them back. *status.py* contains a list of HTTP status codes and checks for a valid HTTP version. *url.py* seperates the path and the query from a URL.

If we access webtrawl-test.fshbwl.ru we are greeted by a database login mask, which we need credentials for. Looking at the headers from the response we can see, that the content is gzip encoded and we get something called *x-magikarp-id* which has the value *fcc851c80287dfb0116c6aa1b0e36fdbca94f788*. Reloading the page does not alter this ID. After some trying around, I found the following things manipulating the ID:

- sending login data via the provided form
- changing the parameters appended to the URL
  
Changing User-Agent or IP address does not seem to change this ID. Using the same data always creates the same ID, which seems like it might be some kind of hash. Also there is a timestamp added. At this point I thought about trying to access the logs, since from *loghandler.py* we know how they are named. But the timestamp does not contain milliseconds and the logs do. So guessing the miliseconds and hoping to access the file might not be a good idea.

Since the request handling seems to be in *request.pys*, which uncompyle6 was not able to decompile I searched for some other methods doing this. I found *Decompyle++*, which contains a tool named *pycdas*, which is a disassembler for python bytecode and another decompiler named *pycdc*. *pycdc* was only able to decompile a small part of those two missing files. Here is the output for request.pyc as an example:
```python
    # Source Generated with Decompyle++
    # File: request.pyc (Python 3.7)

    import base64
    import hashlib
    import json
    import socket
    from webtrawl.http.exceptions import InvalidVerbException, HttpException, InvalidUrlException, HttpVersionNotSupported, HttpBadRequest, InvalidHeaderException, InvalidHostException, HttpNotImplemented, HttpBadGateway, HttpGatewayTimeout
    from webtrawl.http.headers import Headers
    from webtrawl.http.response import Response
    from webtrawl.http.url import URL

    class Request:
    Unsupported opcode: BUILD_CONST_KEY_MAP
        __module__ = __name__
        __qualname__ = 'Request'
        _CHUNK_SIZE = 1048576
        _UPSTREAM_CHUNK_SIZE = 1024
    # WARNING: Decompyle incomplete
```
So we can only see the imports. Using *pycdas* I was able to get the assembler code of these files. *__init__.pyc* disassembled into around 700 lines and *request.py* into around 1500 lines. In the assembler code we can see all functions with their variables and inputs. Now it was time to study the produced assembler code. The application class gave information about what happens, when a request is made.

    32      LOAD_CONST              5: ('strict', 'upstream')
    34      CALL_FUNCTION_KW        3
    36      STORE_FAST              7: request
    38      SETUP_EXCEPT            182 (to 222)
    40      LOAD_FAST               7: request
    42      LOAD_METHOD             3: parse
    44      CALL_METHOD             0
    46      POP_JUMP_IF_TRUE        54

It calls the *parse* function in *request* and if it returns *true* continues. If it fails, it raises a *HttpBadRequest* exception. 

    54      LOAD_FAST               5: filter
    56      POP_JUMP_IF_FALSE       74
    58      LOAD_FAST               5: filter
    60      LOAD_METHOD             5: validate_request
    62      LOAD_FAST               7: request
    64      CALL_METHOD             1
    66      POP_JUMP_IF_TRUE        74

Now *filter* is called and should validate the request. Since in our sample the filters are not implemented, it should execute the jump to 74. 

    74      SETUP_EXCEPT            68 (to 144)
    76      LOAD_GLOBAL             6: generate_response
    78      LOAD_FAST               7: request
    80      LOAD_FAST               4: cache
    82      CALL_FUNCTION           2
    84      STORE_FAST              8: response
    86      LOAD_FAST               5: filter
    88      POP_JUMP_IF_FALSE       106

Since the filters are not implemented it should jump to 106.

    106     LOAD_GLOBAL             8: log_request
    108     LOAD_FAST               3: logger
    110     LOAD_FAST               7: request
    112     LOAD_FAST               8: response
    114     CALL_FUNCTION           3
    116     POP_TOP                 
    118     LOAD_FAST               2: sout
    120     LOAD_METHOD             9: write
    122     LOAD_FAST               8: response
    124     LOAD_METHOD             10: send_bytes

So here the log is written and the response is sent back. In the assembly code we also find the following functions:

- log_request
- generate_response
- log_request

Here is an interesting part of `<listcomp>`, which is part of generate_response:

    12      LOAD_FAST               0: request
    14      LOAD_ATTR               0: headers
    16      LOAD_CONST              1: b'X-Magikarp-Id'
    18      BINARY_SUBSCR           
    20      STORE_FAST              3: cache_id
    22      LOAD_FAST               3: cache_id
    24      LOAD_CONST              2: b''
    26      COMPARE_OP              2 (==)
    28      POP_JUMP_IF_FALSE       40
    30      LOAD_DEREF              0: cache
    32      LOAD_METHOD             1: calculate_id
    34      LOAD_FAST               0: request
    36      CALL_METHOD             1
    38      STORE_FAST              3: cache_id
    40      LOAD_FAST               0: request
    42      LOAD_ATTR               0: headers
    44      LOAD_CONST              3: b'X-Magikarp-List'
    46      BINARY_SUBSCR           
    48      LOAD_CONST              4: b'true'
    50      COMPARE_OP              2 (==)
    52      POP_JUMP_IF_FALSE       94
    54      LOAD_GLOBAL             2: Response

It looks like it checks if the ID is set, and if not it calculates one. Again this uses *request*. It also looks for something called *X-Magikarp-List*. If it is true, it returns 200 and generates a different response. Manipulating the headers and inserting `X-Magikarp-List: true` gives us 400 server error. In the disassembled *request.pyc* we find the object *cache*. It seems to have the function to calculate the ID. It uses SHA1 for this. For the calculation it uses the following things:

- verb: the used method (POST or GET)
- url: the url that was used in the call
- version: the HTTP version used
- headers[Host]: the hostname
- headers[Authorization]: an unknown header field we might be able to send and manipulate
- body: message body

All these things are fed into a SHA1. So the *X-Magikarp-ID* is a SHA1 hash of the above stuff. Since we know all of the above, except what *Authorization* is, I searched for it but was not able to find it anywhere in the sources. Maybe we can set and send the *Authorization* header with our request. Doing this changes the *X-Magikarp-ID* on the server. At this point I was getting frustrated and started trying weird stuff. One of which was trying to bruteforce the contents of the ID using hashcat on my GTX1060. While trying to guess and replicate the hash to understand it better I moved on. While scrolling through the assembly, I found another interesting function, which checks the header size in the *parse* function. The buffer was set to 65537 bytes and the maximum allowed size for the header was 65536 bytes. Further down there is a check for newlines and carriege returns in the header. So I just tried to send large headers but the server gave an error way before reaching the maximum size. After this, I experimented with a lot of combinations of escape characters and newlines. This did also not lead to anything usable.

In the function *_parse_header* I stumbled across this:

    62      LOAD_CONST              8: b'X-Magikarp'
    64      COMPARE_OP              2 (==)
    66      POP_JUMP_IF_TRUE        84
    68      LOAD_FAST               2: name
    70      LOAD_CONST              0: None
    72      LOAD_CONST              7: 10
    74      BUILD_SLICE             2
    76      BINARY_SUBSCR           
    78      LOAD_CONST              9: b'x-magikarp'
    80      COMPARE_OP              2 (==)
    82      POP_JUMP_IF_FALSE       88
    84      LOAD_GLOBAL             3: InvalidHeaderException
    86      RAISE_VARARGS           1

The header is checked, if it contains either '*X-Magikarp*', or '*x-magikarp*' and if it does, an exception is raised. This might explain, why we were unable to send the *X-Magikarp-List* header. While wondering why they checked for it two times, I saw the differnce in capitalization. Maybe sending "*x-Magikarp*" or any other variation might work. Also, I remembered, that at some point the whole header was changed to be lowercase. Sending `x-Magikarp-List: true` did not work with the "edit and resend" feature of Firefox, but strangely enough it worked when I intercepted the call from chromium via BurpSuite. Repeating this failed in around 8/10 cases. The following header made it work sometimes:

    GET / HTTP/1.1
    Host: webtrawl-test.fshbwl.ru
    Connection: close
    Cache-Control: max-age=0
    Upgrade-Insecure-Requests: 1
    User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/76.0.3809.100 Safari/537.36
    Sec-Fetch-Mode: navigate
    Sec-Fetch-User: ?1
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3
    Sec-Fetch-Site: none
    Accept-Encoding: gzip, deflate
    Accept-Language: en-US,en;q=0.9
    x-Magikarp-list:true 

When it succeeded, I got a blank page with only text on it.

8b180c58e624d88879809de813187fab90cce00e;96;Wed Nov  6 19:29:24 2019
fcc851c80287dfb0116c6aa1b0e36fdbca94f788;6;Fri Nov  8 14:08:05 2019
843352a495223a4b5fe8d4dd35c1014395b71725;6;Fri Nov  8 14:08:06 2019
9aa3c13b57aeb838f39a2fbd30125436b458e29d;4;Fri Nov  8 14:07:20 2019

This seems to be a list of the cached IDs, since it lists an ID, number of hits and the last access time. The last three IDs only have a few hits and the second one is definitely the one for a normal request without antyhing added. The first one seems interesting. Since the assembler code suggests, that if you send a *X-Magikarp-ID* in the header it just uses this instead maybe we can do it with this ID. The following header was used: 

    GET / HTTP/1.1
    Host: webtrawl-test.fshbwl.ru
    Connection: close
    Cache-Control: max-age=0
    Upgrade-Insecure-Requests: 1
    User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/76.0.3809.100 Safari/537.36
    Sec-Fetch-Mode: navigate
    Sec-Fetch-User: ?1
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3
    Sec-Fetch-Site: none
    Accept-Encoding: gzip, deflate
    Accept-Language: en-US,en;q=0.9
    x-Magikarp-id: 8b180c58e624d88879809de813187fab90cce00e

The answer contains the flag:

    -----BEGIN FISHBOWL FLAG-----
    S8fV5W5LOGxQXavdDYubxhaVYG5cJX/my8ISCiaCqIvAqXyH
    DgF30JWig9CkESSlhLAtI7VFAaDdgaRVS76yt+riUwBTECp2
    Ah561czC7KT6SbEmD65bFsfyzKfe7W2acbaT29PajVWStjl7
    WEA=
    -----END FISHBOWL FLAG-----
