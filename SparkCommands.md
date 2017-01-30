# To run Spark

```
sudo -u spark spark-shell
```

# In the Spark shell
Type
```
sc.[TAB]
```
to see a list of methods available on the context.  __Note:__  You should be able to do this for any variable.

# Loading files in local mode
Start the shell:
```
spark-shell --master local[n|*] --driver-memory 2g
```

Load the data from the local file system using fully-qualified paths; tilde replacement doesn't work.  Check the path specified as output from the RDD creation.

_first_ is a good method to sanity check the load; _collect_ is good for bringing in small datasets as an array.
