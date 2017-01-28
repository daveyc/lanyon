---
layout: post
comments: true
tags: lua z/os
title: Using Lua to process MVS data sets.
---

[Lua](https://www.lua.org/) is a lovely little language. It's simple, powerful and insanely fast. It's also easy to extend with C/C++ modules.

The [Lua4z](https://www.lua4z.com/) port brings Lua to the z/OS mainframe operating system. One of the most imporant pieces of work for the port
was to patch the lua I/O library to fully support all the different types of MVS data set: Sequential, PDS, VSAM (ESDS, KSDS, RRDS) and z/OS Unix files.
Luckily this was not an onerous task as the C standard library `fopen()` supports all of the above. Some boiler plate code was added to support record based
I/O and voila!

## Sequential Data Sets

To read all the records in a sequential data set we use iterators. The `records` factory function is a z/OS extension that works just like
the standard Lua `lines` function but for record oriented files. The "type=record" parameter signifies that we want to use record I/O.
The "noseek" parameter selects the QSAM access method. If "noseek" is omitted the default BSAM access method is used. 
See the [z/OS fopen documentation](https://www.ibm.com/support/knowledgecenter/en/SSLTBW_2.1.0/com.ibm.zos.v2r1.bpxbd00/fopen.htm)
for a full description of the open mode parameters.

{% highlight lua %}
local hexdump = require "hexdump"
for rec in io.records("MY.BINARY.DATASET", "rb, type=record, noseek") do
  hexdump(rec)                                                     
end                                                               
{% endhighlight %}

Writing to a data set is just as simple. The following example shows how to write to a text file. Note the newline "\n" being appended to the end of the
string (C text files use newlines). Of course, we could also have opened the file in binary mode with "wb, type=record", but would have to take care to
explicitly pad the strings with spaces otherwise the records will be padded with binary zeroes. This is simple using a Lua4z string library extension 
to do the padding. To write to a file with LRECL(80) use `line:left(80)` which uses the `string.left` function inspired by REXX.

{% highlight lua %}
local f = assert(io.open("DD:SYSOUT", "w"))
                                                   
local lines = { "line1", "line2", "line3" }        
                                                   
for _, line in ipairs(lines) do                    
  f:write(line.."\n")                              
end                                                {% endhighlight %}

## Reading a PDS directory

Lua has an awesome feature called [coroutines](http://lua-users.org/wiki/CoroutinesTutorial), a form of co-operative mutli-threading. 
Coroutines are great for writing iterators, which is just what we need to read the PDS directory.

To print all members in a PDS we would run the following loop.

{% highlight lua nowrap %}
local pdsdir = require "pdsdir"

for member in pdsdir("MY.PDS") do 
  print(member) 
end
{% endhighlight %}

Here's the code that implements the PDS directory iterator. Note that it uses the [struct](http://www.inf.puc-rio.br/~roberto/struct/) library to read
and unpack the binary data from the PDS directory block. 

{% highlight lua nowrap %}
---
-- This sample demonstates how to use Lua co-routines to create 
-- a PDS directory iterator.
--
-- Usage: 
--
-- local pdsdir = require "pdsdir"
--
-- for member in pdsdir("MY.PDS") do print(member) end
local struct  = require "struct" 
 
local unpack = struct.unpack
local x = string.x2c

local function pds_dir_iterator(dsname)
  local EOF = x'FFFFFFFFFFFFFFFF' -- end of directory marker
  local endOfList = false
  -- for each directory block in the PDS
  local pds = assert(io.open(dsname, "rb"))                 
  repeat
    local rec = pds:read(256) -- read a directory block
    if not rec then break end
    local size, index = unpack("H",rec) -- size of the block
    -- for each directory entry in the block
    while index < size do
      local member, ttr, info, userData
      -- unpack the directory entry
      member, ttr, info, index = unpack("c8I3I1",rec, index)    
      if member == EOF then -- end of directory
        endOfList = true; break 
      end 
      local dataSize = bit32.band(info, 0x1F) * 2 
      if dataSize > 0 then
        userData, index = unpack("c"..dataSize, rec, index)
      end
      -- return the member name
      coroutine.yield(member:rtrim())
    end
  until endOfList
end 
  
-- return the PDS directory iterator 
return function (dsname)
  return coroutine.wrap(function() pds_dir_iterator(dsname) end) 
end
{% endhighlight %}


## VSAM

The dominant scripting language on z/OS is REXX. Unfortunately, REXX doesn't 
support VSAM and has only supported VBS data sets from z/OS 2.2. 


