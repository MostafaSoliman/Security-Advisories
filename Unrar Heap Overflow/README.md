Introduction
------------------------
I decided to fuzz unrar.exe which is part of the winrar suite, the command line used to extract rar files.
It is available for download from here
```
https://www.rarlab.com/rar/wrar560.exe
```
Such famous tool must have been fuzzed by a lot of researchers, so I decided to look for another channel to fuzz other than the input rar file.

Finding another channel
------------------------
Using process monitor from sysinternal suite, we can see that unrar.exe is reading a configuration file from ```rar.ini``` which is supposed to be in the same directory.
![Alt text](images/1.PNG?raw=true)

Fuzzing the target
-----------------
Since this is file based fuzzing, the best fuzzer to use in this case is WinAFL ```https://github.com/ivanfratric/winafl```.
The command line in this case was as follow.
```
afl-fuzz.exe -f D:\fuzzing\fuzzing-stage\unrar\WinRAR\rar.ini -i D:\fuzzing\fuzzing-stage\unrar\input -o D:\fuzzing\fuzzing-stage\unrar\output-ini -D d:\fuzzing\DynamoRIO-Windows-7.0.17651-0\bin32 -t 20000+ -- -coverage_module unrar.exe -target_module unrar.exe -target_offset 0x17E40 -nargs 2 -fuzz_iterations 5000 -- D:\fuzzing\fuzzing-stage\unrar\WinRAR\unrar.exe p D:\fuzzing\fuzzing-stage\unrar\input\rar2.rar

```
For more info on how to use WinAFL review the tool GitHub page.

The initial input file was containing a simple command line option like below.
```
switches=-cl -m5 -s
switches=-m5 -s
```

Fuzzing results
---------------
I managed to crash ```unrar.exe``` multiple time, all the crashes had the same root cause. The attached ```unrar.ini``` will trigger the crash.


The crash
----------
The application crashes while executing the ```CompareStringW``` function, after copying the file content to the heap. 
![Alt text](images/2.PNG?raw=true)

So the first thing to inspect was the application heap.
![Alt text](images/3.PNG?raw=true)

Then let's inspect the heap starts at ```0x00600000```
![Alt text](images/4.PNG?raw=true)
![Alt text](images/5.PNG?raw=true)
![Alt text](images/6.PNG?raw=true)

As seen from the last figure the last heap segment (Free Fill) seems to be corrupted, it says that its ```prevSize``` is ```1ed48``` while it should be ```0a600```.
So let's inspect this segment. As shown it has been written over by the value ```005c``` which is the wide char of the ASCII char ```/```, this is the payload in the file that trigger the crash.
![Alt text](images/7.PNG?raw=true)

So lets cast this free segment to ```_HEAP_ENTRY```, to see how the overflow data will be interpreted by the heap manager.
![Alt text](images/8.PNG?raw=true)


Refrences
----------
1. https://github.com/ivanfratric/winafl
2. http://www.informit.com/articles/article.aspx?p=1081496&seqNum=2
