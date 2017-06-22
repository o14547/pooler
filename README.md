# **[Parallel Pooler](https://www.nuget.org/packages/Pooler)**

[![Latest Stable Version](https://img.shields.io/badge/Stable-v1.1.2-brightgreen.svg?style=plastic)](https://github.com/tomFlidr/desharp/releases)
[![License](https://img.shields.io/badge/Licence-BSD-brightgreen.svg?style=plastic)](https://raw.githubusercontent.com/parallel-pooler/parallel-pooler/master/LICENSE)
![.NET Version](https://img.shields.io/badge/.NET->=4.0-brightgreen.svg?style=plastic)

.NET parallel tasks executing library.

## Instalation
```nuget
PM> Install-Package Pooler
```

## Examples
1. Basics - create new pool and run tasks
2. Add delegate tasks or functions returning results
3. Use Pooler events
4. Changing threads count at run
5. Regulating CPU and resources load
6. Ways to creating parallel pooler instance
7. Stop processing
8. Async tasks

### 1. Basics - create new pool and run tasks
```cs
using Parallel;

// create new threads pool instance for max 10 threads running simultaneously:
Pooler pool = Pooler.CreateNew(10);

// add 500 anonymous functions to process:
for (int i = 0; i < 500; i++) {
	pool.Add(
		// any delegate or void to process
		(Pooler pool) => {
			double dummyResult = Math.Pow(Math.PI, Math.PI);
		},
		// optional - do not run task instantly after adding, run them a few lines later together
		false,
		// optional - background execution thread priority to execute this task
		ThreadPriority.Lowest, 
		// optional - task is not async (task doesn't use any background threads inside)
		false
	);
}
// let's process all tasks in 10 simultaneously running threads in background:
pool.StartProcessing();
```

### 2. Add delegate tasks or functions returning results
```cs
// There is possible to add task as delegate:
pool.Add(delegate {
	double dummyResult = Math.Pow(Math.PI, Math.PI);
});
// ... or to add task as any void function accepting first param as Pooler type:
pool.Add((Pooler p) => {
	double dummyResult = Math.Pow(Math.PI, Math.PI);
});

// ... or to add task as Func<Pooler, object> function:
// accepting first param as Pooler type and returning any result:
pool.Add((Pooler p) => {
	return Math.Pow(Math.PI, Math.PI);
});
// ... then you can pick up returned result in pool.TaskDone event:
pool.TaskDone += (Pooler p, PoolerTaskDoneEventArgs e)) => {
	double dummyResult = (double)e.TaskResult;
};
```

### 3. Use Pooler events
`pool.TaskDone` event is triggered after each task has been executed (successfuly or with exception):
```cs
pool.TaskDone += (Pooler p, PoolerTaskDoneEventArgs e)) => {
	Console.WriteLine("Single task has been executed.");
	// e.TaskResult [object] - any place for your task result data:
	Console.WriteLine("Task returned result: " + e.TaskResult);
	// e.RunningThreadsCount [Int32]
	Console.WriteLine("Currently running threads count: " + e.RunningThreadsCount);
};
```
`pool.ThreadException` event is triggered immediately when exception inside executing task is catched, before TaskDone event:
```cs
pool.ThreadException += (Pooler p, PoolerExceptionEventArgs e) => {
	Console.WriteLine("Catched exception during task execution.");
	
	// e.Exception [Exception]:
	Console.WriteLine(e.Exception.Message);
};
```
`pool.AllDone` event is triggered after all tasks in pooler store has been executed:
```cs
pool.AllDone += (Pooler p, PoolerAllDoneEventArgs e) => {
	Console.WriteLine("All tasks has been executed.");	
	
	// e.Exceptions [List<Exception>]:
	Console.WriteLine("Catched exceptions count: " + e.Exceptions.Count);
	
	// e.PeakThreadsCount [Int32]:
	Console.WriteLine("Max. running threads peak: " + e.PeakThreadsCount);
	
	// e.ExecutedTasksCount [Int32]:
	Console.WriteLine("Successfully executed tasks count: " + e.ExecutedTasksCount);
	
	// e.NotExecutedTasksCount [Int32]:
	// Not executed (aborted) tasks count by possible pool.StopProcessing(); call:
	Console.WriteLine("Not executed (aborted) tasks count: " + e.NotExecutedTasksCount);
};
```

### 4. Changing running threads count in run
There is possible to change processing background threads count at game play any time you want, Pooler is using locks internaly to manage that properly:
```cs
pool.SetMaxRunningTasks(
	// new threads maximum to process all tasks in background
	50,
	// optional (true by default), to create and run all new background threads 
	// imediatelly inside this function call. If false, each new background thread 
	// to create to fill new maximum will be created after any running thread process 
	// execute it's current task, so there should not to be that increasing heap..
	true
);
```
There is also possible to get currently running maximum. Currently running maximum is not the same as currently running threads count.
```cs
int maxThreads = pool.GetMaxRunningTasks();
```

### 5. Regulating CPU and resources load
For .NET code, there is CPU load and any other resources load for your threads managed by operating system, so the only option how to manage load from .NET code is to sleep sometime. Then you can have some free system resources for another jobs, for example you need to have another free 20% CPU computation capacity. To manage sleeping and sleeping time in your tasks globaly by pool, you can use this:

1. Set up pausing time globaly for all tasks, any time you want, before processing or any time at run to cut CPU or other resources load:
```cs
pool.SetPauseMiliseconds(100);

// And you can read it any time of course:
int pausemiliseconds = pool.GetPauseMiliseconds();
```

2. Use `pool.Pause();` method sometimes in your hard task:
```cs
pool.Add((Pooler p) => {
	double someHardCode1 = Math.Pow(Math.PI, Math.PI);
	p.Pause();
	double someHardCode2 = Math.Pow(Math.PI, Math.PI);
	p.Pause();
	double someHardCode3 = Math.Pow(Math.PI, Math.PI);
});
```
Now resources should not to be so bussy as before, try to put there harder code to process, increase pause time or try to use [WinForms Test Application](https://github.com/parallel-pooler/winforms-application-test).

### 6. Ways to creating parallel pooler instance

Creating new instance by static factory or by new Pooler:
```cs
Pooler pool;
pool = Pooler.CreateNew(10, 100);
pool = new Pooler(10, 100);
```
First (optional) param is max. threads in background to executing all tasks. 10 by default.
Second (optional) param is pause miliseconds to slow down CPU load or other resources by `pool.Pause();` calls inside your tasks, 0 by default.

There is also possible to use single static instance from Pooler._instance by:
```cs
Pooler pool = Pooler.GetStaticInstance(10, 100);

// to get the same instance any time again, 
// just call it without params:
pool = Pooler.GetStaticInstance();

```

### 7. Stop processing
First optinal param (true by default) is to heavy abort - all background threads are aborted by `bgThread.Abort();`, what should be dangerous for your task. So switch this to `false` to let all running background threads go to their natural task end and than abort.
```cs
pool.StopProcessing(true);
```

### 8. Async tasks
To use any other threads or async code in your pool tasks, you need to tell pooler at the end of async task code that you are done by `pool.AsyncTaskDone();` or `pool.AsyncTaskDone(resultObject);`:
```cs
pool.Add(
	// any delegate or void to process
	(Pooler pool) => {
		// some async code start here:
		CustomDownloader client = new CustomDownloader(
			"http://example.com/something/what/takes/some/time/to/load"
		);
		client.Loaded += (object sender, EventArgs e) => {
			// not call pool to continue executing 
			// another tasks by this bg thread:
			pool.AsyncTaskDone(sender);
		};
		client.Load();
	},
	false,
	ThreadPriority.Lowest, 
	// true - task is async!
	true
);
pool.StartProcessing();
```
