# Let's build a 3D model of Manhattan
There's tons of driving + aerial youtube footage available, so we should be
able to do this. Let's start with just a single [driving
video](https://www.youtube.com/watch?v=D3oFGOJr-ak) and see if we can decode
the relative motion into 3D displacement, thus giving us surrounding polygon
structure.

```sh
$ youtube-dl D3oFGOJr-ak -o v1.mp4
```

Now let's extract ten seconds of still frames. PPM is slow to encode but easy
to manipulate.

```sh
$ mkdir v1
$ ffmpeg -i v1.mp4 -t 00:00:10 v1/%04d.ppm
```

## Motion inference
Let's start with a couple of frames with few immediate obstructions. Frame 46
occurs just after the woman leaves the left side of the screen; frame 76
happens one second later, after the car has moved about six feet forwards. If
we tile out the images and do phase correlation on the tiles, we should be able
to get a vector field.

First let's figure out how long the PPM prefix is:

```sh
$ ni v1/0046.ppm bp'r rp "c8"' r10
80      54      10      51      56      52      48      32
50      49      54      48      10      50      53      53
10      25      27      27      25      27      27      29
29      29      30      30      30      30      30      30
30      30      30      33      30      34      34      31
35      36      30      35      37      31      36      36
30      37      35      29      36      36      30      37
40      34      41      60      53      55      86      79
81      88      81      83      91      84      86      94
87      86      100     93      92      103     96      93
```

The last `10` is at byte 17, so that's where the header ends. Since all of the
images are the same size and have no metadata, that will be true for every
frame. Finding possible tile dimensions:

```sh
$ factor 2160 3840
2160: 2 2 2 2         3 3 3 5           # I added the spacing here
3840: 2 2 2 2 2 2 2 2     3 5
```

I'm going to use a square aspect ratio for individual tiles, which means the
largest we can do is 240x240. Let's start with that.

```sh
$ ni i0046 i0076 \
     p'use PDL; use PDL::IO::Pic;
       my $i = rpic "v1/".a.".ppm";
       for (0..3840*2160 / 240**2 - 1) {
         my ($tx, $ty) = ($_ * 240 % 3840, 240 * ($_ >> 4));
         wpic $i->slice("X", [$tx, $tx+239], [$ty, $ty+239]), "v1/".a."-$_.ppm";
       }'
```

Now let's phase-correlate each pair of images. I'm collapsing the color space
to grayscale for the moment, but later we can do better.

```sh
$ ni n0=16*9 \
     p'use PDL; use PDL::IO::Pic; use PDL::FFT; use PDL::Complex; use PDL::Image2D;
       my $i1 = double rpic "v1/0046-$_.ppm";
       my $i2 = double rpic "v1/0076-$_.ppm";
       ($i1 = $i1->slice(0) + $i1->slice(1) + $i1->slice(2))->reshape(240, 240);
       ($i2 = $i2->slice(0) + $i2->slice(1) + $i2->slice(2))->reshape(240, 240);
       fftnd $i1, my $i1i = $i1*0;
       fftnd $i2, my $i2i = $i2*0;
       $i2i *= -1;
       ifftnd my $hr = $i1*$i2 - $i1i*$i2i,
              my $hi = $i1i*$i2 + $i2i*$i1;
       (my $h = (($hr**2)+($hi**2)))->wpic("phc-$_.png");
       r a >> 4, a & 15, $h->max2d_ind' \
     \>phc-offsets
```

Here's the vector field:

```sh
$ ni phc-offsets p'r a*240, b*240, d>120 ? 240 - d : d, e>120 ? 240 - e : e' \
  | nfu -p %v
```

![image](http://storage2.static.itmages.com/i/17/0520/h_1495246121_9262163_7f18e93e83.png)

Ok, the bad news is that this is completely wrong. The good news, though, is
that the vector field actually does resemble what we expect: high values around
the edges and lower values in the middle. In other words, we have a vanishing
point.
