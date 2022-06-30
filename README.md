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
and `gm identify` to `identify` to run this script verbatim.
On Debian, the package `graphicsmagick-imagemagick-compat` does this for you.
Your particular distribution may have equivalents.

Written for Bash and GNU `coreutils`.
(While written on a GNU `coreutils` system, ~~I don't want it 
to depend on it. Consider non-portability a bug, not a feature.~~ 
lol, no, not unless POSIX has a long-accepting `getopt`.)

Tested on Fedora 36, Raspbian Testing, and Termux.

Oh, wait, you wanted more? 

In lieu of the more traditional "this is what it does" explanations 
that pervade docs, I've instead decided to show a rough cookbook of ways 
I've used this script, *why* these options exist the way they are, 
and why I chose to implement this "spin to win" tactic, instead of 
letting it be still.

## Using `genloss` effectively 

The main way I use `genloss` is through GNU Parallel and on a RAM disk.

Install GNU Parallel from your package manager-- the package you want
is usually called `parallel`, or some variation of that.

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

You use lots of disk space this way, but mounting a tmpfs (Linux only, other
OSes may have equivalents) is "free", assuming you have the spare RAM.

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

And proceed as usual.

Maybe you'd like to process multiple images.

```
$ ls
image1.png image2.jpeg snarf.webp
$ parallel genloss {} ::: image1.png image2.jpeg snarf.webp 
```

Or maybe a list of images, for whatever reason.

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

## Explaining the "why?" of the options:

### `-n` and `--number` 

This was included because it's generally hard to predict how images will
degrade, *a priori*, as well as how fast you'll run out of disk/memory
(if you use a RAM disk as scratch space).

Let's make the former case clearer with an example from my own use:

> WebP (use `--format webp`) and JPEG degrade at different "rates", 
> for a particularly loose and intuitionistic definition of "rate", if you will.
> If JPEG at 80% quality gets mangled to unrecognizability in, say, 1500 iterations, 
> then WebP at the same quality might do that in, uh... 500.
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
