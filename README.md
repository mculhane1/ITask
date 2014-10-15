# ITask

Provides an awaitable covariant ITask interface which may be used in place of the built-in Task class.

## Purpose

The built-in `System.Threading.Tasks.Task` and `System.Threading.Tasks.Task<TResult>` classes allow for compiler support of the `async` and `await` keywords.  However, since these types must be used as the return values of the methods which leverage the support of the keywords, any interface containing one of these methods may not be covariant over the type `TResult`.  This can cause a big problems, especially when converting methods on existing interfaces from being synchronous to asynchronous.

This is where the `ITask` interface comes in.  Both `ITask` and `ITask<TResult>` interfaces are included for consistency, but the real power lies in teh `ITask<TResult>` interface.  It exposes the same functionality as `System.Threading.Tasks.Task<TResult>`, simply through an interface.  Because `TResult` is only used in the output position for this interface, it is covariant (its definition is `public interface ITask<out TResult>`) and may be used as a return value within another generic interface without breaking its covariance.

## Usage

### Using an ITask within an interface

Given the following interface:

```c#
interface ITest<out T>
{
    T CopmputeValue();
}
```

converting the `ComputeValue` method from being synchronous to ansynchronous would require removing the covariance of interface `ITest<T>` as follows:

```c#
interface ITest<T>
{
    System.Threading.Tasks.Task<T> CopmputeValue();
}
```

With the `ITask` interface, it is possible to make the `ComputeValue` method compatible with the `await` keyword (indicating that it is asynchronous) while maintaining covariance by changing the interface as follows:

```c#
interface ITest<out T>
{
    ITask<T> CopmputeValue();
}
```

### Awaiting an ITask

The await keyword may be used with an `ITask<TResult>` in the same manner that it is with a `System.Threading.Tasks.Task<TResult>`.

For example, within a method marked with the `async` keyword and given a variable `t` of type `ITest<int>`, the `await` keyword may be used as follows:

```c#
int result = await t.ComputeValue();
```

Note that the functionality required by the compiler to enable `await` keyword support is included in a class in the `MorseCode.ITask` namespace.  Therefore, a `using` statement will need to be added to the file for this code to compile as follows:

```c#
using MorseCode.ITask;
```

If you are using Resharper, it should suggest adding this `using` statement automatically.

### Returning an ITask

Unfortunately, the C# compiler only supports marking methods with the `async` keyword if they return a `System.Threading.Tasks.Task` or a `System.Threading.Tasks.Task<TResult>`.  However, the only reason to use the `async` keyword is to enable support for the `await` keyword within that method.  As long as the method itself returns an awaitable (which `ITask` and `ITask<TResult>` are), then it doesn't matter if that method is marked with the `async` keyword to callers of the method.

However, we do wish to maintain support for the `async` keyword for the code within a method returning either an `ITask` or an `ITask<TResult>`.

The easiest way to accomplish this is to use the `TaskInterfaceFactory` class's `Create` method.  This method expects as its only parameter a method returning either a `System.Threading.Tasks.Task` or a `System.Threading.Tasks.Task<TResult>` and produces either an `ITask` or an `ITask<TResult>`.  This method may be defined as a lambda and marked with the `async` keyword.  The result of the call to `TaskInterfaceFactory.Create` can simply be returned from the method returning the `ITask` or `ITask<TResult>` and should be the only statement within that method.

For example, the following is a sample implementation of the `ComputeValue` method defined above in a class implementing `ITest<int>`:

```c#
public ITask<int> ComputeValue()
{
    return TaskInterfaceFactory.Create(async () =>
        {
            // Do some computing here!
            // The await keyword may be used freely.
            
            //For example:
            int result1 = await DoSomeOtherComputing();
            int result2 = await DoSomeOtherComputingForSomethingElse();
            return result1 + result2;
        });
}
```

If you already have a `System.Threading.Tasks.Task` or a `System.Threading.Tasks.Task<TResult>` and you wish to convert it into an `ITask` or an `ITask<TResult>` simply use the `AsITask` extension method as follows (given a variable `t` of type `System.Threading.Tasks.Task` and a variable `t2` of type `System.Threading.Tasks.Task<int>`):

```c#
ITask iTask = t.AsITask();
ITask<int> iTask2 = t2.AsITask();
```
