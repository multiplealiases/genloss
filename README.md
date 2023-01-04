# genloss

Induce generation loss on images via iterated rotation and saving,
or "spin to win", image edition.

# Usage

## Self-plagiarized:

```
./genloss [options] filename
```

Requires one of ImageMagick or GraphicsMagick.
For GM, alias `gm convert` to `convert`,
and `gm identify` to `identify` to run this script verbatim, or edit the
script.
On Debian, the package `graphicsmagick-imagemagick-compat` does this for you.
Your particular distribution may have equivalents.

Written for Bash and GNU `coreutils`.

Tested on Fedora 36, Raspbian Testing, and Termux.

Oh, wait, you wanted more? 

In lieu of the more traditional "this is what it does" explanations 
that pervade docs, I've instead decided to show a rough cookbook of ways 
I've used this script, *why* these options exist the way they are, 
and why I chose to implement this "spin to win" tactic, instead of 
letting it be still.

## Using `genloss` effectively 

The main way I use `genloss` is through GNU Parallel and on a RAM disk.

I suggest mounting a RAM disk, just because you're doing lots of tiny writes
to your disk, and you might get bottlenecked by it. 

### Mounting a RAM disk (Linux only?)

```
# mount -t tmpfs -o size=2G,noatime,nodev,noexec,mode=600 ramdisk /mnt
$ cp image.png /mnt
$ cd /mnt
```

Adjust the `size=` parameter to taste. `noatime` is... probably unnecessary,
but you may as well state it explicitly. `nodev`, `noexec`, and `mode=600`
prevents device files from working, prevents execution of binaries, and sets 
all files within it to `chmod 600` (read-write for user, nothing for anyone else)
respectively.

