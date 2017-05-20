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
to manipulate (**NB:** this was a mistake in hindsight; PNG is more appropriate
given that we're using PDL to load the images.)

```sh
$ mkdir v1
$ ffmpeg -i v1.mp4 -t 00:00:10 v1/%04d.ppm
```

## Motion inference
Let's start with a couple of frames with few immediate obstructions. Frame 46
occurs just after the woman leaves the left side of the screen; frame 76
happens one second later, after the car has moved about six feet forwards. If
we tile out the images and do phase correlation on the tiles, we should be able
to get a vector field. Here's 46:

![image](http://storage5.static.itmages.com/i/17/0520/h_1495271539_8655813_b241d25609.png)

Possible tile dimensions:

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
$ ni phc-offsets p'r a*240, b*240, e>120 ? e-240 : e, d>120 ? d-240 : d' \
  | nfu -p %v
```

![image](http://storage5.static.itmages.com/i/17/0520/h_1495246704_8832143_d84f6bc10f.png)

Ok, so not perfect -- but not awful either. It's definitely possible to see the
structure here. In fact, we can probably do much better if we choose two images
closer together; let's repeat this for 0046 and 0051.

```sh
$ ni i0046 i0051 \
     p'use PDL; use PDL::IO::Pic;
       my $i = rpic "v1/".a.".ppm";
       for (0..3840*2160 / 240**2 - 1) {
         my ($tx, $ty) = ($_ * 240 % 3840, 240 * ($_ >> 4));
         wpic $i->slice("X", [$tx, $tx+239], [$ty, $ty+239]), "v1/".a."-$_.ppm";
       }'

$ ni n0=16*9 \
     p'use PDL; use PDL::IO::Pic; use PDL::FFT; use PDL::Complex; use PDL::Image2D;
       my $i1 = double rpic "v1/0046-$_.ppm";
       my $i2 = double rpic "v1/0051-$_.ppm";
       ($i1 = $i1->slice(0) + $i1->slice(1) + $i1->slice(2))->reshape(240, 240);
       ($i2 = $i2->slice(0) + $i2->slice(1) + $i2->slice(2))->reshape(240, 240);
       fftnd $i1, my $i1i = $i1*0;
       fftnd $i2, my $i2i = $i2*0;
       $i2i *= -1;
       ifftnd my $hr = $i1*$i2 - $i1i*$i2i,
              my $hi = $i1i*$i2 + $i2i*$i1;
       (my $h = (($hr**2)+($hi**2)))->wpic("phc2-$_.png");
       r a >> 4, a & 15, $h->max2d_ind' \
     :phc2-offsets p'r a*240, b*240, 10*(e>120 ? e-240 : e), 10*(d>120 ? d-240 : d)' \
  | nfu -p %v
```

(Notice I've multiplied the vectors to make them stand out more.)

![image](http://storage8.static.itmages.com/i/17/0520/h_1495246954_3239089_2dc73529c2.png)

Now _that_ looks good. This means the basic idea works; more data points and
more assumptions and we should be able to get a very fine-grained
reconstruction. Let's try using smaller tiles and adjacent frames.

```sh
$ ni i0046 i0047 \
     p'use PDL; use PDL::IO::Pic;
       my $i = rpic "v1/".a.".ppm";
       for (0..3840*2160 / 60**2 - 1) {
         my ($tx, $ty) = ($_ * 60 % 3840, 60 * ($_ >> 6));
         wpic $i->slice("X", [$tx, $tx+59], [$ty, $ty+59]), "v1/".a."-$_.ppm";
       }'

$ ni n0=3840*2160/20**2 \
     p'use PDL; use PDL::IO::Pic; use PDL::FFT; use PDL::Complex; use PDL::Image2D;
       my $i1 = double rpic "v1/0046-$_.ppm";
       my $i2 = double rpic "v1/0047-$_.ppm";
       ($i1 = $i1->slice(0) + $i1->slice(1) + $i1->slice(2))->reshape(60, 60);
       ($i2 = $i2->slice(0) + $i2->slice(1) + $i2->slice(2))->reshape(60, 60);
       fftnd $i1, my $i1i = $i1*0;
       fftnd $i2, my $i2i = $i2*0;
       $i2i *= -1;
       ifftnd my $hr = $i1*$i2 - $i1i*$i2i,
              my $hi = $i1i*$i2 + $i2i*$i1;
       (my $h = (($hr**2)+($hi**2)))->wpic("phc3-$_.png");
       r a >> 6, a & 63, $h->max2d_ind' \
     :phc3-offsets p'r a*60, b*60, 10*(e>30 ? e-60 : e), 10*(d>30 ? d-60 : d)' \
  | nfu -p %v
```

![image](http://storage2.static.itmages.com/i/17/0520/h_1495271457_9644589_0c89d45dc8.png)

### Shape inference
Right now we have a circular problem: in order to recover the camera motion we
need a surface map, and in order to build a surface map we need to know the
camera motion. Fundamentally this is an unavoidable ambiguity; the camera isn't
going to pick up on planetary movement, for example -- so its motion vector is
relative to the ground, not absolute in any sense.

All of that is fine for us, of course. We can safely assume the most common
behavior for an object is not to move, which (almost) implies that we can then
take the largest set of motion vectors that would be consistent with a linear
camera movement and infer camera motion accordingly.

The (almost) comes in because while most objects don't move, most _visible_
objects might. Imagine a driving video that stops for a train, for example, and
the train covers more than half the screen. Then we'd infer that the camera is
moving rapidly left or right and that non-train objects are moving with it. For
now I'm going to ignore this case.

#### More motion vectors
If we want to do shape inference, we're going to need a lot more data. I'm also
going to start considering the magnitude of the phase correlation output as a
way to assess signal strength. Here's the idea:

1. Let's use overlapping tiles to get more motion vectors without losing
   context.
2. Let's also use the normalized phase correlation magnitude to measure changes
   in depth relative to the camera (or any other instability that makes it
   difficult to infer structure).

For (1), let's do 60x60 tiles that overlap by 15px in each dimension. I'm going
to start using proper tile IDs here because the edge cases make it more
complicated to translate the numbers back into coordinates. I'm also going to
fuse the two commands to avoid writing all of the tiles now that we have a
nontrivial number of them.

```sh
$ ni i[0046 0051] \
     p'use PDL; use PDL::FFT; use PDL::Image2D; use PDL::IO::Pic;
       my $ia = rpic "v1/".a.".ppm";
       my $ib = rpic "v1/".b.".ppm";
       for my $tx (map $_*15, 0..(3840-60)/15) {
         for my $ty (map $_*15, 0..(2160-60)/15) {
           my $i1 = $ia->slice("X", [$tx, $tx+59], [$ty, $ty+59]);
           my $i2 = $ib->slice("X", [$tx, $tx+59], [$ty, $ty+59]);
           ($i1 = $i1->slice(0) + $i1->slice(1) + $i1->slice(2))->reshape(60, 60);
           ($i2 = $i2->slice(0) + $i2->slice(1) + $i2->slice(2))->reshape(60, 60);
           fftnd $i1, my $i1i = $i1*0;
           fftnd $i2, my $i2i = $i2*0;
           $i2i *= -1;
           ifftnd my $hr = $i1*$i2 - $i1i*$i2i,
                  my $hi = $i1i*$i2 + $i2i*$i1;
           r $tx, $ty, (($hr**2)+($hi**2))->max2d_ind;
         }
       }' \
     :phc4-offsets p'r a, b, 10*(e>30 ? e-60 : e), 10*(d>30 ? d-60 : d)' \
  | nfu -p %v
```

![image](http://storage5.static.itmages.com/i/17/0520/h_1495274584_5413198_add89fd793.png)

If we immediately map vector magnitude to reciprocal of depth and assume a
horizontal view angle of 90 degrees, here's the model we get.

```sh
$ ni phc4-offsets p'my $dx = d>30 ? d-60 : d;
                    my $dy = e>30 ? 3-60 : e;
                    my $depth = 1 / (1e-12 + sqrt($dx**2 + $dy**2) / 300);
                    my $dx = $depth * (a/3840 - 0.5);
                    my $dy = $depth * (b/3840 - 0.5);
                    my $dz = sqrt $depth**2 - ($dx**2 + $dy**2);
                    r $dx, $dy, $dz, c'
```

![image](http://storage2.static.itmages.com/i/17/0520/h_1495275372_7844836_50d7f963fd.png)

This really isn't bad. There are a lot of artifacts, but there's also a lot of
structure visible in this data; we're getting parallel lines for the buildings.

![image](http://storage9.static.itmages.com/i/17/0520/h_1495275930_1995187_611b5d263a.png)

Let's see if the model converges as we do this for more frames. I'm going to
use 240x240 tiles so we get a more accurate alignment; I've also lowpassed the
FFT so we don't catch so much noise.

```sh
$ ni n290rp'a > 45' \
     e[ xargs -P24 -I{} ni i{} \
        p'r sprintf "%04d\t%04d", a, a+10' \
        p'use PDL; use PDL::FFT; use PDL::Image2D; use PDL::IO::Pic;
          BEGIN{$fm = ones(60, 60)->append(zeroes 180, 60)
                                  ->glue(1, zeroes 240, 180)}
          my $ia = rpic "v1/".a.".ppm";
          my $ib = rpic "v1/".b.".ppm";
          for my $tx (map $_*30, 0..(3840-240)/30) {
            for my $ty (map $_*30, 0..(2160-240)/30) {
              my $i1 = $ia->slice("X", [$tx, $tx+239], [$ty, $ty+239]);
              my $i2 = $ib->slice("X", [$tx, $tx+239], [$ty, $ty+239]);
              ($i1 = $i1(0) + $i1(1) + $i1(2))->reshape(240, 240);
              ($i2 = $i2(0) + $i2(1) + $i2(2))->reshape(240, 240);
              fftnd $i1, my $i1i = $i1*0;
              fftnd $i2, my $i2i = $i2*0;
              $_ *= $fm for $i1, $i2, $i1i, $i2i;
              $i2i *= -1;
              ifftnd my $hr = $i1*$i2 - $i1i*$i2i,
                     my $hi = $i1i*$i2 + $i2i*$i1;
              r a, b, $tx, $ty, (($hr**2)+($hi**2))->max2d_ind;
            }
          }' ] \
     z\>phc5-offsets
```

![image](http://storage2.static.itmages.com/i/17/0520/h_1495284242_8583864_809885e3bd.png)

Now let's apply these offsets to frame 46 to see if it looks reasonable. We can
do this by animating a lateral slide.

```sh
$ ni n400e[ xargs -P24 -I{} ni \
            ::depths[phc5-offsets rp'a eq "0046"' fC. \
              p'my $dx = d>120 ? d-240 : d;
                my $dy = e>120 ? e-240 : e;
                r a.",".b, 1 / (1e-8 + sqrt($dx**2 + $dy**2) / 120)'] \
            i{} \
            p'use PDL; use PDL::IO::Pic; use PDL::Image2D;
              my %d = ab_ depths;
              my $dx = a*20;
              my $ts = 60;
              my $i = rpic "v1/0046.ppm";
              my $o = double $i * 0;
              for my $xy (sort {$d{$b} <=> $d{$a}} keys %d) {
                my ($x, $y)   = split /,/, $xy;
                my ($rx, $ry) = ($x + 120, $y + 120);
                my ($px, $py) = ($rx - $dx/$d{"$x,$y"}, $ry);
                next unless within($ts, 3840-$ts-1, $rx, $px)
                         && within($ts, 2160-$ts-1, $ry, $py);
                $o->slice("X", [$px-$ts, $px+$ts], [$py-$ts, $py+$ts])
                  += $i->slice("X", [$rx-$ts, $rx+$ts], [$ry-$ts, $ry+$ts]);
              }
              $o->clip(0, 256 * ($ts/15)**2)->wpic(sprintf "0046-%04d.jpg", a)' ]

$ ffmpeg -i 0046-%04d.jpg 0046.webm
```

[Here's the video](http://spencertipping.com/mm-0046.webm).

Not bad at all, especially considering that we're doing nothing to correct for
rotation. Frames 66 and 76 should yield a much more accurate depth map for that
reason:

```sh
$ ni n400e[ xargs -P24 -I{} ni \
            ::depths[phc5-offsets rp'a eq "0066"' fC. \
              p'my $dx = d>120 ? d-240 : d;
                my $dy = e>120 ? e-240 : e;
                r a.",".b, 1 / (1e-8 + sqrt($dx**2 + $dy**2) / 120)'] \
            i{} \
            p'use PDL; use PDL::IO::Pic; use PDL::Image2D;
              my %d = ab_ depths;
              my $dx = a*20;
              my $ts = 45;
              my $i = rpic "v1/0066.ppm";
              my $o = double $i * 0;
              for my $xy (sort {$d{$b} <=> $d{$a}} keys %d) {
                my ($x, $y)   = split /,/, $xy;
                my ($rx, $ry) = ($x + 120, $y + 120);
                my ($px, $py) = ($rx - $dx/$d{"$x,$y"}, $ry);
                next unless within($ts, 3840-$ts-1, $rx, $px)
                         && within($ts, 2160-$ts-1, $ry, $py);
                $o->slice("X", [$px-$ts, $px+$ts], [$py-$ts, $py+$ts])
                  += $i->slice("X", [$rx-$ts, $rx+$ts], [$ry-$ts, $ry+$ts]);
              }
              $o->clip(0, 256 * ($ts/15)**2)->wpic(sprintf "0066-%04d.jpg", a)' ]

$ ffmpeg -i 0066-%04d.jpg 0066.webm
```
