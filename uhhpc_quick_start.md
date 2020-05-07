# UHHPC: Quick Start

This is a quick start with an example of how to use the UH cluster with python.

First, have a look through the [documentation](https://uhhpc.herts.ac.uk/wiki/index.php/Main_Page). It's a bit overwhelming at first, but gets easier. Seriously, this guide gives you the minimum info to get started with the cluster, but you should definitely read the wiki at some point.

**TL;DR:** (also, see the guide below)

1. Keep your account private, you're responsible for it and only you should use it.
2. Headnode is a doorway everyone uses, don't block it. Don't run lengthy code directly on the headnode, run the code either in [interactive mode](https://uhhpc.herts.ac.uk/wiki/index.php/Interactive_jobs) or [submit jobs to the queue](https://uhhpc.herts.ac.uk/wiki/index.php/Jobs). You can, however, run something briefly for testing purposes.
3. Don't clutter the cluster. If you have terabytes of data, consider some better solutions for storage and backup. Cluster is not designed for long term storage and you will be asked or forced to delete files if needed (see [Storage](https://uhhpc.herts.ac.uk/wiki/index.php/Storage) for more info).
4. Ask for a reasonable amount of time and computing power. The less you ask for, the earlier the job will run. Adding jobs to the queue is also a more efficient use of resources. The maximum time you can ask for without prior agreement is 1 week (168 hours).
5. [Say thank you in you acknowledgements](https://uhhpc.herts.ac.uk/wiki/index.php/Acknowledgements) when you publish.

## Modes of use

There are two ways of using the cluster, via the [queue](https://uhhpc.herts.ac.uk/wiki/index.php/Jobs) and [interactively](https://uhhpc.herts.ac.uk/wiki/index.php/Interactive_jobs). Both ways are described below.

## Adding jobs to the queue

A more detailed guide about jobs is [here](https://uhhpc.herts.ac.uk/wiki/index.php/Jobs). Below are a few basic cases of cluster use.

First, of course, you need to [get an account](https://uhhpc.herts.ac.uk/wiki/index.php/Accounts) if you don't have it yet and `ssh` into the cluster - run `ssh <your_username>@uhhpc.herts.ac.uk` and enter your password when prompted.
To add jobs to the queue, you'll need to [enable passwordless ssh on the cluster](https://uhhpc.herts.ac.uk/wiki/index.php/Passwordless_ssh).

### Basics: single-threaded job

This is just an example of adding job tht will run on a single core. Yes, it seems odd to use a single core of a computer cluster, but it is a legitimate way of running lengthy code in the background.

Let's say we want to generate a table of integer numbers and their corresponding squared values.

1. Create a working directory to keep everything in one place (e.g. `mkdir test; cd test`).

2. Create the Python file that does the thing, named `test.py`.

    ```python
    for x in range(1000):
        print(x, x**2)
    ```

3. One way of giving the cluster some parameters (e.g. `walltime` or `nodes`, see ["Basic commands"](https://uhhpc.herts.ac.uk/wiki/index.php/Jobs#Basic_commands)) is to specify them in the command line. A much better and simpler approach is to specify them in the shell script that will be actually executed (because the cluster doesn't know how to run your `test.py` unless you explicitly tell it how to do it). Create a file named `test.sh`:

    ```bash
    #!/bin/sh
    #PBS -N test
    #PBS -m abe
    #PBS -l nodes=1:ppn=1
    #PBS -l walltime=00:00:30

    cd test
    python test.py
    ```

    Here, before going to the working directory and executing the code, we give the cluster some essential parameters:

    `-N test` - name of the job, `test`, as it will appear in the queue (default: name of the script, i.e. `test.sh`),

    `-m abe` - email the user about the abort, begin and end of the execution, respectively. You can just use one or two of the letters,

    `-l nodes=1:ppn=1` -  number of nodes and processors that we need. A node is a whole separate computer and you can request up to 32 processors for each node (but don't be greedy). Here we need only one.

    `-l walltime=00:00:30` - how long do you think it will take to execute your job? Here we use 30 seconds, but you can ask for as much as 168 hours.

4. Now we have two files in the `test/` directory: `test.py` and `test.sh`. Add the job to the queue by running `qsub test.sh` in the terminal.

5. **Profit.** Now you have two new files in the `test/` directory: `test.eXXXXX` (with a number assigned to the job instead of `XXXXX`) that will contain any unhandled errors (should be empty if everything is good) and `test.oXXXXX` with the output, containing pairs of integers between 0 and 999 and their corresponding squared values.

    This `test.oXXXXX` file contains the terminal output of the program, but you can create your own output file by amending your `test.py` to include writing to a file:

    ```python
    with open('test.txt', 'w+') as output:
        for x in range(1000):
            output.write("{} squared is {}".format(x, x**2))
    ```

    which will produce the `test.txt` file with the output. This way you separate any print statements or warnings from the actual output data.

### Embarrassingly parallel example

Parallel tasks are considered "embarrassing" if it has little or no dependency or need for communication between those parallel tasks, or for results between them.

#### Approach 1

If you have code that you can run multiple instances of, without the need for them to know about each other, you can just run multiple instances of it. E.g. if you are just generating random numbers:

`test.py`:

```python
from numpy.random import normal
import sys

def generate_value():
    return normal(loc=0, scale=1)

normal_distribution = [generate_value() for i in range(int(sys.argv[1]))]
with open('test.txt', 'w+') as output:
    output.write("\n".join(map(str, normal_distribution)))
```

`test.sh`:

```bash
#!/bin/sh
#PBS -N test
#PBS -m abe
#PBS -l nodes=1:ppn=1
#PBS -l walltime=00:00:30

cd test
python test.py 10000
```

You can add this to the queue and each instance will be adding the output to the `test.txt`.

#### Approach 2

A better way would be making python run some code in parallel. This works very well if, for instance, you have a long list of files to process.

One of the simplest to use packages for this task is probably `joblib`. Let's reuse the example with calculating the squared values.

```python
from joblib import Parallel, delayed

def process_number(x):
    return x**2

input_numbers  = range(0, 10000)
output_numbers = Parallel(n_jobs=4)(delayed(process_number)(number) for number in input_numbers)

table = ["{}, {}".format(input_numbers[i], output_numbers[i]) for i in range(len(input_numbers))]
with open('test.txt', 'w+') as output:
    output.write("\n".join(table))
```

The code above calculates the square value for each of the `input_numbers` and saves them to a `test.txt`.

This time, the `test.sh` will have `ppn=4`:

```bash
#!/bin/sh
#PBS -N test
#PBS -m abe
#PBS -l nodes=1:ppn=4
#PBS -l walltime=00:00:30

cd test
python test.py
```

This is a very basic example and if you need something more interesting, check out [`multiprocessing`](https://docs.python.org/3/library/multiprocessing.html) module in Python. It's a bit of a steep learning curve, but worth it if you need it.

## Interactive use

Using the cluster interactively means you are performing your tasks on your PC or laptop terminal, except you are using the compute nodes on the cluster rather than the processors of your PC or laptop. This can be handy if you need to see the execution on the terminal. Another benefit is that you get the exact resources you asked for (e.g. an 8 core node for 48 hours) which is yours to use for anything until the walltime expires and you are not competing with other users jobs for resources. 

### Starting an interactive job

1. ssh into the cluster the usual way: ssh -X uhhpc.
(The -X is the command for displaying graphics, so best to use it as default).

2. You should now be in the headnode. From there, type the following command to start an interactive job:
qsub -l walltime=24:00:00 -l nodes=1:ppn=8 -I -q main

-walltime: This is the length of time you are asking to use one of the compute nodes, which is 24 hours for the example above. The maximum you can ask for at a time is 168 hours (1 week) for the main nodes, and 48 hours for the SMP machines (see below).
-nodes=1: This is stating that you wish to use one compute node. Usually you will only ever need to use one for an interactive job, but in the case that you wish to use multiple you can do so.
-ppn=8: This is stating the number of processors/cores to use for that node. 8 is just an example, and for python scripts even 1 core will suffice. If you know that using more cores will speed up your executions then you may ask for more. Note that asking for 32 cores will give you the upgraded compute nodes which are much faster than the 8 core nodes.
-main: This is stating that you wish to use the main compute nodes. We have other nodes such as the SMP machines (see https://uhhpc.herts.ac.uk/wiki/index.php/SMP_machines), which have 256 GB of RAM (you'll need to do qsub -l walltime=48:00:00 -l nodes=1:ppn=48 -I -q main for this).

The command above should suffice for normal tasks such as running python scripts or running most other things on the cluster. For advanced usage see https://uhhpc.herts.ac.uk/wiki/index.php/Interactive_jobs.

Note that if your terminal is stuck with 'qsub: waiting for job 123456.uhhpc.herts.ac.uk to start' and nothing is happening, this means your request to access a node is in the queue, as the resources you have asked for are all being used by other people. In this case, press Control-C to come out, and then try to access a node again but with less resources (e.g. lower the walltime and number of cores). People that have asked for less resources are higher up in the queue.

**Important**. Do not leave your interactive job idle for days if you are not using it or running anything. Often during peak research time the nodes are all occupied and people can't access any. If you have finished performing tasks please exit the node by typing 'exit' on your terminal. This will take you back to the headnode. 
