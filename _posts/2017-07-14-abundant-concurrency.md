---
layout: post
title: Abundant concurrency in Go
---

Today, I merged
[a Pull Request](https://github.com/hunterloftis/pbr/pull/9/files)
that's so obviously the right model for my
[Go rendering engine](https://github.com/hunterloftis/pbr#pbr-a-physically-based-renderer-in-go)
that I had to sit down and figure out why I did it any other way in the first place.

It turns out I have a [JavaScripter's](https://github.com/hunterloftis/throng)
mindset on concurrency.

In JavaScript, you have two levels of concurrency to choose from.
On a single CPU you have the single-threaded, non-blocking type,
where an event loop lets you simulate I/O concurrency.
This doesn't help with hashing passwords or processing long lists,
but you can read a file without ignoring HTTP requests.
Across CPUs you have workers (*web workers* in the browser, *cluster workers* in node).
Each worker forks its own process with all that that entails for
startup time, memory use, and inter-process communication.

This generates two habits:
First, you *always* write async code because you're sharing a single thread with the rest of the process.
Iterating over a large array can cause performance issues in a JS app.
Second, you consider concurrency in terms of single digits to map processes to CPUs.
If you spend too much time on a task, you block other functions from the event loop;
if you spin up too many processes, you create contention and waste resources.

So, when I added concurrency to
[pbr](https://github.com/hunterloftis/pbr#pbr-a-physically-based-renderer-in-go)'s API,
I forced the user to spin up a handful of goroutines that they'd monitor over channels.
It was made "easier" via this complex
[Monitor](https://github.com/hunterloftis/pbr/blob/2c876535011379b54d93c58ba72500c8e6c69771/pbr/monitor.go)
type that created goroutines for you:

```go
// AddSampler creates a new worker with that sampler.
// (user's responsibility)
func (m *Monitor) AddSampler(s *Sampler) {
	m.active++
	go func() {
		for {
			frame := s.SampleFrame()
			m.samples.Lock()
			m.samples.count += frame
			total := m.samples.count
			m.samples.Unlock()
			m.Progress <- total
			select {
			case <-m.cancel:
				m.active--
				m.Results <- s.Pixels()
				return
			default:
			}
		}
	}()
}
```

Pushing the concurrent requirements up to the user resulted in
[this monstrosity](https://github.com/hunterloftis/pbr/blob/2c876535011379b54d93c58ba72500c8e6c69771/cmd/render/render.go#L74-L94)
which will be familiar to anyone who's used
[node's cluster API](https://nodejs.org/api/cluster.html#cluster_cluster).

As I sketched out a 100-line, 2-channel "hello, world" example, I realized my mistake.
The pbr renderer could start as many goroutines as it needs to quickly render an image
without ever exposing them to the user.
In Go, I can build internal concurrency while exposing a simple, sequential API to the user.
In JavaScript, it would be unthinkable to spawn several async routines that each `for` loop through two billion pixels at once,
but that's [exactly what pbr does now](https://github.com/hunterloftis/pbr/blob/master/pbr/sampler.go#L68):

```go
// Sample samples every pixel in the Camera's frame at least once.
// (package's responsibility)
// (yeah I know this is a little messy, I'll clean it up later, the point is the user doesn't deal with the mess)
func (s *Sampler) Sample() {
	length := index(len(s.samples))
	workers := index(runtime.NumCPU())
	ch := make(chan sampleStat, workers)

	for i := index(0); i < workers; i++ {
		go func(i index, adapt, max int, mean float64) {
			var stat sampleStat
			rnd := rand.New(rand.NewSource(time.Now().UnixNano()))
			for p := i * Stride; p < length; p += Stride * workers {
				samples := adaptive(s.samples[p+Noise], adapt, max, mean)
				stat.noise += s.samplePixel(p, rnd, samples)
				stat.count += samples
			}
			ch <- stat
		}(i, s.Adapt, s.Adapt*3, s.meanNoise+Bias)
	}

	var sample sampleStat
	for i := index(0); i < workers; i++ {
		stat := <-ch
		sample.count += stat.count
		sample.noise += stat.noise
	}
	s.count += sample.count
	s.meanNoise = sample.noise / float64(sample.count)
}
```

The new method approaches concurrency like a Go developer,
with an *abundance mindset.*

Now you can render pbr's
[Hello, world scene](https://github.com/hunterloftis/pbr#hello-world)
with 15 lines and zero channels.
Underneath, `sampler.Sample()` creates goroutines for every block of *N* pixels to saturate your CPU.
But why would you care?
You just want a pretty picture.

```go
func main() {
	scene := pbr.EmptyScene()
	camera := pbr.NewCamera(960, 540)
	sampler := pbr.NewSampler(camera, scene)
	renderer := pbr.NewRenderer(sampler)

	scene.SetSky(pbr.Vector3{256, 256, 256}, pbr.Vector3{})
	scene.Add(pbr.UnitSphere(pbr.Plastic(1, 0, 0, 1)))

	for sampler.PerPixel() < 200 {
		sampler.Sample()
		fmt.Printf("\r%.1f samples / pixel", sampler.PerPixel())
	}
	pbr.WritePNG("hello.png", renderer.Rgb())
}
```
