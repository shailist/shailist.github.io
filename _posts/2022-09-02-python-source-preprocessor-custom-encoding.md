---
layout: post
title: Python Source Preprocessor (Custom Encoding)
date: 2022-09-02 23:53 +0300
---

Although you can pretty much do anything with the Python programming language, source code manipulation and metaprogramming isn't really a thing.  
In this article we are going to look at how we can make preprocess python source files and add our extensions to the language's syntax.

All the code used in this post and more can be found here:  
[https://github.com/shailist/python-preprocessor](https://github.com/shailist/python-preprocessor/){: target='_blank'}

# Example Usage

I wanted to implement a "vault" in python. That is, a python source file that can only used or accessed with the correct password.  
The password will be set using `#set_password`, and the protected area will be declared between `#vault_start` and `#vault_end`.  
To give an idea of what we are going to implement, lets look at an example:

```python
# coding: vault

print("I wonder what's inside...")

#vault_start
++lIZKsJQrv2pSIrhHAo9WGnOAEM+enhVW/V9il/HoKpaWjNLLAeSdqeveneqcJ+
#vault_end

```
{: file='secret.py'}

And to access the protected area:  

```python
# coding: vault

#set_password S3c4et!
import secret

print(f'The flag is: {secret.FLAG}')
```
{: file='main.py'}
```bash
$ python main.py
I wonder what's inside...
The flag is: {c0dIng-1s-Co0l}
```

The actual implementation can be found in the sources, but we will look at implementing something much simpler - reversed code:

```python
# coding: reverse-utf-8

)'})01(lairotcaf{ = )01(lairotcaF'f(tnirp

)1 - n(lairotcaf * n nruter    
1 nruter        
:0 == n fi    
:)n(lairotcaf fed
```
{: file='reversed.py'}
```bash
$ python reversed.py
Factorial(10) = 3628800
```
# # coding

The python language allows you to manually specify the encoding of a source file.  
In short, if the first or second line of code in your file look something like this:
```python
# coding: utf-8
# -*- coding=utf-8 -*-
# This file uses the following encoding: utf-8
```
then your source file will be decoded using `utf-8`.  
The key part is the `coding: <name>` pattern, but as you can see the format of this line is very flexible.  
You can read more about it inside [PEP 263](https://peps.python.org/pep-0263/).

# Registering a Custom Codec

When you tell python that you want to encode something with an encoding named `utf-8`, how does it know what codec to use?  
For some builtin encodings, this process is a part of the implementation, but for other encodings (or custom ones) it works differently.  
```
Custom codecs are made available by registering a suitable codec search function:

codecs.register(search_function)
    Register a codec search function. Search functions are expected to take one argument, being the encoding
    name in all lower case letters with hyphens and spaces converted to underscores, and return a CodecInfo
    object. In case a search function cannot find a given encoding, it should return None.
```
{: file='[Python Docs] codecs - Codec registry and base classes'}
So we can register our own search function, which takes an encoding name as an input, and returns a matching `CodecInfo` or `None`.  
Lets register a custom encoding, which will just be an alias for `utf-8`:

```python
from codecs import register, lookup

@register
def my_search_function(encoding_name):
    if encoding_name in ('my_encoding', 'my-encoding'):
        return lookup('utf-8')

print('Test'.encode('my-encoding'))
```
{: file='my_encoding.py'}
```bash
$ python my_encoding.py
b'Test'
```

> One caveat with the `encoding_name` we need to be aware of is that starting with Python 3.9 spaces and hyphens in encoding names are replaced with underscores before the lookup.  
> If we want the backwards compatibility we need to check all of these cases or manually replace these characters, but you can also just pick a single word encoding name ¯\\\_(ツ)\_/¯
{: .prompt-info }

# Custom Encoding 101

So we know how to register new encodings, but we still need to implement them.  
If we take a look at the `CodecInfo` class, we can see what do we need to implement:

```
class codecs.CodecInfo(encode, decode, streamreader=None, streamwriter=None,
                       incrementalencoder=None, incrementaldecoder=None, name=None)
    Codec details when looking up the codec registry.
    The constructor arguments are stored in attributes of the same name:

    name
        The name of the encoding.

    encode
    decode
        The stateless encoding and decoding functions. These must be functions or
        methods which have the same interface as the encode() and decode() methods
        of Codec instances (see Codec Interface). The functions or methods are
        expected to work in a stateless mode.
```
{: file='[Python Docs] codecs - Codec registry and base classes'}

As we can see, a codec is defined by its encoding and decoding functions.  
Lets implement a the previous encoding, but instead of using the `lookup` function, lets build the `CodecInfo` ourselves:

```python
from codecs import register, CodecInfo
from encodings import utf_8

@register
def my_search_function(encoding_name):
    # Starting with Python 3.9 hyphens are replaced with an underscores
    if encoding_name in ('my_encoding', 'my-encoding'):
        return CodecInfo(
            name='my-encoding',
            encode=utf_8.encode,
            decode=utf_8.decode
        )

print('Test'.encode('my-encoding'))
```
{: file='my_encoding.py'}
```bash
$ python my_encoding.py
b'Test'
```

By implementing our own `encode` and `decode` functions, we can process the data however we like.  
Just as an example, lets define a new encoding called `reverse-utf-8`, which will reverse the string before encoding it (and after decoding it):
```python
from codecs import register, CodecInfo
from encodings import utf_8

def reverse_utf_8_encode(input, errors='strict'):
    reversed = input[::-1]
    encoded, size = utf_8.encode(reversed, errors)
    return encoded, size

def reverse_utf_8_decode(input, errors='strict'):
    decoded, size = utf_8.decode(input, errors)
    un_reversed = decoded[::-1]
    return un_reversed, size

@register
def search_function(encoding_name):
    encoding_name = encoding_name.replace('_', '-')
    if encoding_name in 'reverse-utf-8':
        return CodecInfo(
            name='reverse-utf-8',
            encode=reverse_utf_8_encode,
            decode=reverse_utf_8_decode
        )

input = 'Hello world!'
encoded = input.encode('reverse-utf-8')
decoded = encoded.decode('reverse-utf-8')

print(f'Input: {input!r}')
print(f'Encoded: {encoded!r}')
print(f'Decoded: {decoded!r}')
```
{: file='reverse_utf_8.py'}
```bash
$ python reverse_utf_8.py
Input: 'Hello world!'
Encoded: b'!dlrow olleH'
Decoded: 'Hello world!'
```

You can pretty much do whatever you want in the `encode` and `decode` functions, but we will stick with simple reversing for now.

# Automatic Loading (PTH Files)

Right now we need to make sure our `@register`ed function is loaded before we try to use our encoding, but since we want to be able to use the encoding for source files, before they import anything, we need to somehow import our module before the code is decoded. I'm sure there are many ways to do it, but we'll take a look at `pth` files.

You can look up the documentation for `pth` files in the [site](https://docs.python.org/3/library/site.html) documentation, but I'll try to explain the parts that are interesting to us.  
If we put a file with the `.pth` extension in the `site-packages` directory, it will be processed at the startup of the Python environment, before any user code is loaded.  
What can `pth` files do? Many things, but in particular, if the file begins with `import`, its contents are executed as Python code.  
We can use this to load our module:

```python
import reverse_utf_8
```
{! file='reverse_utf_8.pth'}

Chuck this file into the `site-packages` folder and run our reversed file:

```bash
$ python reversed.py
SyntaxError: encoding problem: reverse-utf-8
```

Good luck debugging this error, it is not fun.  
The issue is that for source decoding, Python doesn't use the simple `encode` and `decode` functions. Instead it uses an incremental decoder.  
Taking a look at `utf_8.IncrementalDecoder`, we can write our own incremental decoder:

```python
# ...

class ReverseUTF8IncrementalDecoder(utf_8.IncrementalDecoder):
    def decode(self, input, final=False):
        self.buffer += input
        
        if final:
            buffer = self.buffer
            self.buffer = b''
            
            decoded, size = reverse_utf_8_decode(buffer)
            return decoded
        
        return ''

@register
def search_function(encoding_name):
    encoding_name = encoding_name.replace('_', '-')
    if encoding_name in 'reverse-utf-8':
        return CodecInfo(
            name='reverse-utf-8',
            encode=reverse_utf_8_encode,
            decode=reverse_utf_8_decode,
            incrementaldecoder=ReverseUTF8IncrementalDecoder
        )

# ...
```
{: file='reverse_utf_8.py'}

We just override the `decode` function of the `utf_8.IncrementalDecoder` class.  
We save every chunk of input that we get until we recieve the `final` chunk, and then we just call our decode function and return the result.  
We also modified the `search_function` to include our `ReverseUTF8IncrementalDecoder` in the `CodecInfo`.  

Now running our reversed file...
```bash
$ python reversed.py
Factorial(10) = 3628800
```

# Note for Packages

Making the `pth` file appear in the `site-packages` directory when our package is installed can be a bit troublesome.  
I've found using the `data_files` parameter to work the best:

```python
# ...
import sys
from distutils.sysconfig import get_python_lib

SITE_PACKAGES = get_python_lib().split(sys.prefix + os.sep)[1]

setup(
    # ...
    data_files=[(SITE_PACKAGES, ['reverse_utf_8.pth'])]
)

```
{: file='setup.py'}

Getting the `site-packages` by just using `get_python_lib()` isn't enough, since `wheel`s process paths differently.  
Converting the path to be relative to the `sys.prefix` solves the issue.

# Conclusion

I had a lot of trouble finding resources on the topics covered in this post, but I think this concept is very cool and can't wait to see what pepole can come up with using this.  
Big thanks to [@jonatan1609](https://github.com/jonatan1609) for coming up with the concept and doing the original research, and shoutout to all the people in the forward chain that led me to discovering this :)

