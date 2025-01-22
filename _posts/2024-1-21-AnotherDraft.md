---
layout: post
title: My First Real Project, RemoteAdmin.
---
# 1. First off, What Even Is a Command and Control (C2) Program?


A C2 "program" actually consists of two separate programs:

1. A client application that communicates with our command server. It receives commands, executes them locally, and returns the result to our server.
2. A server program that handles the active connections and provides us the IO to send commands and requests to the client program.

Generally speaking, the three chief priorities of a C2 are to:

    Persist (ensure it continues to run or reconnect automatically).
    Remain undetected (avoid antivirus or other security measures).
    Provide easy access and control for the C2 admin (so they can quickly administer the client device).

![sample image]({{site.baseurl}}/assets/images/c2diagram.png)

Above we can see the attacker using the C2 server side program, to interface with the client device program for command execution. Only difference from the diagram being that with my program, the client does not retrieve commands from the server, instead the server sends command requests over the TCP connection directly to the client. 



# 2. A Brief Look at the Remote Admin Server

In late August, I decided to take on my first project. The initial scope was simply to achieve a TCP connection between two machines. Gradually, with time, the scope expanded until the end goal became remote command execution, schedulable jobs, and a queryable logging database. At the moment, the noteworthy capabilities are:

    1. Ability to commands over TCP for the client program to interpret and execute
    2. A very basic logging system
    3. Support for hundreds of concurrent connected clients.
    4. Graceful disconnects.
    5. Name aliasing for client connections.  	

## 2.1 Current Logic Flow

The logic flow of Remote Admin in its current form is fully oriented around a loop:

    Querying the WinAPI for any inbound TCP connections on the server’s port.
	
    Iterating over a vector of objects. With each of these struct objects represents a single client.

These struct objects are the heart of Remote Admin’s functionality. The object contains various members that enable its reliable, albeit limited, range of functionality, and logging.

```javascript
struct clientSocketInformation
{
    std::string alias; //Allows for name support
    FD_SET socketOfClientAsSet; //FD_SET with only this socket
    SOCKET socketOfClient{};

    bool isReadable(FD_SET);
    bool isDisconnected(FD_SET, bool, int);
    void closeConnection(SOCKET&);

    std::vector<messageInformation> messages; //Logs messages with this client
    
    void log(clientSocketInformation& socketForKey, char serverSendingMsg[4096]);
    void logSent(clientSocketInformation, char storeThis[4096]);
    
    //Constructor
    clientSocketInformation(SOCKET socketOfClientInput)
    {
        FD_ZERO(&socketOfClientAsSet); //clear fd_set which will hold a single socket 
        socketOfClient = socketOfClientInput;
        FD_SET(socketOfClientInput, &socketOfClientAsSet);
    }

    clientSocketInformation() {}
};
```

As a brief aside, the logging system is currently comprised of each clientSocketInformation object owning a messages vector, whch is populated the two logging functionalities.

After you emotionally process the distress of seeing my memory-unsafe usage of C-style buffers, I would like to quickly mention that this struct, and the logging functionality is going to be replaced by a class, incorporating the many lessons learned so far.




# 3. Why Write a C2? That Sounds Way Too Complicated!

Writing a Command and Control system requires constant tinkering with sockets, which generally forces you to get comfortable with Windows programming concepts. The abundance of Stack Overflow discussions ensures you won’t drown in the sometimes esoteric Microsoft documentation. It cannot be understated how much more approachable the project is because of these secondary reference sources.

I hadn’t fully appreciated this luxury of the Winsock2 library until I recently found myself nearly pulling my hair out while working with the Kernel-Mode Driver Framework on a few projects. My only potential for reprieve was the occasional, semi-relevant, 20-year-old post on the OSR community forum!

> <i>“Anxiety is the dizziness of freedom.”</i>
 (Freedom courtesy of the Windows API) — Soren Kierkegaard

