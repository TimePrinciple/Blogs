## UNIX Architecture

In a strict sense, an operating system can be defined as the software that controls the hardware resources of the computer and provides an environment under which programs can run. Generally, we call this software the *kernel*, since it is relatively small and resides at the core of the environment.

The interface to the kernel is a layer of software called the *system calls*. Libraries of common functions are built on top of the system call interface, but applications are free to use both. The shell is a special application that provides an interface of running other applications.

In a broad sense, an operating system consists of the kernel and all the other software that makes a computer useful and gives the computer its personality. This other software includes system utilities, applicaions, shells, libraries of common functions, and so on.

> Linux is the kernel used by the GNU operating system. Some people refer to this combination as the GNU/Linux operating system, but it is more commonly referred to as simply Linux. Although this usage may not be correct in a strict sense, it is understandable, given the dual meaning of the phrase *operating system*. (It also has the advantage of being more succinct)

## Login Name

When logging in to a UNIX system, we enter our login name, followed by the according password. The system then looks up out login name in its password file, usually the file `/etc/passwd`. Every entry is composed of seven colon-separated fields:
```
{login name}:{encrypted password}:{numeric user ID}:{numeric group ID}:{comment field}:{home directory}:{shell program}

sar:x:205:105:Stephen Rago:/home/sar:/bin/bash
```
