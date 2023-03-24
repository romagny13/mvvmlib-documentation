# Mvvm.Core

* Commands: **DelegateCommand**, **AsyncCommand** and **CompositeCommand**
* Task Extensions: for **Async** with **void method**
* Mvvm: **BindableBase**, **Validatable** and **ModelWrapper** base classes
* **EventAggregator** : allows to **subscribe**, **publish** and filter messages

## Commands

### DelegateCommand and DelegateCommand<T>

Example

```cs
private bool _canSave;
public bool CanSave
{
    get { return _canSave; }
    set
    {
        if (SetProperty(ref _canSave, value))
            SaveCommand.RaiseCanExecuteChanged();
    }
}

private DelegateCommand _saveCommand;
public DelegateCommand SaveCommand =>
    _saveCommand ?? (_saveCommand = new DelegateCommand(ExecuteSaveCommand, CanExecuteSaveCommand));

private void ExecuteSaveCommand()
{
    // do something
}
```

### AsyncCommand

Example

```cs
public class ShellViewModel : BindableBase
{
    private bool isBusy;
    public bool IsBusy
    {
        get { return isBusy; }
        set { SetProperty(ref isBusy, value); }
    }

    private IAsyncCommand myCommand;
    public IAsyncCommand MyCommand
    {
        get
        {
            if (myCommand == null)
                myCommand = new AsyncCommand(ExecuteMyCommand);
            return myCommand;
        }
    }

    private async Task ExecuteMyCommand()
    {
        IsBusy = true;

        await Task.Delay(3000); // do some work

        IsBusy = false;
    }
}
```

Supports cancellation:

```cs
myCommand.Cancel(); 
// other sample 
myCommand.CancellationTokenSource.CancelAfter(500);
```

Checks cancellation

```cs
if (myCommand.IsCancellationRequested)
    return;
```

Handle exception

```cs
myCommand = new AsyncCommand(ExecuteMyCommand, ex => { });
```

### CompositeCommand

Example

```cs
public interface IApplicationCommands
{
    CompositeCommand SaveAllCommand { get; }
}
public class ApplicationCommands : IApplicationCommands
{
    public CompositeCommand SaveAllCommand { get; } = new CompositeCommand();
}
```

Inject the application commands in ShellViewModel (and bind the SaveAllCommand in the View)


```cs
public class ShellViewModel
{
    public ShellViewModel(IApplicationCommands applicationCommands)
    {
        ApplicationCommands = applicationCommands;
    }

    public IApplicationCommands ApplicationCommands { get; }
}
```

... Inject the application commands in other ViewModels and add commands to the composite command

```cs
public class TabViewModel : BindableBase
{
    private bool _canSave;
    private string _updatedText;
    private DelegateCommand _saveCommand;
    private readonly IApplicationCommands _applicationCommands;

    public TabViewModel(IApplicationCommands applicationCommands)
    {
        _applicationCommands = applicationCommands;
        _applicationCommands.SaveAllCommand.Register(SaveCommand);
    }

    public string UpdatedText
    {
        get { return _updatedText; }
        set { SetProperty(ref _updatedText, value); }
    }

    public bool CanSave
    {
        get { return _canSave; }
        set
        {
            if (SetProperty(ref _canSave, value))
                SaveCommand.RaiseCanExecuteChanged();
        }
    }

    public DelegateCommand SaveCommand =>
        _saveCommand ?? (_saveCommand = new DelegateCommand(ExecuteSaveCommand, CanExecuteSaveCommand));

    private void ExecuteSaveCommand() =>  UpdatedText = $"Save {DateTime.Now}";
}
```

The composite command can execute (all commands) only when each registered command can be executed.

## Task Extensions

```cs
// call a task from void method
public void VoidMethod()
{
    DelayedTask().Await(completedCallback: () =>
    {
      
    });
}

public async Task DelayedTask()
{
    await Task.Delay(1000);
}
```
It's possible to intercept exception with `errorCallback` and configure await to true (for the UIThread) with `continueOnCapturedContext` (false by default)