It is normal to feel absolutely lost in a sea of jargon when starting a project—raw sockets, endianness, Winsock2, Berkeley sockets—it can be overwhelming. That’s where I was, and thankfully the free Beej’s Guide to Network Programming saved my life[https://beej.us/guide/bgnet/]. It not only explains all the vital background theory of sockets but also gives you a strong footing with Winsock2 itself.

Also, if you’re someone who is comfortable occasionally leaning on generative AI to explain or troubleshoot code, you’ll be happy to know that the predominant LLMs are quite well-trained on Winsock2 and many adjacent WinAPI functions. So, worst-case scenario, you’ll never truly be left high and dry. With enough time, the Windows programming hurdles will become smaller and smaller, allowing you to gradually focus less on troubleshooting Windows functionalities and more on realizing your vision.



# 4. Creative Flexibility of C2 Development with the WinAPI

Once you’ve eventually established basic data transmission over TCP, the program is your oyster! Do you want to focus on designing a professional GUI? Or perhaps you’d prefer to enhance the program’s security, ensuring excellent handling of client transmissions and building a multi-admin system for collaborative use by security professionals. Alternatively, you might find more enjoyment in introducing polymorphic features—like a custom communication stack utilizing raw sockets. This can give you the granular control to (hopefully) communicate with a client without lighting up the SIEM like a Christmas tree.

With the WinAPI at your side, as either a friend or a foe, you have such a wide range of functionalities at your disposal that you can implement almost any functionality your mind can conceive… Although, taking off the rose-colored glasses, the WinAPI syntax, naming practices, and sheer number of parameters may “borrow” some of your sanity along the way.


![sample image]({{site.baseurl}}/assets/images/linuxVsWindows.jpg)

Oh, one could only dream....



# 5. Reflection on the Development Process

I've been fortunate to experience the misfortunes of developing Remote Admin with the WinAPI. Every large change made to the codebase represents a few sleepless nights with an ever-furrowed brow, followed by a few days of deep satisfaction… and an inevitable repeat of that cycle! But for some reason, I keep coming back to it. I initially started this project with the goal of zero dependencies other than the WinAPI, but unfortunately at the beginning I threw the baby out with the bath water, I bone-headedly avoided modern C++ features, this can be seen with the c-style buffers which still exist in my code. But unlike that entirely self-inflicted problem, I also had to remediate problems, and make changes that I had not anticipated.

##### Early Growing Pain 1: Client Handling

The first growing pain was an early-September change to how clients were handled. Initially, I used a simple C-style array to hold the socket file descriptors—these descriptors are handle-like representations of the connection. But I wanted the program to support resizing the array at runtime. That’s not possible with a C-style array, so I opted for the heap container std::vector.

##### Early Growing Pain 2: Transition to OOP

That was quickly followed by, an implemention of object-oriented programming into my code via the struct `clientSocketInformation` to hold not only the socket file descriptor but other data relating to each client. I feel like this was what took me from an absolute C++ beginner, to an absolute C++ beginner with an budding appreciation for very basic system design.

##### Early Growing Pain 3: Multithreading, and Mutexes

I remember the day I first saw this daunting task ahead of me: running my server’s client communication loop alongside a responsive server CLI. The only real solution was multithreading. While I already know the hardware-level basics of multithreading from an excellent operating systems fundamentals class, I was worried that knowledge wouldn’t be enough to properly implement concurrency. After watching a few hours of “safe C++ multithreading” videos, I took the plunge and started writing code. To my surprise, everything worked properly after only about an hour and a half! From this experience, I realized that we don’t just overestimate the difficulty of our problems—we also underestimate the value of our preparation.


The proceeding three months of other growing pains introduced me to multithreading, memory safety, and design patterns, and taught me a lot about the non-technical aspects of project design. Now, I am working to replace the struct (and the litany of direct member accesses) with a class. I’m looking forward to working with privatized class members and pushing myself to learn how to write a proper interface.

> <i>"If you have a well-designed interface, then one programmer can write the implementation for the class while other programmers write the code that uses the class. Even if you are the only programmer working on a project, you have divided one larger task, into two smaller tasks, which makes your program either to design and to debug"</i>
-- Walter Savitch (Absolute C++ Second Edition)



## 5.1 Future MySQL Integration & T-Shaped Knowledge

My next extension to the server side of RemoteAdmin is integrating MySQL for communication logging. This desire to expand the program’s capabilities also aligns with building T-shaped knowledge. If you’re not familiar with the concept of T-shaped knowledge, it emphasizes a strong and wide base of general understanding alongside deep specialization in a particular domain.

Hiring preferences in tech make it clear that we should prioritize for T-shaped learning, but what exactly would that look like in code? Well, if it’s true that “every painter paints themselves,” then the T-shaped engineer probably writes T-shaped programs! That’s also why a C2 is an excellent project: it necessarily demands a progression of understanding in socket programming, and soon you’ll feel the temptation to integrate multithreading, new features, encryption, GUIs, databases, or even file transferring capabilities. Before you know it, it’s 4 AM, and you’ve planned a very diverse development roadmap that will occupy the next three weeks.




# 6. Conclusion

In the end, it’s not all about the code itself—it’s about the process. A process that forces you to stretch, stumble (probably fall a few times), and yet emerge as a better developer. So go ahead, write that C2. And when you’re done, take a step back, admire the work you’ve put in… and then immediately spot the design flaws you hadn’t noticed before.

   > <i>“In filth it will be found.”</i>
    — Carl Gustav Jung

Each obstacle you overcome not only strengthens your technical skills but also grows your capacity as a problem solver. Although the process of creating something meaningful can be messy, chaotic, and confusing, it’s also incredibly rewarding.