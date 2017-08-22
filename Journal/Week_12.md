# Week 12 (17 August to 28 August)

## Summary

## Progress

### Scheduling inputs into pipeline

Last week I have managed to establish an example pipeline that handled 
several inputs and use one at a time (see [here](https://github.com/bionode/GSoC17/blob/master/Journal/Week_11.md#between-tasks)). 
However, this approach can exhaust all available resources in a given 
computer/server if too many files are given as input. Therefore it urged to 
characterize a simple way in which anyone can limit this I/O process.

One way to do this is by using `bluebird` npm module.

```javascript
// calls bluebird
const Promise = require('bluebird')
// makes concurrency the third argument of code in shell
const concurrency = parseFloat(process.argv[2] || "Infinity")

// [...] pipeline stuff
// Then after pipeline definition...

// checks for files in current working directory (where the script is executed)
fs.readdir(process.cwd(), (err, files) => {
  const desiredFiles = []
  files.forEach(infile => {
    // checks file extension, in this case '.fas' and creates an array with
    // just these files
    if (path.extname(infile) === '.fas') {
      desiredFiles.push(infile)
    }
  })
  // bluebird Promise.map is used here for concurrency where we can
  // limit the flow of files into the pipeline. So, if concurrency is set
  // to 1 it will first execute the first file, then the second and so on.
  // But if 2 it will execute the first 2 files and then the other 2.
  return Promise.map(desiredFiles, (infile) => {
    const pipelineMaster = fileFlow(infile)
    return pipelineMaster()
  }, {concurrency}) // in ES6 there is no need to repeat {concurrency: concurrency}
})
```

With this approach users still have to take in account programs cpu usage 
within each task and then see how many files can be processed at a time. 
Users should avoid having more processes fired by bionode-watermill than the 
available cpu usage.

Some examples (imagining a hypothetical server with 20 cpus available):

* 10 files and one of the tasks (the one that uses more cpus) in the pipeline 
use 5 threads.
    * only 4 files should be processed at the same time, therefore 
    `concurrency` should be **4** (4*5 = 20).

* 10 files and one of the tasks (the one that uses more cpus) in the pipeline
 use 2 threads.
    * The 10 files can processed all at once without limiting concurrency 
    (10*2 = 20).
    
* 1000 files and all tasks use one cpu.
    * batches of 20 files should be passed because the limit would be 20 cpus
     and you can have 20 files running same tasks at the same time.
     
Also, this concurrency parameter can be passed through `process.argv` instead
 of hardcoding it inside the script.js.


### Fixing fork nested in join inside another fork

The following pipeline:

```javascript
const pipeline2 = join(
  task0,
  fork(
    join(
      task4,
        fork(
          task1,
          task3
        ),
      task6
    ),
    task2
  ),
  task5
)
```

Wasn't being properly executed, because `downstreamTasks` were not being 
duplicated as they were supposed to. In commit [2934325](https://github.com/bionode/bionode-watermill/commit/293432584712d30f913c87c7611b3722ade55ae5)
this behavior was fixed. Basically, `outermostTasks.info` and 
`downStreamTasks.info` were `undefined` because `outermostTasks` and 
`downStreamTasks` are in fact arrays rather than task objects where `.info` 
is accessible. Therefore it was needed a way to compare both these arrays, 
but even when tasks are the same their `uid` might differ, depending on its 
`context.trajectory`. Therefore the only variable that is defined and 
different in different tasks is `operationCreator`. Hence, I used each task 
`operationCreator` in both arrays to generate an hash for each array. Then, 
when the hashes from the two arrays (`outermostTasks` and `downStreamTasks`) 
is equal we have a fork followed by another fork. But if these two hashes are
 different then we have a fork nested in a join inside another fork.
 
 So if equal it will properly render this pipeline:
 
 ```javascript
const pipeline3 = join(
task0,
fork(
    fork(
      task1,
      task3
    ),
    task6
),
task5
)
```

but if the two hashes are different, it will execute the code for a pipeline 
like this:

```javascript
const pipeline2 = join(
  task0,
  fork(
    join(
      task4,
        fork(
          task1,
          task3
        ),
      task6
    ),
    task2
  ),
  task5
)
```

This issue was already reference in [previous week](https://github.com/bionode/GSoC17/blob/master/Journal/Week_11.md#forkception),
although `downstreamTasks` where absent from the tests made. And here notice 
that `task6` is a task that is inside the `downstreamTasks` array and it was 
not being added to the graph because there was no code to properly handle this.

Expected result:

![](https://github.com/bionode/GSoC17/blob/master/Experimental_code/Experimental_Pipelines/fork_fork/fork_join_fork_fix_exp.png)

Actual result:

![](https://github.com/bionode/GSoC17/blob/master/Experimental_code/Experimental_Pipelines/fork_fork/fork_join_fork_fix_res.png)

### Tests for forkception

#### fork inside fork, wrapped in join

Added a test that tests for the existance of a fork inside another fork and 
check if it renders the appropriate number of edges and vertices
 in 
[this commit](https://github.com/bionode/bionode-watermill/pull/73/commits/aa182b27ed683c40d244ea607500e63c982726f6#diff-5922682f49c3344eec99d28dd70031ec).

This example should work:

```javascript
const pipeline = join(task1, fork(task2, join(task2, fork(task3, task4))), 
tasak5)
```

#### fork inside fork, without join wrapping

A similar test, that checks for edges and vertices, was added to check if 
this works properly ([commit](https://github.com/bionode/bionode-watermill/pull/73/commits/aa182b27ed683c40d244ea607500e63c982726f6#diff-67b4d5d3ea1c32216d4133d376f5d6c5)).

Although this might be an odd pipeline it should be properly resolved:

```javascript
const pipeline = join(task1, fork(task2, fork(task3, task4)), task5)
```

### junction inside fork

Another test was added to check junction when inside a fork (see [this commit](https://github.com/bionode/bionode-watermill/pull/73/commits/aa182b27ed683c40d244ea607500e63c982726f6#diff-e28ce03431eaab6457a088286cec0158))

This tests the execution of a pipeline like this:

```javascript
const pipeline = join(
  task0,
  fork(
    join(
      task1,
      junction(
        task2,
        task3
      ),
      task4
    ),
    task5
  ),
  task6
)
```

This test, checks if a junction node exists and if both end nodes of junction
 exist. It also checks if the expected number of edges and vertices exist 
 after the pipeline execution

#### final note on all these tests

All these tests rely on the execution of each others because the total number
 of nodes and vertices for tests are cumulative, i.e., each pipeline executed
  in tests are being added to graph, as if it is a single pipeline with 
  multiple subpipelines. This behavior was unintentional but it unveiled a 
  nice hint on how to handle a pipeline that runs a bunch of tasks in a `fork` 
  like manner until the end of fork shape, but that can take middle tasks to 
  perform other tasks.
  
  So something like this:
  
  ```javascript
// pipeline that executes something using fork
const pipeline = join(task0, fork(task1, task2), task3)

// then other task to be executed that joins the results from task1 and task2
const pipeline2 = someOtherTask

// tentative execution

pipeline.then(pipeline2)
```