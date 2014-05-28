---
layout: post
title: "Timing (nested) function executions in Python"
---

Supposed you quickly want to find out how much time a bunch of functions in your code take to execute. You could use the [timeit][1] module Python comes with. With this you either have to run from command line or provide the statement to execute as a string. 

{% highlight py %}
#command line
$ python -m timeit '"-".join(str(n) for n in range(100))'
10000 loops, best of 3: 40.3 usec per loop

#python
>>> import timeit
>>> timeit.timeit('"-".join(str(n) for n in range(100))', number=10000)
0.818726062774658
{% endhighlight %}

Good for analysing short snippets very accurately but not very convenient to set up fast and it will only tell you total time. What if you algorithm calls several functions in succession and you want to know execution time for each of them?

For getting timing information on the fly I use this little function:
{% highlight py %}
import time

def time_it(f):
    time_it.active = 0

    def tt(*args, **kwargs):
        time_it.active += 1
        t0 = time.time()
        tabs = '\t'*(time_it.active - 1)+'>'
        name = f.__name__
        print('{tabs} Executing {name}'.format(tabs=tabs, name=name))
        res = f(*args, **kwargs)
        print('{tabs} Function {name} execution time: {time:.3f} seconds'.format(
            tabs=tabs, name=name, time=time.time() - t0))
        time_it.active -= 1
        return res
    return tt
{% endhighlight %}

This is meant to be used as a decorator on the function you want to time. It will print out the name of the function when it starts executing and the time it took once it's finished. That's all. The neat thing is that if you have nested function calls, you can put the decorator on all the functions and get timing and execution order info.


{% highlight py %}
import time
import scipy.ndimage as nd
import numpy as n

@time_it
def do_all(data, blurs):
    for blur in blurs:
        process(data, blur)
        
@time_it
def process(data, amount):
    blurred = nd.gaussian_filter(data, amount)
    out = get_stats(blurred)
    return out

@time_it
def get_stats(c):
    return c.mean(), c.std()
{% endhighlight %}

In this contrived example, we're applying a gaussian blur to some data. The function `do_all` gets an array and a list of blurring amounts to apply to it. It calls `process` for all of these amounts which in turn does the blurring and calls `get_stats` to calculate some stats on the blurred array. Not very realistic or efficient but it's just to demo the timing :) 


{% highlight py %}
>>>data = n.random.randn(5000, 5000)
>>>blurs = n.arange(0, 40, 10)

>>>do_all(data, blurs)
{% endhighlight %}
This gives the output:

{% highlight py %}
> Executing do_all
	> Executing process
		> Executing get_stats
		> Function get_stats execution time: 0.212 seconds
	> Function process execution time: 0.329 seconds
	> Executing process
		> Executing get_stats
		> Function get_stats execution time: 0.237 seconds
	> Function process execution time: 3.095 seconds
	> Executing process
		> Executing get_stats
		> Function get_stats execution time: 0.225 seconds
	> Function process execution time: 4.861 seconds
	> Executing process
		> Executing get_stats
		> Function get_stats execution time: 0.243 seconds
	> Function process execution time: 6.609 seconds
> Function do_all execution time: 14.894 seconds
{% endhighlight %}


So we get the timing info for all functions that have the `time_it` decorator and the order of execution. It's easy to modify this to also print the arguments each function gets called with and/or print the return value of each function call. Adding/removing timing for a function is just a matter of adding/removing the decorator.


[1]:https://docs.python.org/2/library/timeit.html

