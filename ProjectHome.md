Inspired by Google's [MapReduce](http://labs.google.com/papers/mapreduce.html) and [Starfish](http://tech.rufy.com/2006/08/mapreduce-for-ruby-ridiculously-easy.html) for Ruby, octo.py is a fast-n-easy MapReduce implementation for Python.

Octo.py doesn't aim to meet all your distributed computing needs, but its simple approach is amendable to a large proportion of parallelizable tasks. If your code has a for-loop, there's a good chance that you can make it distributed with just a few small changes. If you're already using Python's map() and reduce() functions, the changes needed are trivial!

It is not an exact clone of the Big-G's MapReduce, but I'm guessing that you aren't operating a Google-like cluster with a distributed Google File System and can't use a MapReduce clone. Instead, the scope of the project is more akin to Starfish, running on an ad-hoc cluster of computers. The data semantics bears closer resemblance to MapReduce though, except the part about the ordering of intermediate results.

For examples, look at UsageExamples. For detailed usage instructions, take a look at UsersGuide. And if you're interested in modifying the source, take a look at DevelopersGuide.