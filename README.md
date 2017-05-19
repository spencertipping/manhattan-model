# Let's build a 3D model of Manhattan
There's tons of driving + aerial youtube footage available, so we should be
able to do this. Let's start with just a single [driving
video](https://www.youtube.com/watch?v=D3oFGOJr-ak) and see if we can decode
the relative motion into 3D displacement, thus giving us surrounding polygon
structure.

```sh
$ youtube-dl D3oFGOJr-ak -o v1.mp4
```

Now let's extract ten seconds of still frames.

```sh
$ ffmpeg -i v1.mp4 -t 00:00:10 -r 1/1 v1-%d.png
```
