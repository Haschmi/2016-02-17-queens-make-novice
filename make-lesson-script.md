---
layout: page
title: Make
subtitle: Lesson script
minutes: 90
---

# Make

One of the things you might find while programming is that you end up making a whole bunch of scripts to a whole bunch of little jobs. How do you tie them together to make a workflow?

`make` helps solve this- it's an automation tool that ties things together.

A couple common uses:
+ Automating a workflow
+ Compiling code
+ Combine scripts and figures to create papers

You're not required to use it or anything, like Git, this is a tool that you can use when you think it might be useful.

## Introduction

This folder is what we'll be working with for this tutorial. It contains a couple complete books (in the books/ directory) and several Python scripts to generate statistics based on the contents of the books.

> Make sure students are using the anaconda3 usepackage

+ Take a look at the contents of the books
+ `python wordcount.py books/isles.txt isles.dat`
+ `python plotcount.py isles.dat show`
+ `python plotcount.py isles.dat isles.jpg`

We're going to try using make to automate this process and pack up the results nicely for us

## Makefile basics

Start editing Makefile, explain what targets/dependencies are, the tabbed commands are just shell commands.

```{.bash}
# Count words
isles.dat : books/isles.txt
        python wordcount.py books/isles.txt isles.dat
```

```{.bash}
make isles.dat
ls
```

Make sure people are using tabs instead of spaces

Try running `make isles.dat` again, does it do anything?

How does make know when stuff is up-to-date? It actually checks the time each file was last modified- if the dependencies are more recent than the target, it remakes the target.

```
touch books/isles.txt
# ls -l both files
make isles.dat
```

Add a second rule
```{.bash}
abyss.dat : books/abyss.txt
        python wordcount.py books/abyss.txt abyss.dat
```

```{.bash}
make abyss.dat
ls
```

What if we want to clean up after ourselves? Let's create a target called "clean" that will clean up all the files we make...

```{.bash}
clean :
	rm -f *.dat
```

```{.bash}
make clean
```

What happens if we make a file called clean?
```{.bash}
touch clean
make clean
```

`make` thinks that "clean" makes a file called `clean`. How can we tell it that "clean" doesn't actually make anything?

At the top of our Makefile (or anywhere) we tell `make` that "clean" is a phony target.

```{.bash}
.PHONY : clean
```
```{.bash}
make clean
ls
```

Typing `make isles.dat` then `make abyss.dat` seems inefficient. Let's create another target that makes both of them.

```{.bash}
.PHONY : clean dats
dats : abyss.dat isles.dat
```
```{.bash}
make dats
```

Student's Makefiles should look like this:
```{.bash}
.PHONY : clean dats

dats : isles.dat abyss.dat

isles.dat : books/isles.txt
        python wordcount.py books/isles.txt isles.dat

abyss.dat : books/abyss.txt
        python wordcount.py books/abyss.txt abyss.dat

clean :
        rm -f *.dat
```

> ### Writing custom rules {.challenge}
> Write some more rules:
> 
> + Write a new rule for last.dat, created from books/last.txt.
>
> + Update the dats rule with this target.
>
> + Write a new rule for analysis.tar.gz, which creates an archive of the data files. The rule needs to:
> 	+ Depend upon each of the three .dat files.
> 	+ Invoke the action tar -czf analysis.tar.gz isles.dat abyss.dat last.dat
>
> + Update clean to remove analysis.tar.gz.

## Automatic variables

Student's stuff should now look like this:
```{.bash}
.PHONY : clean dats

dats : isles.dat abyss.dat last.dat

isles.dat : books/isles.txt
    python wordcount.py books/isles.txt isles.dat

abyss.dat : books/abyss.txt
    python wordcount.py books/abyss.txt abyss.dat

last.dat : books/last.txt
	python wordcount.py books/last.txt last.dat

analysis.tar.gz : isles.dat abyss.dat last.dat
	tar -czf analysis.tar.gz isles.dat abyss.dat last.dat

clean :
        rm -f *.dat
```

We've been duplicating a lot of code and typing the same words twice. Let's try and avoid this.

+ We can refer to a target with `$@`
+ `$^` means "all dependencies"

Rewrite the analysis.tar.gz part
```{.bash}
analysis.tar.gz : isles.dat abyss.dat last.dat
        tar -czf $@ $^
```
```{.bash}
make clean
make analysis.tar.gz
```

We can even use bash wildcards in our dependency list:
```{.bash}
analysis.tar.gz : *.dat
        tar -czf $@ $^
```
```{.bash}
touch books/*.txt
make analysis.tar.gz
```