## Mvvm

### BindableBase

> Implements _INotifyPropertyChanged_ interface.

Allows to observe a property and notify the view that a value has changed.

SetProperty

```cs
public class UserViewModel : BindableBase
{
    private string firstName;
    public string FirstName
    {
        get { return firstName; }
        set { SetProperty(ref firstName, value); }
    }
}
```

OnPropertyChanged

```cs
public class UserViewModel : BindableBase
{
    private string firstName;
    public string FirstName
    {
        get { return firstName; }
        set
        {
            SetProperty(ref firstName, value);
            OnPropertyChanged(nameof(FullName));
        }
    }

    private string lastName;
    public string LastName
    {
        get { return lastName; }
        set
        {
            SetProperty(ref lastName, value);
            OnPropertyChanged(nameof(FullName));
        }
    }

    public string FullName
    {
        get { return $"{firstName} {LastName}"; }
    }
}
```

Or with **Linq** Expression

```cs
public class User : BindableBase
{

    private string firstName;
    public string FirstName
    {
        get { return firstName; }
        set
        {
            firstName = value;
            OnPropertyChanged(() => FirstName);
        }
    }
}
```

### Validatable

> Validation + Edition

Allows to **validate** the model with **Data Annotations** and **custom validations**.  

| Validation Type | Description |
| --- | --- |
|  OnPropertyChange | Default. Validation on property changed |
|  OnSubmit | Validation only after "ValidateAll" invoked. Validation on property changed only after "ValidateAll" invoked |
|  Explicit | Validation only with "ValidateProperty" and "ValidateAll" |


| Property | Description |
| --- | --- |
|  UseDataAnnotations | Ignore or use Data Annotations for validation  |
|  UseCustomValidations | Ignore or use Custom validations |


| Method | Description |
| --- | --- |
|  ValidateProperty | Validate one property  |
|  ValidateAll | Validate all properties |
|  Reset | Reset the errors and is submitted |

The model requires to use SetProperty

```cs
public class User : Validatable
{
    private string firstName;

    [Required]
    [StringLength(50)]
    public string FirstName
    {
        get { return firstName; }
        set { SetProperty(ref firstName, value); }
    }

    // object, list , etc.
}
```

```cs
var user = new User
{
    FirstName = "Marie",
    LastName = "Bellin",
    // etc.
};


// validate a property
user.ValidateProperty("FirstName");

// validate all
user.ValidateAll();

// summary
var allErrors = user.GeErrorSummary();

if (user.HasErrors)
{ }

// reset the errors and is submitted
user.Reset();
```

### ModelWrapper

> Allows to wrap, edit and validate a model.

The model does not require to use SetProperty

```cs
public class User
{
    public int Id { get; set; }

    [Required]
    [StringLength(50)]
    public string FirstName { get; set; }

    [StringLength(50)]
    public string LastName { get; set; }

    // object, list , etc.
}
```

Create a Generic model wrapper class

```cs
public class UserWrapper : ModelWrapper<User>
{
    public UserWrapper(User model) : base(model)
    {
    }

    public int Id { get { return Model.Id; } }

    public string FirstName
    {
        get { return GetValue<string>(); }
        set { SetValue(value); }
    }

    // etc.

    // custom validations
    protected override IEnumerable<string> ValidateCustom(string propertyName)
    {
        switch (propertyName)
        {
            case nameof(FirstName):
                if (string.Equals(FirstName, "Marie", StringComparison.OrdinalIgnoreCase))
                {
                    yield return "Marie is not allowed";
                }
                break;
        }
    }
}
```

**Wpf**

Binding

```xml
<!-- the default value of UpdateSourceTrigger is LostFocus -->
<TextBox Text="{Binding User.FirstName, UpdateSourceTrigger=PropertyChanged}" />
```

Create a Style that displays errors

