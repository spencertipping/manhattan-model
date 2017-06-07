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

[![lateral slide](https://img.youtube.com/vi/ZG5mH6ywe7U/0.jpg)](https://www.youtube.com/watch?v=ZG5mH6ywe7U)

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

[![lateral slide](https://img.youtube.com/vi/oEhnxTyzNHA/0.jpg)](https://www.youtube.com/watch?v=oEhnxTyzNHA)

The buildings skew opposite directions in the two videos, which means depth
inference is very sensitive to camera rotation. That's probably the next thing
to address.

### Better depth mapping
Right now I'm using a bogus heuristic to infer depth: if we're moving towards
the vanishing point, then depth = 1 / tile shifting. But this is a problem in a
world where the camera can tilt or shift within the XY plane. We need a better
model.

The math is nontrivial, but we can go ahead and generate the full motion
inference frame by frame. I'm going to use single-frame differences and 60x60
tiles with 15x15 spacing.

```sh
$ ffmpeg -i v1.mp4 v1/%06d.png
$ ls v1/*??????.png | wc -l
62292
$ ni n62291p'r sprintf "%06d\t%06d", a, a+1' \
     S24p'use PDL; use PDL::FFT; use PDL::Image2D; use PDL::IO::Pic;
          BEGIN{$ts = 60;
                $to = 15;
                $maxs = 16;
                $fm = ones($ts/4, $ts/4)
                  ->append(zeroes $ts*3/4, $ts/4)
                  ->glue(1, zeroes $ts, $ts*3/4)}
          my $ia = rpic "v1/".a.".png";
          my $ib = rpic "v1/".b.".png";
          for my $tx (map $_*$to, 0..(3840-$ts)/$to) {
            for my $ty (map $_*$to, 0..(2160-$ts)/$to) {
              my $i1 = $ia->slice("X", [$tx, $tx+$ts-1], [$ty, $ty+$ts-1]);
              my $i2 = $ib->slice("X", [$tx, $tx+$ts-1], [$ty, $ty+$ts-1]);
              ($i1 = $i1->slice(0) + $i1->slice(1) + $i1->slice(2))->reshape($ts, $ts);
              ($i2 = $i2->slice(0) + $i2->slice(1) + $i2->slice(2))->reshape($ts, $ts);
              fftnd $i1, my $i1i = $i1*0;
              fftnd $i2, my $i2i = $i2*0;
              $_ *= $fm for $i1, $i2, $i1i, $i2i;
              $i2i *= -1;
              ifftnd my $hr = $i1*$i2 - $i1i*$i2i,
                     my $hi = $i1i*$i2 + $i2i*$i1;
              my $h = $hr**2 + $hi**2;
              my @maxs;
              until (@maxs >= $maxs * 3) {
                push @maxs, my ($m, $mi, $mj) = $h->max2d_ind;
                $h->set($mi, $mj, 0);
              }
              r a, b, $tx, $ty, @maxs;
            }
          }' \
     z\>phc-full-offsets
```

Rough guess for ETA (based on values from the ni monitor):

```sh
$ units -t '700MB / 48 * 62291 / (1541kB/s)' days
6.9398116
```

Alright, that's now done (about nine days total, but I was running other stuff
while it was going). Animating these vectors:

```sh
$ ls -lh phc-full-offsets               # this file's kinda big
-rw-r--r-- 1 spencertipping spencertipping 327G May 28 18:26 phc-full-offsets

$ export NI_ROW_SORT_BUFFER=16384M
$ ni phc-full-offsets fACDFGoz:phc-full-frames \
     p'r a, b, c, d>=30 ? d-60 : d, e>=30 ? e-60 : e' p'r a, b, c, d*4, e*4' \
     GAJ1600x900'plot [0:3840] [0:2160] "-" with vectors lc rgb "#c0000000"' \
     GF[-y -qscale 5 phc-v1-motion.avi]
```

The video is 34GB, so I'm going to spare my wifi and upload directly from the
server. I used xpra to access a browser running inside the docker container,
then uploaded the video from there. CenturyLink says they only count downloaded
data in their excessive use policy so I'm reasonably confident they won't send
me angry letters for uploading this much. (We'll see.)

[![motion vectors](https://img.youtube.com/vi/XuB35x2V9u0/0.jpg)](https://www.youtube.com/watch?v=XuB35x2V9u0)

### Aerial motion vectors
These are way less straightforward than driving because drones are both more
agile and less directional than cars are. [Here's the source
video](https://www.youtube.com/watch?v=pbNEeAAiIKk).

```sh
$ youtube-dl pbNEeAAiIKk -o v2.webm
$ mkdir v2
$ ffmpeg -i v2.webm v2/%06d.png
```

This video is a lot shorter than the first one, but it also includes scene
transitions and other complications. Using the same tiling strategy we did for
v1:

```sh
$ ls v2/*.png | wc -l
6477
$ ni n6476p'r sprintf "%06d\t%06d", a, a+1' \
     S24p'use PDL; use PDL::FFT; use PDL::Image2D; use PDL::IO::Pic;
          BEGIN{$ts = 60;
                $to = 15;
                $maxs = 16;
                $fm = ones($ts/4, $ts/4)
                  ->append(zeroes $ts*3/4, $ts/4)
                  ->glue(1, zeroes $ts, $ts*3/4)}
          my $ia = rpic "v2/".a.".png";
          my $ib = rpic "v2/".b.".png";
          for my $tx (map $_*$to, 0..(3840-$ts)/$to) {
            for my $ty (map $_*$to, 0..int((2026-$ts)/$to)) {
              my $i1 = $ia->slice("X", [$tx, $tx+$ts-1], [$ty, $ty+$ts-1]);
              my $i2 = $ib->slice("X", [$tx, $tx+$ts-1], [$ty, $ty+$ts-1]);
              ($i1 = $i1->slice(0) + $i1->slice(1) + $i1->slice(2))->reshape($ts, $ts);
              ($i2 = $i2->slice(0) + $i2->slice(1) + $i2->slice(2))->reshape($ts, $ts);
              fftnd $i1, my $i1i = $i1*0;
              fftnd $i2, my $i2i = $i2*0;
              $_ *= $fm for $i1, $i2, $i1i, $i2i;
              $i2i *= -1;
              ifftnd my $hr = $i1*$i2 - $i1i*$i2i,
                     my $hi = $i1i*$i2 + $i2i*$i1;
              my $h = $hr**2 + $hi**2;
              my @maxs;
              until (@maxs >= $maxs * 3) {
                push @maxs, my ($m, $mi, $mj) = $h->max2d_ind;
                $h->set($mi, $mj, 0);
              }
              r a, b, $tx, $ty, @maxs;
            }
          }' \
     z\>phc-v2-offsets
```

Here's an animation of those motion vectors (the simple version; we're not
doing subpixel registration yet):

```sh
$ ni phc-v2-offsets fACDFGoz:phc-v2-frames \
     p'r a, b, c, d>=30 ? d-60 : d, e>=30 ? e-60 : e' p'r a, b, c, d*4, e*4' \
     GAJ1600x900'plot [0:3840] [0:2160] "-" with vectors lc rgb "#c0000000"' \
     GF[-y -qscale 5 phc-v2-motion.avi]
```

[![motion vectors](https://img.youtube.com/vi/naJPkZfB0Xk/0.jpg)](https://www.youtube.com/watch?v=naJPkZfB0Xk)

### Subpixel registration
Right now I'm just taking the largest-magnitude alignment offset, but a more
accurate approach would be to start there and use a linear weighting to shift
the centroid. I'm also removing any points that would pull the centroid too far
away since those are most likely outliers. Here's what this process looks like:

```sh
$ export NI_ROW_SORT_BUFFER=16384M
$ ni phc-full-offsets S8rp'a < 1000' \
     S8p'my ($f, undef, $x, $y) = F_ 0..3;
         my @mags = F_ map $_*3 + 4, 0..15;
         my @xs   = map $_<30 ? $_ : $_-60, F_ map $_*3 + 5, 0..15;
         my @ys   = map $_<30 ? $_ : $_-60, F_ map $_*3 + 6, 0..15;
         my ($cx, $cy, $w) = ($xs[0], $ys[0], 0);
         for (0..$#mags) {
           next if 3 < l2norm $cx - $xs[$_], $cy - $ys[$_];
           $cx = $cx*$w + $xs[$_]*$mags[$_];
           $cy = $cy*$w + $ys[$_]*$mags[$_];
           $w += $mags[$_];
           $cx /= $w; $cy /= $w;
         }
         r $f, $x, $y, $cx, $cy' \
     oz:phc-v1-subpixel-1k-sorted \
     p'r a, b, c, d*4, e*4' \
     GAJ1600x900'plot [0:3840] [0:2160] "-" with vectors lc rgb "#80000000"' \
     GF[-y -qscale 5 phc-v1-subpixel.avi]
```

[![subpixel phase correlation](https://img.youtube.com/vi/buU-wan1kow/0.jpg)](https://www.youtube.com/watch?v=buU-wan1kow)

### Making this remotely scalable
One of the major problems with the current approach is that we're losing all of
the benefits of video encoding by unpacking each frame. This results in some
fairly major disk space usage; over 100x expansion:

```sh
$ du -shL v1.mp4 v1
4.5G    v1.mp4          # original video
596G    v1              # unpacked frames
```

Since we're just looking at adjacent frames, it's a bit silly to unpack
everything up front. It's better to use image compositing to adjoin + shift the
images so we end up with pairs of frames. Here's the general idea:

```sh
$ convert v1/000001.png v1/000002.png -append v1-frames-joined.png
```

![image](http://storage1.static.itmages.com/i/17/0529/h_1496083295_8153734_7b96bb1ed4.png)

We start with 3840x4320 of solid black and crop downwards to maintain a sliding
window of two images:

```sh
$ convert -size 3840x4320 xc:black v1/000001.png -append \
          -crop 3840x4320+0+2160\! v1-frames-joined-cropped.png
```

Here's that process inside a compositing loop that emits these windows as
individual 3840x4320 images, in this case reassembling into another AVI so I
can make sure it's working:

```sh
$ ni e[ffmpeg -i v1.mp4 -to 00:05 -f image2pipe -c:v png -] \
     IC[-size 3840x2160 xc:black -insert 0 -append] \
       [- -append -crop 3840x4320+0+2160\!] : \
     IJGF[-y v1-combined.avi]
```

[![streaming frame pairs](https://img.youtube.com/vi/M7h26AmYTTs/0.jpg)](https://www.youtube.com/watch?v=M7h26AmYTTs)

**Warning:** I recommend against doing things like this if you're using an SSD
instead of a magnetic disk. Although it won't use the 596GB all at once, we're
going to write about twice that to the disk total -- so if your SSD is 200GB,
then that's six writes to every byte on the disk.

The command above just re-encodes the frame pairs into a movie, but it's simple
enough to use the phase correlation code to generate offsets (and now we
migrate the multithreading to rows of the image, rather than separate images).
Let's do this with `v3.mp4`, another driving video.

There's one snag to be aware of here. Now that we're multithreading inside a
single process and sharing an output pipe, every write to stdout needs to be
small enough that it will happen atomically. I'm going to make this happen by
binary-packing the outputs and then unpacking them immediately in the next
step, which is fine because everything's numeric. This is a bit of an overdue
optimization in any case; gzipping numbers in text is kind of egregious.

I'm also adding in the subpixel registration from above; this should improve
the quality substantially.

```sh
$ ni e[ffmpeg -i v3.mp4 -f image2pipe -c:v png -] \
     IC[-size 3840x2160 xc:black -insert 0 -append] \
       [- -append -crop 3840x4320+0+2160\!] : \
     Ie[ perl -e \
       'use PDL; use PDL::FFT; use PDL::Image2D; use PDL::IO::Pic;
        use File::Temp qw/tmpnam/;
        BEGIN{$ts = 60;
              $to = 15;
              $maxs = 16;
              $fm = ones($ts/4, $ts/4)
                ->append(zeroes $ts*3/4, $ts/4)
                ->glue(1, zeroes $ts, $ts*3/4)}
        my $t  = tmpnam; system "cat > $t";
        my $i  = rpic $t, {FORMAT => "PNG"};
        my $ia = $i->slice("X", "X", [0, 2159]);
        my $ib = $i->slice("X", "X", [2160, 4319]);
        my @pids;
        for my $tx (map $_*$to, 0..(3840-$ts)/$to) {
          my $pid;
          if ($pid = fork) {
            push @pids, $pid;
          } else {
            for my $ty (map $_*$to, 0..(2160-$ts)/$to) {
              my $i1 = $ia->slice("X", [$tx, $tx+$ts-1], [$ty, $ty+$ts-1]);
              my $i2 = $ib->slice("X", [$tx, $tx+$ts-1], [$ty, $ty+$ts-1]);
              ($i1 = $i1->slice(0) + $i1->slice(1) + $i1->slice(2))->reshape($ts, $ts);
              ($i2 = $i2->slice(0) + $i2->slice(1) + $i2->slice(2))->reshape($ts, $ts);
              fftnd $i1, my $i1i = $i1*0;
              fftnd $i2, my $i2i = $i2*0;
              $_ *= $fm for $i1, $i2, $i1i, $i2i;
              $i2i *= -1;
              ifftnd my $hr = $i1*$i2 - $i1i*$i2i,
                     my $hi = $i1i*$i2 + $i2i*$i1;
              my $h = $hr**2 + $hi**2;
              my @maxs;
              until (@maxs >= $maxs * 3) {
                push @maxs, my ($m, $mi, $mj) = $h->max2d_ind;
                $h->set($mi, $mj, 0);
              }
              syswrite STDOUT, pack "Lss(dss)16", $ENV{KEY}, $tx, $ty, @maxs;
            }
            exit 0;
          }
        }
        waitpid $_, 0 for @pids;
        unlink($t) or die "not unlinking temp images: $!"' ] \
     :phc-v3-streaming-offsets-Lssdss16 bf'Lss(dss)16' \
     S24p'my ($f, $x, $y) = F_ 0..2;
          my @mags = F_ map $_*3 + 3, 0..15;
          my @xs   = map $_<30 ? $_ : $_-60, F_ map $_*3 + 4, 0..15;
          my @ys   = map $_<30 ? $_ : $_-60, F_ map $_*3 + 5, 0..15;
          my ($cx, $cy, $w) = ($xs[0], $ys[0], 0);
          for (0..$#mags) {
            next if !$mags[$_] || 3 < l2norm $cx - $xs[$_], $cy - $ys[$_];
            $cx = $cx*$w + $xs[$_]*$mags[$_];
            $cy = $cy*$w + $ys[$_]*$mags[$_];
            $w += $mags[$_];
            $cx /= $w; $cy /= $w;
          }
          r $f, $x, $y, $cx*4, $cy*4' o \
     GAJ1600x900'plot [0:3840] [0:2160] "-" with vectors lc rgb "black"' \
     GF[-y -qscale 5 phc-v3-streaming-motion.avi]
```

**TODO:** Upload this video once it's done

And another one:

```sh
$ ni e[ffmpeg -i v4.mp4 -f image2pipe -c:v png -] \
     IC[-size 3840x2160 xc:black -insert 0 -append] \
       [- -append -crop 3840x4320+0+2160\!] : \
     Ie[ perl -e \
       'use PDL; use PDL::FFT; use PDL::Image2D; use PDL::IO::Pic;
        use File::Temp qw/tmpnam/;
        BEGIN{$ts = 60;
              $to = 15;
              $maxs = 16;
              $fm = ones($ts/4, $ts/4)
                ->append(zeroes $ts*3/4, $ts/4)
                ->glue(1, zeroes $ts, $ts*3/4)}
        my $t  = tmpnam; system "cat > $t";
        my $i  = rpic $t, {FORMAT => "PNG"};
        my $ia = $i->slice("X", "X", [0, 2159]);
        my $ib = $i->slice("X", "X", [2160, 4319]);
        my @pids;
        for my $tx (map $_*$to, 0..(3840-$ts)/$to) {
          my $pid;
          if ($pid = fork) {
            push @pids, $pid;
          } else {
            for my $ty (map $_*$to, 0..(2160-$ts)/$to) {
              my $i1 = $ia->slice("X", [$tx, $tx+$ts-1], [$ty, $ty+$ts-1]);
              my $i2 = $ib->slice("X", [$tx, $tx+$ts-1], [$ty, $ty+$ts-1]);
              ($i1 = $i1->slice(0) + $i1->slice(1) + $i1->slice(2))->reshape($ts, $ts);
              ($i2 = $i2->slice(0) + $i2->slice(1) + $i2->slice(2))->reshape($ts, $ts);
              fftnd $i1, my $i1i = $i1*0;
              fftnd $i2, my $i2i = $i2*0;
              $_ *= $fm for $i1, $i2, $i1i, $i2i;
              $i2i *= -1;
              ifftnd my $hr = $i1*$i2 - $i1i*$i2i,
                     my $hi = $i1i*$i2 + $i2i*$i1;
              my $h = $hr**2 + $hi**2;
              my @maxs;
              until (@maxs >= $maxs * 3) {
                push @maxs, my ($m, $mi, $mj) = $h->max2d_ind;
                $h->set($mi, $mj, 0);
              }
              syswrite STDOUT, pack "Lss(dss)16", $ENV{KEY}, $tx, $ty, @maxs;
            }
            exit 0;
          }
        }
        waitpid $_, 0 for @pids;
        unlink($t) or die "not unlinking temp images: $!"' ] \
     :phc-v4-streaming-offsets-Lssdss16 bf'Lss(dss)16' \
     S24p'my ($f, $x, $y) = F_ 0..2;
          my @mags = F_ map $_*3 + 3, 0..15;
          my @xs   = map $_<30 ? $_ : $_-60, F_ map $_*3 + 4, 0..15;
          my @ys   = map $_<30 ? $_ : $_-60, F_ map $_*3 + 5, 0..15;
          my ($cx, $cy, $w) = ($xs[0], $ys[0], 0);
          for (0..$#mags) {
            next if !$mags[$_] || 3 < l2norm $cx - $xs[$_], $cy - $ys[$_];
            $cx = $cx*$w + $xs[$_]*$mags[$_];
            $cy = $cy*$w + $ys[$_]*$mags[$_];
            $w += $mags[$_];
            $cx /= $w; $cy /= $w;
          }
          r $f, $x, $y, $cx*4, $cy*4' o \
     GAJ1600x900'plot [0:3840] [0:2160] "-" with vectors lc rgb "black"' \
     GF[-y -qscale 5 phc-v4-streaming-motion.avi]
```

### Nighttime vs daytime
I'm interested to see how well all of this works for nighttime videos, so let's
take a look at the Afroduck speed lap (edited version, unfortunately; the
original was taken down).

Checking the frame size:

```sh
$ mkdir afroduck-fast
$ ffmpeg -i afroduck-fast.mkv -to 00:20 -f image2 afroduck-fast/%06d.png
$ file afroduck-fast/000001.png
afroduck-fast/000001.png: PNG image data, 1920 x 1080, 8-bit/color RGB, non-interlaced
```

Now time for a modified phase correlation setup. Besides the different image
size, some frames are encoded in black/white -- which means when PDL loads them
we don't get separate RGB channels. The simplest workaround is to just skip
those frames, ideal because they're overlay text.

```sh
$ ni e[ffmpeg -ss 00:19 -i afroduck-fast.mkv -deinterlace \
              -f image2pipe -c:v png -] \
     IC[-size 1920x1080 xc:black -insert 0 -append] \
       [- -append -crop 1920x2160+0+1080\!] : \
     Ie[ perl -e \
       'use PDL; use PDL::FFT; use PDL::Image2D; use PDL::IO::Pic;
        use File::Temp qw/tmpnam/;
        BEGIN{$ts = 120;
              $to = 5;
              $maxs = 16;
              $fm = ones($ts/4, $ts/4)
                ->append(zeroes $ts*3/4, $ts/4)
                ->glue(1, zeroes $ts, $ts*3/4)}
        my $t  = tmpnam; system "cat > $t";
        my $i  = rpic $t, {FORMAT => "PNG"};
        exit 0 unless $i->shape->at(0) == 3;
        my $ia = $i->slice("X", "X", [0, 1079]);
        my $ib = $i->slice("X", "X", [1080, 2159]);
        my @pids;
        for my $tx (map $_*$to, 0..(1920-$ts)/$to) {
          my $pid;
          if ($pid = fork) {
            push @pids, $pid;
          } else {
            for my $ty (map $_*$to, 0..(1080-$ts)/$to) {
              my $i1 = $ia->slice("X", [$tx, $tx+$ts-1], [$ty, $ty+$ts-1]);
              my $i2 = $ib->slice("X", [$tx, $tx+$ts-1], [$ty, $ty+$ts-1]);
              ($i1 = $i1->slice(0) + $i1->slice(1) + $i1->slice(2))->reshape($ts, $ts);
              ($i2 = $i2->slice(0) + $i2->slice(1) + $i2->slice(2))->reshape($ts, $ts);
              fftnd $i1, my $i1i = $i1*0;
              fftnd $i2, my $i2i = $i2*0;
              $_ *= $fm for $i1, $i2, $i1i, $i2i;
              $i2i *= -1;
              ifftnd my $hr = $i1*$i2 - $i1i*$i2i,
                     my $hi = $i1i*$i2 + $i2i*$i1;
              my $h = $hr**2 + $hi**2;
              my @maxs;
              until (@maxs >= $maxs * 3) {
                push @maxs, my ($m, $mi, $mj) = $h->max2d_ind;
                $h->set($mi, $mj, 0);
              }
              syswrite STDOUT, pack "Lss(dss)16", $ENV{KEY}, $tx, $ty, @maxs;
            }
            exit 0;
          }
        }
        waitpid $_, 0 for @pids;
        unlink($t) or die "not unlinking temp images: $!"' ] \
     :afroduck-fast-offsets-Lssdss16 bp'r rp"Lss(dss)16"' \
     S24p'my ($f, $x, $y) = F_ 0..2;
          my @mags = F_ map $_*3 + 3, 0..15;
          my @xs   = map $_<60 ? $_ : $_-120, F_ map $_*3 + 4, 0..15;
          my @ys   = map $_<60 ? $_ : $_-120, F_ map $_*3 + 5, 0..15;
          my ($cx, $cy, $w) = ($xs[0], $ys[0], 0);
          for (0..$#mags) {
            next if !$mags[$_] || 3 < l2norm $cx - $xs[$_], $cy - $ys[$_];
            $cx = $cx*$w + $xs[$_]*$mags[$_];
            $cy = $cy*$w + $ys[$_]*$mags[$_];
            $w += $mags[$_];
            $cx /= $w; $cy /= $w;
          }
          r $f, $x, $y, $cx*4, $cy*4' o \
     GAJ1600x900'plot [0:1920] [0:1080] "-" with vectors lc rgb "black"' \
     GF[-y -qscale 5 afroduck-fast-motion.avi]
```

**TODO:** Upload this video once it's done

## 3D motion
Most of the motion in these videos is into or out of the frame, which means
we'll have a vanishing point. It's not always obvious where that vanishing
point is, however; if the camera rotates at all then each motion vector will be
offset by the same unknown amount.

Here's what the vectors look like if we move towards a random cloud:

```sh
$ ni nE4p'r rand() - 0.5, rand() - 0.5, rand() + 2' \
        p'r a/c, b/c, a/(c-0.1) - a/c, b/(c-0.1) - b/c' \
  | nfu -p 'with vectors'
```

![image](http://pix.toile-libre.org/upload/original/1496710343.png)

And here's what it looks like if we add a uniform offset for a camera rotation:

```sh
$ ni nE4p'r rand() - 0.5, rand() - 0.5, rand() + 2' \
        p'r a/c, b/c, a/(c-0.1) - a/c + 0.005, b/(c-0.1) - b/c + 0.002' \
  | nfu -p 'with vectors'
```

![image](http://pix.toile-libre.org/upload/original/1496710487.png)

It's subtle, but the offset is recoverable; you can see the vectors misaligning
if you zoom in:

![image](http://pix.toile-libre.org/upload/original/1496710672.png)

Now it's time for the math (or at least for a non-platitudinous answer).

### Recovering camera rotation
The misaligned vectors happen because they didn't all start out at the same
distance from the camera and the magnitude of a tile's motionvector is
1/distance. The direction of each motionvector, absent camera rotation, is
always towards or away from the vanishing point.

Initially we have an unknown vector `R` for rotation, an unknown `v` for the
vanishing point, and each input tile (specified by `o` for offset and `Δ` for
motion) carries an unknown `ρ` specifying the vector multiple required to reach
the vanishing point. So we have this:

```
o1 + ρ1(Δ1 + R) = v
o2 + ρ2(Δ2 + R) = v
...
```

This isn't a linear system, but it's easy to solve for `R` numerically. Once we
know `R` we have:

```
o1 + ρ1Δ1 = v
o2 + p2Δ2 = v
...
```

Rewriting it all as scalar terms:

```
Δx1ρ1 - vx = ox1
Δy1ρ1 - vy = oy1
...
```

This forms a simple matrix system that converges to twice as many equations as
variables. Here's the layout and coefficients:

```
   ρ1   ρ2  ... ρN vx vy
 | Δx1             -1    |   | ox1 |
 | Δy1                -1 |   | oy1 |
 |      Δx2        -1    | = | ox2 |
 |      Δy2           -1 |   | oy2 |
            ...                ...
```

But to get there we first need to know `R`. Here's what the error contour looks
like if we just assume different values for it (I'm going to use a handful of
input points to minimize runtime):

```sh
$ ni ::mvs[nE5p'r rand() - 0.5, rand() - 0.5, rand() + 2' \
              p'r a/c, b/c, a/(c-0.1) - a/c, b/(c-0.1) - b/c'] \
     1p'use PDL; r zeroes(150)->xlinvals(-1, 1)->list' p'cart [F_], [F_]' \
     S24p'^{use PDL; use PDL::MatrixOps;
            ($ox, $oy, $dx, $dy) = (pdl(a_ mvs), pdl(b_ mvs),
                                    pdl(c_ mvs), pdl(d_ mvs));
            $n = 10}
          my ($rx, $ry) = F_;
          my $m = zeroes $n + 2, $n * 2;
          $m->set($_,     $_ << 1,     $dx->at($_) + $rx)
            ->set($_,     $_ << 1 | 1, $dy->at($_) + $ry)
            ->set($n,     $_ << 1,     -1)
            ->set($n + 1, $_ << 1 | 1, -1) for 0..$n-1;
          my $mt  = $m->transpose;
          my $rhs = $ox->slice([0, $n-1])->transpose
           ->append($oy->slice([0, $n-1])->transpose)->flat->transpose;
          my $s   = ($mt x $m)->inv x $mt x $rhs;
          r $rx, $ry, ($m x $s - $rhs)->flat->abs->average' \
     z\>r-numsolve-error
```

![image](http://storage6.static.itmages.com/i/17/0607/h_1496804504_6589574_57a7ceea9c.png)

That's awkward.
