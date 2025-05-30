# Project

repo: https://github.com/go-co-op/gocron

## Source code research

### scheduler
interface: `Scheduler`

implement struct: `scheduler`

constructor: `func NewScheduler()`
> will create `scheduler`, then start a goroutine(I call it `bigGR`) to wait multiple signals via multiple chans to trigger some operations.  
> init `scheduler.exec` field as `executor` struct.  

struct: `executor`
> [pending] clock: 

`scheduler` listen chans:

- `startCh`
    > trigger task run
- `scheduler.exec.jobOutRequest`
    > 
- `runJobRequestCh`
### job
interface: `JobDefinition`
> a running timer difinition that determine when you want to run the job.

implement struct: 
- `dailyJobDefinition`
- `weeklyJobDefinition`
- ...

type alias: `Task`: `type Task func() task`
> your concrete business task function.

struct: `task`
> wrap your business function and parameters.

constructor: `NewTask`

interface: `Job`

implement struct: `job`

`scheduler.NewJob` will register your `Task` and `JobDefinition` to scheduler as `Job`.

> interface: `jobSchedule`  
> implement struct: `internalJob`,`dailyJob`, `oneTimeJob`    
>
> init a `internalJob`   
> verify task function and parameters.  
> apply global options and job options  
> append context to task function if necessary  
> call `JobDefinition.setup`   
> push job to `scheduler.newJobCh`, wait  
>> `bigGR` will receive `scheduler.newJobCh` and call `scheduler.selectNewJob`  
>>> `scheduler.selectNewJob` will push the`internalJob` to `scheduler.jobs` map.    
>
> construct `job`  from `internalJob` return as `Job`

#### JobDefinition.setup
- func: `dailyJobDefinition.setup` 
    > translate atTimes to `dailyJob`, and assign to field `jobSchedule` of `internalJob`  
- func: `oneTimeJobDefinition.setup`
    > apply startAt option  
    > translate empty atTimes to `oneTimeJob`, and assign to field `jobSchedule` of `internalJob`  

### trigger scheduler
you call func: `scheduler.Start()` to trigger scheduler.
> push to `scheduler.startCh` and wait `scheduler.startedCh`  
>> `bigGR` will receive `scheduler.startCh` and call `scheduler.selectStart()` that will push to `scheduler.startedCh`  

func `scheduler.selectStart()`
> start a goroutine(I call it `execGR`) to run `scheduler.exec.start()` 
> loop all jobs in `scheduler.jobs` map    
>> trigger the first run of job  
>> push to `scheduler.exec.jobsIn`: `execGR` will receive from.  
> push to `scheduler.startedCh`

goroutine `execGR`
> loop forever, block wait chan `scheduler.exec.jobsIn`    
> [pending] check limitMode  
> run a goroutine(I call it `execInternalGR`)

goroutine `execInternalGR`
> [pending] check limitMode  
> call func `requestJobCtx`: get job by id from `scheduler.jobs` map  
> check if the job is singletonMode  
>> if the job is singletonMode, init the `singletonRunner`
>>> run a goroutine(I call it `execSingletonRunnerGR`)  
>>> check limit mode if is 1
>>>> select wait push chan `singletonRunner.rescheduleLimiter`   
>>>>> if can push (it means singleton has no job running)  
>>>>>> push chan `singletonRunner.in` and call func `executor.sendOutForRescheduling`  
>>>>
>>>>> else  
>>>>>> call func `executor.sendOutForRescheduling`  
>>>
>>> if not 1 
>>>> just push to chan `singletonRunner.in` and call func `executor.sendOutForRescheduling`  
>> 
>> else  
>>> call func `executor.runJob`

goroutine `execSingletonRunnerGR`
> forever loop wait chan `singletonRunner.in`  
> trigger func `executor.runJob`  
> use chan `singletonRunner.rescheduleLimiter` to limit running jobs count to 1


func `requestJobCtx`: get job by id from `scheduler.jobs` map  
> push to chan `scheduler.exec.jobOutRequest` and wait `item.outChan`  
> `scheduler.exec.jobOutRequest`  
> `bigGR` will receive `scheduler.exec.jobOutRequest` and call func `scheduler.selectJobOutRequest`    
>> func `scheduler.selectJobOutRequest` will push to `item.outChan` in chan `scheduler.exec.jobOutRequest` then close.    
>> (which is the registered internalJob in `scheduler.jobs` map)   




### executor
func `executor.runJob`
> elector  
> locker  
> call before listeners  
> call task function  
> call after listeners  

func `executor.sendOutForRescheduling`
> push to chan `executor.jobsOutForRescheduling`
> `bigGR` will receive `scheduler.exec.jobsOutForRescheduling`, and compute next time to run the job and run after that.  
> for dailyJob, if not provide atTimes, then will run immediately after scheduler started, and never trigger again in the future.

#### [pending] job options
