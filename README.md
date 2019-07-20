gztool
======

GZIP files indexer and data retriever.
Create small indexes for gzipped files and use them for quick and random data extraction.
No more waiting when the end of a 10 GiB gzip is needed!

Considerations
==============

* Please, note that the initial index creation still consumes as much time as a complete file decompression.   
Once created the index will reduce this time.

* Index size is approximately 5% or less of compressed gzip file. The bigger the gzip, the better the proportion.

**There are optimizations to these considerations planned for coming versions !**

Background
==========

By default gzip-compressed files cannot be accessed in random mode: any byte required at position N requires the complete gzip file to be decompressed from the beginning to the N byte.   
Nonetheless Mark Adler, the author of [zlib](https://github.com/madler/zlib), provided years ago a cryptic file named [zran.c](https://github.com/madler/zlib/blob/master/examples/zran.c) that creates an "index" of "windows" filled with 64 kiB of uncompressed data at different positions along the un/compressed file, which can be used to initialize the zlib library and make it behave as if data begin there.   

This work builds upon zran.c to provide a useful command line tool.   
Also, some optimizations has been made:

* an *ex novo* index file format has been created to store the index
* windows are compressed in file
* windows are not loaded in memory unless they're needed
* data can be provided from/to stdin/stdout

More functionality is planned.

Compilation
===========

    gcc -O3 -o gztool gztool.c -lz

Compilation in Windows
======================

Compilation in Windows is possible using *gcc* for Windows and compiling the original *zlib* code to obtain the library.    
Please, note that executables for different platforms are provided on the Release page.    

* download gcc for Windows: [mingw-w64](https://sourceforge.net/projects/mingw-w64/files/latest/download)
* Install it and add the path for gcc.exe [to your Windows PATH](https://www.computerhope.com/issues/ch000549.htm)
* Download zlib code and compile it with your new gcc: [zlib](https://github.com/madler/zlib/archive/master.zip)
* The previous step generates a **zlib.a** that you need to compile *gztool*:
Copy gztool.c to the directory where you compiled zlib, and do:
    
    `gcc -static -O3 -I. -o gztool gztool.c libz.a`


Usage
=====

      $ gztool [-b #] [-cdefhil] [-I <INDEX>] <FILE> ...

     -b #: extract data from indicated byte position number
          of gzip file, using index
     -c: raw-gzip-compress indicated file to STDOUT
     -d: raw-gzip-decompress indicated file to STDOUT
     -e: if multiple files are indicated, continue on error
     -f: with `-i` force index overwriting if one exists
         with `-b` force index creation if none exists
     -h: print this help
     -i: create index for indicated gzip file (For 'file.gz'
         the default index file name will be 'file.gzi')
     -I INDEX: index file name will be 'INDEX'
     -l: list info contained in indicated index file


Examples of use
===============

Make an index for test.gz. The index will be named test.gzi:

    $ gztool -i test.gz

Make an index for test.gz with name 'test.index'

    $ gztool -I test.index test.gz

Retrieve data from uncompressed byte position 1000000 inside test.gz:

    $ gztool -b 1000000 test.gz

In this latter case, if index hasn't yet been created the program will complain and stop. But index creation can be `forced` if it does not exist yet:

    $ gztool -fb 1000000 test.gz

Index file format
=================

Index files are created by default with extension '.gzi' appended to the original file name of the '.gz' file:

    filename.gz     ->     filename.gzi

There's a special header to mark ".gzi" files as index files usable for this app:

    +-----------------+-----------------+
    | 0x0 64 bits     |    "gzipindx"   |
    +-----------------+-----------------+

Note that this header has been built so that this format will be "compatible" with index files generated for *bgzip*-compressed files. **[bgzip](http://www.htslib.org/doc/bgzip.html)** files are totally compatible with gzip: they've just been made so every 64 kiB of uncompressed data the zlib library is restart, so they are composed of independent gzipped blocks one after another. The *bgzip* command can create index files for bgzipped files in less time and with much less space than with this tool as they're already almost random-access-capable. The first 64-bit-long of bgzip files is the count of index pairs that are next, so with this 0x0 header gztool-index-files can be ignored by *bgzip* command and so bgzipped and gzipped files and their indexes could live in the same folder without collision. In the next version *gztool* will also ignore *bgzip*-index files (and in a future version *gztool* will hopefully read both index file types).

All numbers are stored in big-endian byte order (platform independently). Big-endian numbers are easier to interpret than little-endian ones when inspecting them with an hex editor (like `od` for example).

Next, and almost one-to-one pass of *struct access* is serialize to the file. *access->have* and *access->size* are both written even though they'd always be equal. Nonetheless one of them could be used in the future to add new functionality (some is in preparation).

After that, comes all the *struct point* data. As previously said, windows are compressed so a previous register (32 bits) with their length is needed.

After all the *struct point* structures data, the original uncompressed data of the gzipped file is stored (64 bits).

Please note that not all the data is 64-bit long. This is because some counters will always fit in less length. Refer to code.

Version
=======

This version is **v0.1**.

Please, read the *Disclaimer*. This is still a beta release. In case of any errors, please open an *Issue*.

License
=======

    A work by Roberto S. Galende
    distributed under the same License terms covering
    zlib from Mark Adler (aka Zlib license):
      This software is provided 'as-is', without any express or implied
      warranty.  In no event will the authors be held liable for any damages
      arising from the use of this software.
      Permission is granted to anyone to use this software for any purpose,
      including commercial applications, and to alter it and redistribute it
      freely, subject to the following restrictions:
      1. The origin of this software must not be misrepresented; you must not
         claim that you wrote the original software. If you use this software
         in a product, an acknowledgment in the product documentation would be
         appreciated but is not required.
      2. Altered source versions must be plainly marked as such, and must not be
         misrepresented as being the original software.
      3. This notice may not be removed or altered from any source distribution.

    /* zlib.h -- interface of the 'zlib' general purpose compression library
      version 1.2.11, January 15th, 2017
      Copyright (C) 1995-2017 Jean-loup Gailly and Mark Adler
      This software is provided 'as-is', without any express or implied
      warranty.  In no event will the authors be held liable for any damages
      arising from the use of this software.
      Permission is granted to anyone to use this software for any purpose,
      including commercial applications, and to alter it and redistribute it
      freely, subject to the following restrictions:
      1. The origin of this software must not be misrepresented; you must not
         claim that you wrote the original software. If you use this software
         in a product, an acknowledgment in the product documentation would be
         appreciated but is not required.
      2. Altered source versions must be plainly marked as such, and must not be
         misrepresented as being the original software.
      3. This notice may not be removed or altered from any source distribution.
      Jean-loup Gailly        Mark Adler
      jloup@gzip.org          madler@alumni.caltech.edu
      The data format used by the zlib library is described by RFCs (Request for
      Comments) 1950 to 1952 in the files http://tools.ietf.org/html/rfc1950
      (zlib format), rfc1951 (deflate format) and rfc1952 (gzip format).
    */

Disclaimer
==========

**This software is provided "as is", without warranty of any kind, express or implied. In no event will the authors be held liable for any damages arising from the use of this software.**

Author
======

by [Roberto S. Galende](loopidle@gmail.com)   

on code by Mark Adler's [zlib](https://github.com/madler/zlib).