But there's a problem: notice how this *doesn't* work:
```{.bash}
make clean
make analysis.tar.gz
```

If you use wildcards and there aren't any of the file type specified, **make won't like it**. Let's change things back.

```{.bash}
analysis.tar.gz : isles.dat abyss.dat last.dat
        tar -czf $@ $^
```

> ### The `$<` variable {.challenge}
> Whereas `$^` means all dependencies `$<` simply means the first dependency.
>
> Rewrite all of our .dat rules to use `$@` and `$<`

Our .dat rules actually depend on two things: the actual book files as well as the `wordcount.py` script used to create them.

Add `wordcount.py` as a dependency to all of the .dat rules

Also, I'm feeling really lazy. What happens if we simply type `make` with no arguments?

```{.bash}
make clean
make
```

When make is run with no arguments it makes the first rule that doesn't begin with a `.` (so the `.PHONY` stuff doesn't count). If we want to make a default rule, the convention is to call it `all`. 

Let's make a phony target called `all` that depends on analysis.tar.gz.

```{.bash}
.PHONY : clean dats all

all : analysis.tar.gz
```
```{.bash}
make clean
make
```

## Pattern rules

We've still got a ton of redundancy. Why do we have three rules that pretty much all do the same thing? Is there a way of using wildcards in a non-buggy way in `make`.

`%` functions as `make`'s wildcard. Let's rewrite the .dat rules to be non-redundant.

Another special variable: `$*`. `$*` is a filename minus its extension. There's no real difference between it and `$*.dat` and `$@` in this scenario (just teaching something extra!).

```{.bash}
%.dat : books/%.txt wordcount.py
	python wordcount.py $< $*
```
```{.bash}
make clean
make
```

## Variables and functions

Our Makefile should now look like this:
```{.bash}
.PHONY : all clean

all : analysis.tar.gz

%.dat : books/%.txt wordcount.py
	python wordcount.py $< $@

analysis.tar.gz : isles.dat abyss.dat last.dat
	tar -czf $@ $^

clean :
	rm -f *.dat
	rm -f analysis.tar.gz

```

For maximum laziness, let's see if we can write down all of our intended targets in a variable.

```{.bash}
DATS=isles.dat abyss.dat last.dat

analysis.tar.gz : $(DATS)
	tar -czf $@ $^
```

But my fingers are getting tired! Isn't there any way we can figure out the names of the .dat files automatically? We can use some functions to do this:
```{.bash}
TXTS=$(wildcard books/*.txt)
DATS=$(patsubst books/%.txt, %.dat, $(TXTS))

analysis.tar.gz : $(DATS)
	tar -czf $@ $^
```

> ### Putting it all together {.challenge}
> Edit your Makefile so that:  
> + You create a variable called JPGS that contains the name of every .jpg file you'll make
> + You make a .jpg for every .dat file 
> + The .jpgs are put into the analysis.tar.gz file
> + The .jpgs get cleaned up when you use `make clean`

## Final notes

Answer from last question:

```{.bash}

```

`make` has a habit of cleaning up files when it's done sometimes. Let's demonstrate this.

```{.bash}
all : $(JPGS)
```
```{.bash}
make clean
make
```

Sometimes those intermediate files took hours to make and we don't want to lose them! We can avoid this by adding `.SECONDARY :` to our Makefile!

Our final Makefile:
```{.bash}
.PHONY : all clean
.SECONDARY :

TXTS=$(wildcard books/*.txt)
DATS=$(patsubst books/%.txt, %.dat, $(TXTS))
JPGS=$(patsubst %.dat, %.jpg, $(DATS))

all : analysis.tar.gz

%.dat : books/%.txt wordcount.py
	python wordcount.py $< $@

%.jpg : %.dat plotcount.py
	python plotcount.py $< $@

analysis.tar.gz : $(DATS) $(JPGS)
	tar -czf $@ $^

clean :
	rm -f *.dat *.jpg analysis.tar.gz
```

> ## Interpreting other peoples' Makefiles {.challenge}
> See if you can understand the following Makefile:
> ```{.bash}
> CC=gcc
> CFLAGS=-g -Wall -O2 -Wno-unused-function
> 
> all:seqtk trimadap
> 
> seqtk:seqtk.c khash.h kseq.h
> 		$(CC) $(CFLAGS) seqtk.c -o $@ -lz -lm
> 
> trimadap:trimadap.c kseq.h ksw.h
> 		$(CC) $(CFLAGS) ksw.c trimadap.c -o $@ -lz -lm
> 
> clean:
> 		rm -fr gmon.out *.o ext/*.o a.out seqtk trimadap *~ *.a *.dSYM session*
> ```

