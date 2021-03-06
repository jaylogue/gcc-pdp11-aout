# GNU GCC 9.2.0 for PDP-11

This repo contains the `Dockerfile` and other files useful for compiling simple C programs for the PDP-11 line of
mini computers. Becuase PDP-11's are ancient technology, the real purpose is to compile programs for the PDP
machines emulated by [`simh`](http://simh.trailing-edge.com/).

This includes the [PiDP-11](https://obsolescence.wixsite.com/obsolescence/pidp-11)!

## Using the prebuilt image

The Docker image can be pulled from Docker Hub here: https://hub.docker.com/r/jameshagerman/gcc-pdp11-aout

Pull the image down using one of the tags:

```
docker pull jameshagerman/gcc-pdp11-aout:95392fa
```

The image isn't set up to have volume mounts or anything tricky so you'll want to run the Docker image so it drops you into a shell:

```
docker run -it jameshagerman/gcc-pdp11-aout:95392fa bash
```

Once you have a shell inside the container, you will be able to compile C programs!

*Note: Keep in mind, this is not your normal C world. Things are different in PDP land! If this is your first time
coding for the PDP-11, you've got some reading ahead of you...*


### GitHub Package Registry Woes

*Note: I ran into far too many stupid issues with GitHub's Package Registry so I've abandoned it. The existing image in GitHub's Registry will remain, but I suggest **NOT** using it for future projects.*


### Compiling the C example for PDP-11

*Note: There is also an untested ASM example in the image under `example-asm/`*

There is an extremely simple `foo.c` in the image that can be used to test the compiler as follows. 

```
root@14031c390c9f:/usr/local/lib# cat foo.c 
int start() { return 0; }
root@14031c390c9f:/usr/local/lib# pdp11-aout-gcc -nostdlib foo.c
root@14031c390c9f:/usr/local/lib# pdp11-aout-objdump -D a.out

a.out:     file format a.out-pdp11


Disassembly of section .text:

00000000 <_start>:
   0:	1166           	mov	r5, -(sp)
   2:	1185           	mov	sp, r5
   4:	0a00           	clr	r0
   6:	1585           	mov	(sp)+, r5
   8:	0087           	rts	pc
root@14031c390c9f:/usr/local/lib#
```

While that may get you a correctly compiled binary, the tricky is getting that binary running under `simh`.

Honestly, I'm still testing this process...

### Converting `a.out` formatted binaries to DEC LDA format for `simh`

The standard `a.out` file format generated by GNU GCC is not directly compatible with PDP-11/`simh` for a number
of reasons. This means that before the compiled binary can be executed, it need to be converted to an appropriate
format.

The `load` command offered by `simh` can be used to load a binary in the DEC LDA format - the old paper tape format!

There are a number of ways to do this conversion. Some are documented in various links down in the [References]
section.

One option is `/usr/local/lib/tools/atolda` provided in the Docker image. It is compiled from `atolda.c` in the
same directory as a part of the Docker build.

```
root@14031c390c9f:/usr/local/lib# ./atolda   
Usage: atolda file
root@14031c390c9f:/usr/local/lib# ./atolda a.out 
0000000      4 000000 02 /tmp/ccK4U5Ev.o
0000000     20 000000 42 _start
0000012     27 000012 42 __etext
0000012     35 000012 42 _etext
0000012     42 000400 44 __end
0000012     48 000400 42 __edata
0000012     56 000400 44 __bss_start
0000012     68 000400 42 _edata
0000012     75 000400 44 _end
START  = 000000
root@14031c390c9f:/usr/local/lib# ls
... a.out  a.out.lda ...
```

The LDA formated binary is now ready to be loaded by the `Absolute Loader`!

## Getting files in and out of the Docker image

There are a number of ways to do this. One is to start the Docker image using `docker run -it
gcc-pdp11-aout:working-v2 bash` and then use `docker cp` to get files in and out of the image:

```
# Get the name of your docker container
% docker ps | grep pdp                  
14031c390c9f        gcc-pdp11-aout:working-v2   "bash"                   About an hour ago   Up About an hour                               bold_einstein

# Copy a file in
% docker cp my-cool-app.c bold_einstein:/usr/local/lib/

# Copy a file out
% docker cp bold_einstein:/usr/local/lib/my-cool-app.lda .

# Copy your LDA tape file to your PiDP-11 (optional)
% scp ./my-cool-app.lda pidp11:~/
```

## Loading and running your program on the PiDP-11/`simh`/PDP-11

To load a program in the LDA format, you can use the `papertape` system configuration under the
`useful-simh-systems/` directory. The contents of that directory can be copied to the `/opt/pidp11/systems/`
directory on a `PiDP-11` install. The contents of the `useful-simh-systems/selections` file should be appended
to the end of the same file and checked to make sure the IDs of the various systems do not conflict.

The `papertape` system uses the bootstrap loader at the back of the PDP-11 Programming Card to load the Absolute
Loader tape. Once that tape has been loaded into memory it attaches the `display.lda` example to the `ptr` papertape
device, then executes the Absolute Loader to load the display register example into RAM.

As soon as the tape is loaded, the program is executed!

To run YOUR program, you'll probably be best off modifying the `useful-simh-systems/papertape/boot.ini` file to
load your LDA file.

The `blank` system just has a simple blink program in it. I use it to play with the console...

## Building the image

This will take a while. You'll be compiling `binutil` and `gnu gcc` from source.

*Note: It seems the GitHub docker registry requires you to use github tokens to authenticate correct. Also, only
use lower case tags.*

```
docker login
# Enter your Docker Hub username (not your email!) and password

docker build --tag jameshagerman/gcc-pdp11-aout:`git rev-parse --short HEAD` .
docker push jameshagerman/gcc-pdp11-aout:`git rev-parse --short HEAD`
```

## Resources for taking this further

Here are a few sites that were useful to me when building this and for learning more about the PDP-11 itself:

- https://xw.is/wiki/Bare_metal_PDP-11_GCC_9.2.0_cross_compiler_instructions
- http://docs.cslabs.clarkson.edu/wiki/Developing_for_a_PDP-11#Building_Software
- https://ancientbits.blogspot.com/2012/07/programming-barebones-pdp11.html
- https://github.com/jguillaumes/retroutils
- https://github.com/Grissess/pdp11-utils
- https://www.pcjs.org/apps/pdp11/tapes/absloader/
- https://skn.noip.me/pdp11/pdp11.html (minimal mirror here: https://static.zenpirate.com/nankervis-pdp11-js/pdp11.html and here: https://github.com/JamesHagerman/nankervis-pdp11-js in case anything happens to the original site)



