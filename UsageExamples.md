# Examples #

## Triangular numbers ##

Here's an example calculating and printing the first 100 [triangular numbers](http://en.wikipedia.org/wiki/Triangular_number).

```
### triangular.py
# server
source = dict(zip(range(100), range(100)))

def final(key, value):
    print key, value

# client
def mapfn(key, value):
    for i in range(value + 1):
        yield key, i

def reducefn(key, value):
    return sum(value)
```

Assuming the file is called `triangular.py`, you would run it with octo.py like this:

On the server (assuming an IP address 192.168.0.10):
`octo.py server triangular.py`

On each client:
`octo.py client 192.168.0.10`

That's it! No network code, no partitioning of tasks, just one variable and three functions.