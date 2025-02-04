# Rx.Net.Plus

## Table of content

- [Introduction](#introduction)
- [RxVar](#rxvar)
    - [Basics](#basics)
    - [Comparison](#comparison)
    - [Boolean Operators](#boolean-operators)
    - [Distinct mode](#distinct-mode)
    - [IReadOnlyRxVar](#IReadOnlyRxVar)
    - [Disposing](#disposing)
    - [Serialization](#serialization)
    - [Json Flat Serialization](#json-flat-serialization)
    - [Limitations](#limitations)
- [RxProperty for WPF](#rxproperty-for-WPF)
    - [Purpose](#purpose)
    - [How to do](#How-to-do)
    - [Example](#example)
- [Extension Methods](#extension-methods)
- [Nuget](#nuget)
- [License](#license)
- [Contact](#contact-us)

### Introduction

ReactiveX gains popularity and is widely used in several platforms and languages.

Following is an excerpt of Rx.Net *Brief Intro* :

"*The Reactive Extensions (Rx) is a library for composing asynchronous and event-based programs using observable sequences and LINQ-style query operators. Using Rx, develope*rs ***represent*** *asynchronous data streams with [Observables](https://docs.microsoft.com/dotnet/api/system.iobservable-1),* ***query*** *asynchronous data streams using [LINQ operators](http://msdn.microsoft.com/en-us/library/hh242983.aspx), and* ***parameterize*** *the concurrency in the asynchronous data streams using* [*Schedulers*](http://msdn.microsoft.com/en-us/library/hh242963.aspx). 

*Simply put, **Rx = Observables + LINQ + Schedulers**."*

If you have **used** Rx.Net (System.Reactive), you probably already wished to turn regular variable (like *int*, *bool* ) to become reactive.

What do we mean? We would like to:

1. Be notified asynchronously whenever the value is changed (**push** model vs **pull** model)
2. Connect variables to observable sources (be an **observer**) and utilize **LINQ** semantics
3. Avoid repetitive notification of same value (as it doesn't matter with regular variables)
4. Be able to synchronously get **value of** **reactive** **variables** using regular c# variable semantics. **Compare** them to other variables\constants, **convert** them to another type...

Hence, we could state that **Rx.Net.Plus** will lead to the following formula:

​          **Rx.Net.Plus** =  **Rx.Net** + **Observers** + classic (**pull model**) c# semantics.

**Rx.Net.Plus** is a small but smart library which introduces **two** new types for this purpose (+ several extension methods):

| Type             | Purpose                                                      |
| :--------------- | ------------------------------------------------------------ |
| ***RxVar***      | supersede basic types (as int, bool...)                      |
| ***RxProperty*** | Support of MVVM to replace view-model *properties* (*NotifyPropertyChanged* pattern) |

### RxVar

#### Basics

Let's start with an example:

Suppose you have in your code, some status of a connection named *isConnected*.

It may be defined as following:

```c#
bool IsConnected { get; set; }
```

You may want to react to change of connection status. One way is to **poll** this status periodically.

ReactiveX introduces the **push** model extending the *Observer* pattern.

One have to define an observable and update its state.

Other have to listen to events related to this observable as following:

```c#
IObservable<bool> IsConnected { get; set; }
....
IsConnected.Subscribe (_isConnected => _isConnected ? doSomething() : doAnotherThing());
```

Some questions arrive:

1. How do we create this observable in a simple and straightforward way?
2. How do we update the state of the observable?
3. How can we use it as an ordinary variable - by polling its values - `if (IsConnected) doSomething()` ?
4. How can we chain information from another source to update the IsConnected status?

The answer is: **Rx.Net.Plus** !!

With **Rx.Net.Plus**, it is super simple to define this kind of variable.

```c#
 var isConnected = false.ToRx(); // Create a reactive variable
```

We can easily update its state.

```c#
isConnected.Value = true;
```

Suppose we have a <u>sequential</u> algorithm, no problem to write this kind of code:

```c#
if (isConnected)		// Note we don't use .Value of isConnected
{
	StartWork();
}
else
{
	StopWork();
}
```

Let's say, *connectionStatus* is of type Observable

```c#
IObservable<bool> connectionStatus;
```

we can write:

```c#
connectionStatus.Subscribe (isConnected);
```

or using **Rx.Net.Plus** semantics:

```c#
	isConnected.ListenTo (connectionStatus);
or
	connectionStatus.RedirectTo (isConnected);
or
	connectionStatus.Notify (isConnected);
```

**Rx.Net.Plus** provides full comparison and equality support:

```c#
var anotherRxBool = true.ToRx();

if (isConnected == anotherRxBool)
{
	// Do some action
}
```

With primitive types  (*int*, *double*...)

```c#
var intRx = 10.ToRx();               // Create a rxvar

if (intRx > 5)
{
	// React !!
}
```

RxVar could be used as part of **state machines**, and instead of polling statuses, flags, counters...

Just react upon change in this manner:

```c#
isConnected.When(true).Notify (v => reactToEvent());
```

It could be used for storing application information:

```c#
class SystemInfo
{
    RxVar<DateTime> CurrentTime;
    RxVar<bool>		AreHeadphonesConnected;
}
...

    systemInfo.CurrentTime.Notify (dt => DisplayToScreen (dt));	// Reactive !
....
   
   DateTime deadlineDate;
   if (systemInfo.CurrentTime > deadlineDate)					// Classic polling !
   {
       SendNotification();
   }
```



Note that ***RxVar*** behaves exactly as a classic variable. Therefore, as for classic variable, **in case it is not initialized** the value will be the default, ***RxVar*** will also publish the default value **on subscription**.

One option is to use the `IgnoreNull`* extension to avoid receiving the null value, or use `Skip(1)` to skip first value.

RxVar implements the following interfaces:

```c#
ISubject, IConvertible, IDisposable, IComparable, IEquatable, ISerializable
```

#### Comparison

By default, **RxVar** uses for its comparisons the default comparer of the generic type used

```C#
Comparer<T>.Default
```

In case a specific comparer is needed, RxVar provides the **SetComparer** method to specify it.

#### Boolean Operators

**RxVar** has the following 'pseudo' boolean operators: **True, False, Not** for comparison

```C#
if (isConnected.Not) equivalent if (! isConnected)
```

#### Distinct mode

By default, **RxVar** propagates its data (on new value assignment) only when a new **distinct** value is set.

That means that if some observer is subscribed to RxVar and the same value is assigned to RxVar, it will not be published.

In order to allow every update to be propagated, RxVar provide the following property:

```C#
IsDistinctMode
```

**Note**: *distinct mode* may be changed at any time even after subscription of observers.

#### IReadOnlyRxVar

**RxVar** can be published as read-only, similarly to `IReadOnlyList` concept for arrays.

This allows using of RxVar to be used without caring of being modified outside.

```C#
var rxVar = 10.ToRxVar();
var roRxVar = (IReadOnlyRxVar<int>) rxVar;
roRxVar.Value = 30;		// Won't compile !!!
rxVar.Value = 20;		// Compile
var val = roRxVar;		// val is equal to 20

```

#### Disposing

**Rx.Net.Plus** provides a base class which implements *IDisposable*, called ***DisposableBaseClass***.

*DisposableBaseClass* has a built-in *CancellationToken* used by RxVar and RxProperty to automatically unsubscribe (in case they are used as observers) on disposing.

This is very convenient as there no need to handle IDisposable...when subscribing RxVar to observables.

The following methods utilize the automatic unsubscription



- ListenTo(IObservable<T> observable)
- RedirectTo (IObserver<T> observer)
- Notify(IObserver<T> observer)
- Notify (Action<T> onNext)

```c#
// Classic style
IDisposable subscription = observable.Subscribe (rxVar);
subscription.Dispose();

// Thanks to DisposabeBaseClass (used by RxVar)
rxVar.ListenTo (observable);
....
// RxVar is automatically unsubscribed from observable when disposed
```

#### Serialization

**RxVar** supports standard serialization (have been tested for binary, xml, Json formats).

The following fields are serialized: ***Value, IsDistinct.***

#### Json Flat Serialization

What is Json *"flat"*  serialization? 

let's give an example. The following is part of the Test module in the solution.

```c#
    public class Info
    {
        public string Title { get; set; } = "Mr";
        public double Height { get; set; } = 76.5;
    }


    public class User
    {
        public RxVar<string> Name = "John".ToRxVar();
        public RxVar<bool> IsMale = true.ToRxVar();
        public RxVar<int> Age = 16.ToRxVar();
        public RxVar<Info> InfoUser = new Info().ToRxVar();

        public User()
        {
            Name.IsDistinctMode = false;
        }
    }
```



Normal Json serialization will output:

```json
{
  "Name": {
    "IsDistinctMode": false,
    "Value": "John"
  },
  "IsMale": {
    "IsDistinctMode": true,
    "Value": true
  },
  "Age": {
    "IsDistinctMode": true,
    "Value": 16
  },
  "InfoUser": {
    "IsDistinctMode": true,
    "Value": {
      "Title": "Mr",
      "Height": 76.5
    }
  }
```

With flat serialization:

```json
{
  "Name": "John",
  "IsMale": true,
  "Age": 16,
  "InfoUser": {
    "Title": "Mr",
    "Height": 76.5
  }
}
```

Json flat serialization is available through a dedicated [Nuget](https://www.nuget.org/packages/Rx.Net.Plus.Json/) package.

In order to apply flat serialization, follow these steps:

1. Add a reference to *Rx.Net.Plus.Json* package.

2. Call once the following method to apply this serialization style:

  ```C#
  RxVarFlatJsonExtensions.RegisterToJsonGlobalSettings();
  ```

#### Limitations

As C# variables are reference, assigning <u>directly</u> to RxVar is impossible (because it would replace the referenced RxVar itself with another one!)

Therefore the following notation is disallowed:

```c#
var number = 20.3.ToRx();
number = 10; // Not allowed by RxVar
```

Instead, *RxVar* provides the following semantics:

```c#
number.Value = 10;
number.Set (10);
```

### RxProperty for WPF

#### Purpose

**Rx.Net.Plus** also provides means to leverage RxVar to WPF.

The name of class to use for properties is: **RxProperty** and it is directly derived from RxVar.

#### How to do

##### Step 1: Implement *IPropertyChangedProxy* in View Model

For *data binding* **of RxProperty** to work as expected,  the view model shall implement a dedicated interface named *IPropertyChangedProxy*.

Assuming the View Model implements *INotifyPropertyChanged*, it shall **also** implements *IPropertyChangedProxy*as following:

```c#
void IPropertyChangedProxy.NotifyPropertyChanged(PropertyChangedEventArgs eventArgs)
{
	OnPropertyChanged (eventArgs);
}
```

Binding shall occur **after** the view has been created and bound to view model.

In *Caliburn* framework for instance, this could occur inside the OnViewAttached method as following:

```c#
class ViewModel : Screen, IPropertyChangedProxy
{
    protected override void OnViewAttached(object view, object context)
    {
        base.OnViewAttached(view, context);
        this.BindRxPropertiesToView (view);			// Bind RxProperties to view
    }
}
```

In classic code-behind MVVM or other frameworks (like Prism, MVVM Light...), call to BindRxPropertiesToView can be handled in Loaded event handler.

##### Step 2: Define properties in View model

```c#
public class ViewModel :  IPropertyChangedProxy
{
	public RxProperty<bool>		IsFunctionEnabled { get; } = false.ToRxProperty();
	public RxProperty<int> 		Counter { get; }
	public RxProperty<string>	Message { get; }
}
```

##### Step 3: Binding in XAML

```XML
<CheckBox  IsChecked="{Binding IsFunctionEnabled}"/>
<TextBox   Text="{Binding Counter}"/>
<TextBox   Text="{Binding Message}"/>
```

Notes:

1. **Rx.Net.Plus** supports ***TwoWay*** binding mode (where target can update source as for CheckBox).
2. Binding is specified directly to property name (and not to its Value - although it is allowed).

#### Example

The following code is a full implementation example. It relies on Caliburn but as stated above it is easy to implement in other frameworks or code-behind MVVM.

```c#
public class ViewModel :  IPropertyChangedProxy, Screen
{
	public RxProperty<bool>		IsFunctionEnabled { get; } = false.ToRxProperty();
	public RxProperty<int> 		Counter { get; }
	public RxProperty<string>	Message { get; }

	public ViewModel()
    {
    	IObservable<int> counter = IOC.GetSessionCounter();
    	Counter = counter.ToRxPropertyAndSubscribe();
    	
    	IObservable<string> errorMessage = IOC.GetErrorMessage();
    	Message = errorMessage.Select (msg => $"Session#{Counter} an error occured: {msg}").ToRxPropertyAndSubscribe();
    }
	
	void IPropertyChangedProxy.NotifyPropertyChanged(PropertyChangedEventArgs eventArgs)
	{
		OnPropertyChanged (eventArgs);
	}
	
	protected override void OnViewAttached(object view, object context)
    {
        base.OnViewAttached(view, context);
        this.BindRxPropertiesToView (view);			// Bind RxProperties to view
    }
}
```

### Extension Methods

#### Subscription

| Method           | Description             | Usage                      |
| ---------------- | ----------------------- | :------------------------- |
| ***RedirectTo*** | an alias of *Subscribe* | `rxVar.RedirectTo(rxVar2)` |
| ***Notify***     | an alias of *Subscribe* | `rxVar.Notify(rxVar2)`     |



#### Conditional data propagation

| Method                                                       | Description                                                  | Usage                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ | :----------------------------------------- |
| ***IgnoreNull***                                             | equivalent to*Where* (v => v != null)                        | `rxVar.IgnoreNull().Notify (rxVar2)`       |
| ***When***(value)                                            | equivalent to*Where* (v => v.Equals(value))                  | `rxVar.When(true).Notify (rxVar2)`         |
| ***If*** (value)                                             | equivalent to*Where* (v => v.Equals(value))                  | `rxVar.If (10).Notify (rxVar2)`            |
| **IfNot** (value)                                            | equivalent to*Where* (v => false == v.Equals(value))         | `rxVar.IfNot (5).Notify (rxVar2)`          |
| **IfEqualTo, IfNotEqualTo, IfLessThan, IfLessThanOrEqualTo, IfGreaterThan, IfGreaterThanOrEqualTo** | Comparison 'operators': <, <=, >, >=                         | `rxVar.IfGreaterThan (5).Notify (rxVar2)`  |
| **IfInRange, IfInStrictRange, IfOutOfRange, IfOutOfStrictRange** | IfInRange(a,b): a <= value <= b                                    IfInStrictRange(a,b): a < value < b                            IfOutOfRange(a,b):  value <= a OR value >= b                IfOutOfStrictRange(a,b): value < a OR  b < value | `rxVar.IfInRange(5,7).Notify (rxVar2)`     |
| **Clip**                                                     | Clip(min,max) force value to be in range {min,max}           | `sensorReading.Clip(0,200).Notify(screen)` |

#### Instantiation and subscription

| Method                         | Description                                                  | Usage                                                        |
| ------------------------------ | ------------------------------------------------------------ | :----------------------------------------------------------- |
| ***ToRxVar***                  | creates an *instance* of **RxVar** of the type of **var**    | `var rxVar = 10.ToRxVar()`                                   |
| ***ToRxVarAndSubscribe***      | creates an instance of RxVar of the same type of **observable source** and subscribe the new instance to **source** | `IObservable<int> obs; var rxvar = obs.Where(v => v > 10).ToRxVarAndSubscribe();` |
| ***ToRxProperty***             | creates an *instance* of **RxProperty** of the type of **var** | `RxProperty<int> Counter => 10.ToRxProperty();`              |
| ***ToRxPropertyAndSubscribe*** | creates an instance of RxProperty of the same type of **observable source** and subscribe the new instance to **source** | `rxvar.ToRxPropertyAndSubscribe();`                          |

#### MVVM

| Method                       | Description                                                  | Usage                                |
| ---------------------------- | ------------------------------------------------------------ | :----------------------------------- |
| ***BindRxPropertiesToView*** | Bind the RxProperties of the view model *vm* (which implements *IPropertyChangedProxy*) to the view.an alias of *Subscribe* | `this.BindRxPropertiesToView (view)` |

### Nuget

Rx.Net.Plus packages are available at [Nuget](https://www.nuget.org/packages?q=rx.net.plus)

### License

Refer to [license](https://github.com/ReactiveSoftware/Rx.Net.Plus/blob/master/License.md)

### Contact us

For any comments, suggestions, bugs...please contact us at [reactivesw@outlook.com](mailto:reactivesw@outlook.com)

![](Logo.jpg)