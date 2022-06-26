# genloss

Induce generation loss on images via iterated rotation and saving,
or "spin to win", image edition.

# Usage

## Self-plagiarized:

```
./genloss [options] filename
```

Oh, wait, you wanted more? 

In lieu of the more traditional "this is what it does" explanations 
that pervade docs, I've instead decided to explain *why* these options exist,
a rough cookbook of ways I've used this script, and why I chose to implement
this "spin to win" tactic, instead of letting it be still.

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
arbitary media (visuals, sound) through a lossy (as opposed to a lossless) 
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