```xml
<Style TargetType="TextBox">
    <Setter Property="Validation.ErrorTemplate">
        <Setter.Value>
            <ControlTemplate>
                <StackPanel>
                    <AdornedElementPlaceholder x:Name="placeholder"/>
                    <!--TextBlock with error -->
                    <TextBlock FontSize="12" Foreground="Red" 
                               Text="{Binding ElementName=placeholder,Path=AdornedElement.(Validation.Errors)[0].ErrorContent}"/>
                </StackPanel>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
    <Style.Triggers>
        <Trigger Property="Validation.HasError" Value="True">
            <Setter Property="Background" Value="Red"/>
            <!--Tooltip with error -->
            <Setter Property="ToolTip" 
                    Value="{Binding RelativeSource={RelativeSource Self}, Path=(Validation.Errors)[0].ErrorContent}"/>
        </Trigger>
    </Style.Triggers>
</Style>
```

**Uwp**

```xml
<TextBox Text="{Binding User.FirstName, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}" />
<TextBlock Text="{Binding User.Errors[FirstName][0]}" Foreground="Red"></TextBlock>
```

Or Create a _Converter_ that displays the _first error_ of the list

```cs
public sealed class FirstErrorConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, string language)
    {
        var errors = value as IList<string>;
        return errors != null && errors.Count > 0 ? errors.ElementAt(0) : null;
    }

    public object ConvertBack(object value, Type targetType, object parameter, string language)
    {
        throw new NotImplementedException();
    }
}
```

And use it
```xml
<Page.Resources>
    <converters:FirstErrorConverter x:Name="FirstErrorConverter"></converters:FirstErrorConverter>
</Page.Resources>
```

```xml
 <TextBox Text="{Binding User.FirstName, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}" />
<!-- with converter -->
<TextBlock Text="{Binding User.Errors[FirstName], Converter={StaticResource FirstErrorConverter}}" Foreground="Red"></TextBlock>
```


## EventAggregator

> allows to subscribe, publish and filter messages


Register the eventAggregator as singleton with an ioc container. 

```cs
public class ShellViewModel
{
    public ShellViewModel(IEventAggregator eventAggregator)
    { }
}
```

**Subscribe** and **publish** with empty event (no parameter)

Create the event class
```cs
public class MyEvent : EventBase
{ }
```

```cs
// subscriber
eventAggregator.GetEvent<MyEvent>().Subscribe(() => { /* do something with args.Message */ });
```

```cs
// publisher
eventAggregator.GetEvent<MyEvent>().Publish();
```

**Subscribe** and **publish** with `parameter`

```cs
public class DataSavedEvent : EventBase<DataSavedEventArgs>
{ }

public class DataSavedEventArgs
{
    public string Message { get; set; }
}
```

```cs
// subscriber
eventAggregator.GetEvent<DataSavedEvent>().Subscribe(args => { /* do something with args.Message */ });
```

```cs
// publisher
eventAggregator.GetEvent<DataSavedEvent>().Publish(new DataSavedEventArgs { Message = "Data saved." })
```

**Filter**

Example: Filter on "user id"

```cs
eventAggregator.GetEvent<MyUserEvent>().Subscribe(user => { /* do something */ }).WithFilter(user => user.Id == 1); // <= not notified

messenger.GetEvent<MyUserEvent>().Publish(new User { Id = 2, UserName = "Marie" });  // <=
```
The event class:

```cs
public class MyUserEvent : EventBase<User>
{ }
```

**Execution strategy**:

* **PublisherThread** (default)
* **UIThread**
* **BackgroundThread**

```cs
eventAggregator.GetEvent<DataSavedEvent>().Subscribe(_ => { }).WithExecutionStrategy(ExecutionStrategyType.UIThread);
```

**Unsubscribe** with the token received on subscription.

```cs
var subscriberOptions = eventAggregator.GetEvent<DataSavedEvent>().Subscribe(_ => { });

var success = eventAggregator.GetEvent<DataSavedEvent>().Unsubscribe(subscriberOptions.Token);
// or
subscriberOptions.Unsubscribe();
```