`ramdisk` is just the name of the new filesystem, and `/mnt` the mount point,
which you're free to change at your discretion. If you'd like to break FHS,
go right ahead and `mkdir /ramdisk` and use that as the mount point. (Don't.)

### Using it with `parallel`

It's not exactly fun to do just one quality setting, nor does the script
make full use of all available cores (blame ImageMagick and GraphicsMagick),
so what I do is iterate over them, like so:

```
$ parallel genloss --quality {} image.png ::: $(seq 0 10 100) 
```

(if you're wondering what image formats are supported, it's everything that
your local copy of ImageMagick/GraphicsMagick supports, since it piggybacks off
their capabilities. This is effectively "all of the ones you've heard of", unless
your copy is absurdly minimal.)

You use lots of disk space this way, but you did just mount a tmpfs, didn't
you?

#### Multiple images

Maybe you'd like to process multiple images.

```
$ ls
image1.png image2.jpeg snarf.webp
$ parallel genloss {} ::: image1.png image2.jpeg snarf.webp 
```

#### List of images

What about a list of images?

```
$ cat list
image1.png
image2.jpeg
snarf.webp
~/Pictures/faarf.heic
-oh-damn-you.jpeg
--double-damn-you.jp2
$ parallel genloss -- {} :::: list
```
Note: `:::` and `::::` aren't the same. `:::` is roughly equvalent to
`<command> | parallel`, which you can also do, but `::::` reads input from
the files provided after it.

`--` disables argument parsing for `genloss`, useful for problematic file names
that start with `-`, otherwise `-oh-damn-you.jpeg` and `--double-damn-you.jp2`
will get parsed as if they're options called `-oh-damn-you.jpeg`, and
`--double-damn-you.jp2`.

#### Over images and quality settings

```
$ parallel genloss {2} --quality {1} ::: seq 0 1 100 ::: image1.png image2.jpeg image4.webp
```

This is probably hilariously overkill, but I don't care how you use this thing,
so long you're following the MIT license.

### Making "timelapses" out of the output

This isn't at all covered by `genloss` (though it probably should!), but it's 
included here because I do this. For this, you'll need FFmpeg in your PATH.

To make timelapses in a lossless format, for instance, you'd do something like this:

```
ffmpeg -i filename-quality-%d.jpeg -c:v ffv1 output-lossless.mkv
```
2 things to note:

* `%d` on the filename is a feature that works analogously to C's printf 
   format specifiers -- you use it to indicate that it's a **d**igit that
   goes up from 0 to however high it goes.

* `ffv1` is FFV1, a lossless video format that's more efficient than HuffYUV.

You could use H.264/H.265 (`libx264`/`libx265`) in their lossless modes, too.
I just like FFV1 better because I don't have to memorize all of
`-crf 0`/`-qp 0`/`-x265-params lossless=1` to get lossless
H.264 8-bit/H.264 10-bit/H.265 working. All of them work nicely, though.

Regardless, I prefer to keep my timelapses in lossless video formats because
all those lossless formats produce a smaller file than if I just kept the
raw images as-is. Plus, they're neat to look at.

#### Dumping the individual frames of a "timelapse"

```
ffmpeg -i timelapse.mkv timelapse-%d.png
``` 

This takes the frames within the timelapse and converts them to PNGs named
`timelapse-1.png`, `timelapse-2.png`, `timelapse-3.png`, and so on.

The reason why I chose PNG (or any lossless image format FFmpeg supports)
is because choosing JPEG or lossy WebP will cause the lossless frames inside
the lossless timelapse to be lossily compressed on the way out. This is
generally considered "less than ideal".

### The transparency hack

The format that `genloss` defaults to is JPEG. Unfortunately, it doesn't support
transparency. Not even remotely, unless you like carrying two JPEGs
around per image and doing some image magick (hah!) to combine them later.

If you'd like to keep the transparency, what I'd suggest instead is to use
a more modern format like WebP (pass `--format webp --background transparent`
to `genloss` to do this). It degrades faster and differently, though.

There are others! JPEG 2000 (`jp2`) and JPEG XL (`jxl`) do have
transparency support, but they're less likely to be supported -- the
repos for Fedora 36, for instance, package an ImageMagick without
support for JPEG XL.

Your mileage might vary, and if all else fails: compile your own.

## Explaining the "why?" of the options:

### `-n` and `--number` 

This was included because it's generally hard to predict how images will
degrade, *a priori*, as well as how fast you'll run out of disk/memory
(if you use a RAM disk as scratch space).

Let's make the former case clearer with an example from my own use:

> WebP (use `--format webp`) and JPEG degrade at different "rates", 
> for a particularly loose and intuitionistic definition of "rate", if you will.
> If JPEG at 80% quality gets mangled to unrecognizability in, say, 1500 iterations, 
> then WebP at the same quality might do that in, say... 500.
>
> If that's the case, why should I sit around generating 2500 "frames" for 
> 2000 "frames" of garbage, in the case of WebP? It's a bit of a waste of time,
> and as much as we'd like to believe that compute is cheap, time sure as
> hell isn't.

I'd like to hope that you understand that higher resolutions mean higher
disk/memory usage, especially in tandem with the format case.

###


# Rambling below!

## Wait, why?!

There's this phenomenon called "generation loss", where if you put some 
arbitrary media (visuals, sound) through a lossy (as opposed to a lossless) 
process over and over, you'll end up with a degraded copy of it.

I'm not going to embed media for this, I'll just point you to
[the Wikipedia article on generation loss](https://en.wikipedia.org/wiki/Generation_loss),
as well as 
[this Reddit post of a YouTube video](https://www.reddit.com/r/programming/comments/4dg2t5/generation_loss_comparison_of_flif_webp_and_jpeg/) 
of Jon Sneyers, who showed this phenomenon for digital images.

Now, I wasn't too pleased with 
[the script he published for this](https://www.reddit.com/r/programming/comments/4dg2t5/comment/d1qwwk8/). 
Fair play to him, I'm sure it would generate roughly the same results that
he showed off, but it's not a tool as much as it is a presentation generator.

Thus: whatever the hell my script is.
