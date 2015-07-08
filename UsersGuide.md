Interested in using octo.py? You're in the right place.

# Introduction #

Octo.py was intended to make distributed computing easy for Python programmers. There are only a few simple requirements:

  * A Python installation
  * A program you want to parallelize
  * Several computers connected by a network
  * octo.py itself
  * A parallel computing mindset

Of all these requirements, the last is probably the hardest. This User's Guide will walk you through the process of parallelizing a task and running octo.py.

# Your first octo.py #

## Word counts ##

This example is borrowed from the Google paper on MapReduce. The task is to count the occurrences of words in a collection of documents.

We will try to count the word frequencies in books from the Project Gutenberg. I'll be using the March 2007 Sci-fi CD collection, but any of the collections from the Project Gutenberg [download page](http://www.gutenberg.org/wiki/Gutenberg:The_CD_and_DVD_Project) will do.

## Parts of an octo.py script ##

An octo.py script has 4 parts: the source of the data, a map function, a reduce function, and a function that's called on the final results.

## Data source ##
The source of the data is a dictionary, or dictionary-like object, that contains a list of keys and values. In this case, the keys will be the file names of the books from Project Gutenberg and the values will be the actual contents of the books.

The Sci-fi collection weighs in at 153 MB, not too large to fit entirely in memory for most computers. This makes things a little easier for us. We'll take a look at how to deal with large data sets, too big for main memory, in the advanced section of this guide. For now, everything will be loaded into memory.

To let octo.py know of our data source, the dictionary will have to be stored in a variable called `source`. We'll use Python's `glob` module to get a list of all the text files (those with the `.txt` extension) in the directory. We'll then loop through each file, read the contents, and store the file name and contents in the dictionary.

```
import glob

text_files = glob.glob('Gutenberg SF/*.txt')

def file_contents(file_name):
    f = open(file_name)
    try:
        return f.read()
    finally:
        f.close()

source = dict((file_name, file_contents(file_name))
              for file_name in text_files)
```

That's it for `source`!

## Splitting word boundaries ##

Our map function will take each book's contents, split it up into words, and return each word as an _intermediate result_. Octo.py requires the map function be called `mapfn()` and (like MapReduce) that each intermediate result be a key-value pair. We'll just return a 1 as the other half of the pair, i.e. (_word_, 1).

To determine word boundaries, we will just use Python's built-in `split()` function for strings. There are more accurate ways to determine word boundaries, but for the purposes of this example, we will just use the simple method. We will also normalize the words using `lower()`.

```
def mapfn(key, value):
    for line in value.splitlines():
        for word in line.split():
            yield word.lower(), 1
```

If you're not familiar with the `yield` statement, I encourage you to read the Python [reference](http://docs.python.org/ref/yield.html) and [PEP](http://docs.python.org/whatsnew/pep-342.html) on it. We could store all the words in a list and return that, but that would take a lot of memory---even more than the memory used to store the entire book.

## Counting the words ##

Next up is the reduce function, called, rather plainly, `reducefn()`. Octo.py will give the reduce function a key-value pair derived from the intermediate results.

`mapfn()` already produces intermediate key-value pairs of (_word_, 1). Octo.py will then combine all values with the same key into a list. The intermediate key, and the list of intermediate values are the ones passed to `reducefn()`. In this case, the key-value given to `reducefn()` will be something like (_word_, [1, 1, 1, 1, ..., 1, 1, 1]). The job of the reduce function is to sum/count these tallies and return a key-value pair indicating the count for each word.

```
def reducefn(key, value):
    return key, len(value)
```

Now we have the number of occurrences of every word. That was easy wasn't it? But what will we do with these results? That's where `final()` comes in. For this simple program, we will just print out the counts of all the words.

## Printing the results ##

```
def final(key, value):
    print key, value
```

Notice that `final()` is also called with the key-value pair produced from `reducefn()`.

That's all to it. You can save the code as `wordcount.py` or download the entire source of this program in the **Downloads** section.

## Running the script ##

You will need a copy of octo.py on each computer you plan to use for counting. If you don't have any other computer handy, you can just run multiple copies of octo.py on a single computer just to see how it works.

One computer (or copy of octo.py) will be the "server." It is responsible for producing the initial data, assigning the tasks, and finally collating all the data. The "clients" will just do what the server tells them to do.

On the server, we run octo.py with the `wordcount.py` script like this:

