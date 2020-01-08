**Study Notes for exam 70-483 – Programming in C#**

# Manage Program Flow

## Threads

If you mark a field with [ThreadStatic] then each thread has its own copy of the variable, otherwise they will share it.

ThreadStart delegate can be ParameterizedThreadStart – new Thread(()=>{…})

Thread.CurrentThread static property gives you access to the currently execting thread

- Can then access Culture, Principal, Priority

ThreadPool is used to re-use threads so they don&#39;t have to be created and the destroyed

ThreadPool.QueueUserWorkItem((s)=>{…});

- In a thread pool requests only get processed when a thread is available to handle it, so resources are managed. You should not rely on shared state with thread pool threads

Thread.Abort() instantly terminates the thread

Thread.Join() –caller waits for thread to complete before continuing

ThreadLocal<T> can be used when each thread needs its own local data

- ThreadLocal<Random> RandomGen = new ThreadLocal<Random>(()=>…..)

ThreadPool

- Should not use local state variables because the state is never cleared
- You cannot obtain a foreground thread from the threadpool
- Cannot manage thread priority
- You can block the threadpool if you create a lot of threads

## Tasks

Favour Task.Run() over Task.Factory.StartNew().Task.Run is a simpler way to call Task.Factory.StartNew with some of the defaults already set.

Task.Run(action) is the same as Task.Factory.StartNew(action, CancellationToken.None,TaskCreationOptions.DenyChildAttach, TaskScheduler.Default);

- In the above if you use TaskCreationOptions.LongRunning you can run the task on a new thread not in the threadpool

Executing a Task on another thread only makes sense if you wish to keep the UI free for other work or if you want to parallelise your work on multiple processors.

Task t = Task.Run(()=>{…});

t.wait() //equivalent to calling thread.join()

=> calling t.Result causes the thread to block and wait for the result to come back

TaskCompletionSource<T>

- Can wrap an external async process and enables creation of a task that consumers can await. This is useful for Async process created using legacy methods
- var tcs = new TaskCompletionSource<bool>();
- tcs.SetResult() //returns the async result also tcs.SetException() tcs.SetCancelled()
- tcs.Task returns the task

Cancel tasks with CancellationToken
<pre><code>
CancellationTokenSource source = new CancellationTokenSource();

CancellationToken token = source.Token;