`python octo.py server wordcount.py`

(Details of running a Python script can be found in the documentation appropriate for your platform.)

That will start the octo.py server. It will take a moment to get a list of the books and to read all of them to memory. When it's done, we can start the clients:

`python octo.py client SERVER_ADDRESS`

Where `SERVER_ADDRESS` is the host name or IP address of the server. The octo.py client will automatically download a copy of the `wordcount.py` script and then ask the server what to do. If you're running all the copies on a single computer, you would type:

`python octo.py client localhost`

## What runs where ##

The server is only concerned with the `source` and `final()` parts of the script, while the clients run `mapfn()` and `reducefn()`. This is why we put the `print` statement in `final()` instead of `reducefn()`---if we didn't, the final results will be printed on each of the respective clients instead of being collated by the server.

Depending on the speed of the cluster (or single computer), this will take a short while or much longer and eventually you will see the words and their counts being displayed on the server.

Notice that you don't need to worry about network issues (other than the IP address of the server), or synchronization issues typical of multi-threaded programs. This is all handled transparently by octo.py. All you need to worry about is the actual computation itself, and that only involves a handful of methods.

# How to distribute everything (and maybe solve world hunger in the process) #

WORK IN PROGRESS

# Behind the scenes #

## Task scheduling ##

Octo.py assigns tasks on a first-come-first-serve basis. Initially, the tasks are all map operations on the `source`. Each client that connects will get a task, work on the task, and report the results. The server will store these intermediate results. Then the cycle repeats until all map tasks are done. The server then switches to the reduce phase and begins handing out reduce jobs, based on the intermediate results, to the clients. For the clients, the reduce jobs happen much the same way. And when the last reduce job is done, the server checks for a `final()` function. If one is available, it will be run on the final results.

The current implementation of octo.py assumes that all the clients are reliable and will report back with results. It will not retry tasks or assign a single task to multiple clients. This will likely change in the near future.

## Network protocol ##

Octo.py currently supports only one method for communication between the client and the server. This is using its own internal network protocol. In future, other protocols like SSH and HTTP may be added. You may also add your own by extending `TaskServer` and `TaskClient`. Refer to the DevelopersGuide for more information.

## Large datasets ##

There are several issues with large datasets, but only one thing to remember: we are _not_ Google. Octo.py is targeted as smaller problems and does not strive to mimic MapReduce exactly. Handling large data sets is possible, but it isn't handled as transparently as with MapReduce. Future versions of octo.py may offer some additional functionality to alleviate this.

### Memory ###

The first issue with large data sets is memory. The word-counting example above loads all the data into the memory of the server. Hardly efficient, considering the data is most likely used only once. An alternative method is to present a dictionary-like object that loads the data on demand:

```
class DataSource(object):
    def __init__(self):
        self.text_files = glob.glob('Gutenberg SF/*.txt')

    def __len__(self):
        return len(self.text_files)

    def __iter__(self):
        return iter(self.text_files)

    def __getitem__(self, key):
        f = open(key)
        try:
            return f.read()
        finally:
            f.close()

source = DataSource()
```

Here, the `DataSource` object maintains only a list of file names. As each file name is retrieved via `__getitem__()`, it is opened, read, and the contents returned. You don't need to implement all dictionary methods, only `__iter__()` and `__getitem__()`.

### Network ###

The next issue with large datasets is the network. MapReduce has the distributed Google File System to help it, but octo.py uses a centralized server to distribute and store all results and this will be the bottleneck for network operations.

The current network protocol does compress data, so that the issue isn't so obvious with the textual data in the word-counting example above. However, if you're going to do the same with large images, say, for medical image processing, you will find that the technique above doesn't work very well. `source` aside, large intermediate datasets also pose a problem. The server has to store all intermediate data from each client so that it can group the data by the keys. This requires a lot of space on the server, but that's minor compared to the wasted bandwidth since the same data will be sent from client to server during the map phase, and again from server to client during the reduce phase.

Octo.py works best if you consider the task of distributing the computation separate from the task of distributing the data. Octo.py will handle the computational aspect, but something else will have to be done about the data. One way would be to use a dedicated file server, or even multiple file servers, with each client being its own file server. Instead of passing the data around, you could just send the complete paths of the data files. This will work for the intermediate data too, if the intermediate data is written to globally accessible and globally unique paths. This is somewhat similar to what MapReduce and GFS do.