Task.Run(()=> {while !token.IsCancellationRequested{//do stuff},token);

</code></pre>
Then source.Cancel to signal to cancel the task, which will throw CancelledOperationException

You cannot update the UI from within a task, you will get a Marshalling Exception

### Async

Async void methods cannot throw exceptions that can be caught. Use async Task instead.

Awaited methods create a state machine and save the context of the thread, the rest of the method is saved as a continuation delegate, so that when the awaited method finishes it marshals back to the thread context and executes the rest of the code.

Use async for IO bound operations, not for CPU-bound operations – this means you can have IO operations and code execution overlapping.

Use Task.Run for cpu bound operations

You cannot use in, ref, out modifiers in async methods (or ones with yield return)

### In Modifier

In modifier (c#7.2) is like ref, except that in arguments cannot be modified in that method

<pre><code>
void Test(in int number)
</code></pre>
{ //cannot set value of number}

## Parallel

Use Parallel class only when code doesn&#39;t have to be executed sequentially

Parallel ForEach runs multiple threads in parallel for different items

Parallel.ForEach(numbers;i=>{Thread.Sleep(1000);});

ParallelLoopState – can cancel Parallel loop with Break() or Stop()

- For(0,1000,(int i; ParallelLoopState loopState)=>{….if(???) loopState.Break()))

## Concurrent Collections

ConcurrentQueue<T> is faster and scales better than Queue<T>.

- If the processing time for each element is large and you have a dedicated producer queue and a dedicated consumer queue
- In a mixed producer-consumer, if the processing time per item is small then Queue<T> is faster

ConcurrentStack<T> is faster than Stack<T> in mixed producer consumer scenarios.

ConcurrentDictionary<TKey, TValue> is faster if you are doing many updates, but not by much if there are not many reads.

ConcurrentBag<T> will be slower than other collections in pure producer/consumer scenarios.

- In mixed scenarios ConcurrentBag<T> is generally faster than any other concurrent collection type.
- Is useful for creating a pool of objects where those objects are relatively expensive

BlockingCollection<T> is fastest, when bounding (e.g. size limit) and blocking semantics are required. Also supports rich cancellation, enumeration, and exception handling

## PLINQ

Queries on multiple threads and then combines back to original thread

<pre><code>
var numbers = Enumerable.Range(0,10000000);

var parallelResult = numbers.AsParallel().Where(i=>i%2==0).ToArray()

</code></pre>
It only makes sense for expensive operations to be to the right of AsParallel()

The runtime will determine whether the query should be made parallel

The runtime will generate Task objects and start executing them

You can use ForAll() to iterate over a collection in a parallel way

- parallelResult.ForAll(e=>Console.Write(e));

### Locking

You cannot use a value type for a lock object

volatile keyword stops the compiler from re-ordering lines of code

### Interlocked

Makes operations atomic

Interlocked.Increment() and Interlocked.Decrement(), create an atomic operation out of n=n+1; n=n-1 (which is a read and then a write)


<pre><code>
int n=0;

var up = Task.Run(()=> {for (int i=0; i<1000000;i++); Interlocked.Increment(ref n)});
</code></pre>
Also Interlocked.Exchange() – retrieves current value and sets it to new value in the same operation

Interlocked.CompareExchange(ref value, newValue, compareTo)

### Exceptions

Finally block is not executed in 2 cases

1. 1)code is stuck in an infinite loop
2. 2)Environment.FailFast is used in the try block

You can now use when clause when catching an exception to filter for a particular condition

- Catch(CalcException ce) when (ce.Error==CalcException.CalcErrorCode.BadError) //filter condition

Aggregate Exceptions

They contain a list of exceptions in InnerExceptions property

<pre><code>
try{….}

catch(AggregateException ag) { foreach (Exception e in ag.InnerExceptions\_){….}}

</code></pre>
e.g. in PLINQ event handlers

### SqlException

Class – returns the severity level of the error

Errors – collection of SqlError objects that give info about error

FileIOPermission.RevertAssert() – removes all permission assertions

RevertDeny() – removes all permission denials

RevertAll() – removes both

Trace.Switch:

0 = off

1 = Error

2 = Warning

3 = Info

4 = Verbose

Caspol -  a tool that lets you modify an assembly&#39;s permissions

### General Information

Static constructor – is called only once before any instances of the class are created. Good place to load resources and initialise values

CopyConstructor – is where you pass an instance of a class into a constructor and it returns a new instance with the same values;

### Dynamic

Dynamic  uses ExpandoObjects

e.g. dynamic person = new ExpandoObject();

person.Name=&quot;….&quot;;

person.Age=21;

You can query an Expando object with linq;

expandoObjects also expose IDictionary interface

### IEnumerable

Call GetEnumerator() to return an enumerator

Enumerator supports Current property and MoveNext() method

Foreach automatically fetches enumerator

Inside GetEnumerator() you can use the yield keyword to return the value at the current iteration and returns control to the iterating method.

Public IEnumerator<int> GetEnumerator(){yield return 1; yield return 2; yield return 3;}

IDisposable – provides a mechanism for releasing unmanaged resources in Dispose() method.

IUnkown – is used by .Net objects interacting with COM objects.

### XML Serialization

You can only serialize a class that has a parameterless constructor

Only public fields are serialized

### RegularExpressions

String replaced = RegEx.Replace(input, regular ExptoMatch, patternToReplace)

### PerformanceCounter

If you instantiate a counter like so:

Var myCounter = new PerformanceCounter(categoryName, counterName); they are read only. There is a further constructor overload where you can set the readonly property.

Use Increment() or IncrementBy() to change it

### WinMD Assembly

Windows MetaData files –make operating system apis available to .net

New Project | Windows Runtime Component

- Will generate a WinMD file

Must use Windows runtime types for all member variables, fields and return values

Cannot use generics

Can only derive classes from Windows Runtime

Public classes must be sealed

Public structs can only contain public members: strings or value types

The namespace of a public type must match the assembly name

Public types must not start with &quot;Windows&quot;

### Compiler Directives

#pragma warning disable – disables compiler warnings.

#line sets line numbers of statements (and filename). Can also cause statements to be hidden from the debugger

#line default –resumes normal file name and line numbering

[Debugger StepThrough]

- Debugger won&#39;t hit any statements in method or class

XmlReader

Provides forward only access to XML

Using (xmlReader reader = XmlReader.Create(stream, settings))

 While(await reader.ReadAsync())

Use XmlReaderSettings class to specify validation

Read() reads one node at a time

MoveToContent() skips non content nodes and moves to the next content node or eof

ReadSubTree() reads an element and its children

XMLReader methods:

MoveToFirstAttribute() NextAttribute ToAttribute()

MoveToElement() moves to the element that owns the current attribute

ReadAttributeValue()

ReadToFollowing() – advances the reader to the next element that matches the parameter

### FileStream

<pre><code>
using (FileStream outStream = new FileStream(&quot;out.txt&quot;, FileMode.OpenOrCreate, FileAccess.Write)

{ string outMessage=&quot;….&quot;;

Byte[] outMessageBytes = Encoding.UTF8.GetBytes(out message);

outStream.Write(otMessageBytes,0,otMessageBytes.Length);

TextWriter/ TextReader abstract classes

StreamWriter extends TextWriter

using (StreamWriter writer = new StreamWriter(&quot;out.txt&quot;)

{writer.Write(&quot;…..&quot;)}

Using(StreamReader reader = new StreamReader(&quot;out.txt&quot;)

{string read = reader.ReadToEnd()}
</code></pre>

There is no Open() method for StreamWriter.

### DataContractSerializer

Serialises to XML

All members need to be marked with [DataMember]

Private members can be serialised

To serialise members in a different order use [DataMember Order=…]

If there is no Order value set they will be serialised first in alphabetical order

You can only serialize to XML as elements, not as attributes

## Handling Events

To raise an event add a protected virtual method named e.g. OnMyEvent(EventArgs args)

The event is associated with the EventHandler delegate and raised in the method.


<pre><code>
public class Key

{ public event EventHandler<MyKeyPressEventArgs> MyKeyPressed // can be EventHandler or EventHandler<T>

protected virtual void OnMyKeyPressed(MyKeyPressedEventArgs args)

{MyKeyPressed?.Invoke(this, args)}

public PressKey(char key) //public way of raising the event

{ OnMyKeyPressed(new MyKeyPressedEventArgs{Key=key}}

}

var key = new Key();

Key.MyKeyPressed += MyKeyPressedMethod

private void MyKeyPressedMethod(object sender, MyKeyPressedEventArgs e)
{//perform some action in response to the event}

</code></pre>

### Pattern Matching

Whereas you used to have to say:

<pre><code>
if (shape  is Square)

{var s = (Square) shape

return s.side \* s.side}

</code></pre>

You can now say:

<pre><code>
if (shape is Square s)
 return s.Side \* s.Side
</code></pre>

Switch is now not restricted to numbers and strings

<pre><code>
switch (shape)

{ case Square s when s.side==0;

case Circle c when c.radius ==0;

 return 0;

case square s:

 return s.side\* s.side;

case circle c:

 return c.radius \* c.radius \* Math.Pi
</code></pre>

###  Auto Properties

public string MyProp {get; set}

From c#6 on you can have immutable auto properties

public string MyProp {get;} //previously required private set

### Expression Backed Members

Instead of:

 public Timespan Age {return DateTime.Now-DateOfBirth}

Can have:

public Timespan Age => DateTime.Now – DateOfBirth;

You can only do this where you would have one line of code in the braces.

This is only for read only properties

### IComparable and IComparer

For custom objects you may have to provide IComparable or IComparer definitions if you need to sort collections of the object.

IComparable compares 2 objects of a particular type
<pre><code>
int IComparable.CompareTo(object obj)

{ Car c = (Car) obj;

Return string.Compare(this.make, c.make)}
</code></pre>
IComparer allows more complicated sorting mechanisms, e.g. on several fields

<pre><code>
private class sortYearAscendingHelper: IComparer

{ int IComparer.Compare(object a, object b)

 {car c1 = (car)a; car c2=(car) b;

 If(c1.year > c2.year) return 1;

 If (c1.year < c2.year) return -1 else return 0;

}

public static IComparer SortYearAscending()

{ return (IComprer) new SortYearAscendingHelper();}

//you then pass this as the scond parameter of Array.Sort() that accepts IComparer

Array.Sort(carArray, car.SortYearAscending())
</code></pre>