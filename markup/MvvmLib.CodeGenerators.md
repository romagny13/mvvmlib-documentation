# MvvmLib.CodeGenerators

> Allows to generate code with [Source Generators](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview) 

Support: MvvmLib, Prism (installation detected)

Class Level Attributes:

* Inpc attribute: to implement INotifyPropertyChanged
* BindableObject attribute: allows to add BindableBase Class (and implement IActiveAware for Prism)

Field Level Attribute:

* BindableProperty attribute: allows to use "Setproperty", specify properties (OnPropertyChanged/RaisePropertyChanged) and commands (RaiseCanExecuteChanged) notified when the value has changed.

Method Level Attribute:

* Command attribute: for add a DelegateCommand with CanExecute method and Name to specify a custom command name.

Comments supported on fields and methods for generated properties and commands.

Sample with MvvmLib

```cs
using MvvmLib.CodeGenerators;

namespace CodeGeneratorsSample.ViewModels
{
    [BindableObject]
    public partial class ShellViewModel
    {
        [BindableProperty(nameof(CompleteTitle))]
        string _title;

        [BindableProperty(CommandsToNotify = new[] { nameof(ChangeTitleCommand) })]
        bool _canChange;

        public ShellViewModel()
        {
            _title = "Initial title";
        }

        public string CompleteTitle
        {
            get { return _title + "!!!!"; }
        }

        [Command(CanExecute = nameof(CanChangeTitle))]
        private void ChangeTitle(string title)
        {
            Title = title;
        }

        private bool CanChangeTitle(string title) => _canChange;
    }
}
```

Prism version


```cs
using MvvmLib.CodeGenerators;

namespace CodeGeneratorsSample.ViewModels
{
    [BindableObject(ImplementIActiveAware = true)]
    public partial class ShellViewModel
    {
        [BindableProperty(nameof(CompleteTitle))]
        string _title;

        [BindableProperty(CommandsToNotify = new[] { nameof(ChangeTitleCommand) })]
        bool _canChange;

        public ShellViewModel()
        {
            _title = "Initial title";
        }

        public string CompleteTitle
        {
            get { return _title + "!!!!"; }
        }

        [Command(CanExecute = nameof(CanChangeTitle))]
        private void ChangeTitle(string title)
        {
            Title = title;
        }

        private bool CanChangeTitle(string title) => _canChange;
    }
}
```

Generated :

```cs
// generated code
using System;
using Prism;
using Prism.Mvvm;
using Prism.Commands;
namespace CodeGeneratorsSample.ViewModels
{
	partial class ShellViewModel : BindableBase, IActiveAware
	{
		public string Title
		{
			get { return _title; }
			set
			{
				if(SetProperty(ref _title, value))
				{
					RaisePropertyChanged("CompleteTitle");
				}
			}
		}

		public bool CanChange
		{
			get { return _canChange; }
			set
			{
				if(SetProperty(ref _canChange, value))
				{
					ChangeTitleCommand.RaiseCanExecuteChanged();
				}
			}
		}

		private DelegateCommand<string> _changeTitleCommand;
		public DelegateCommand<string> ChangeTitleCommand =>
			_changeTitleCommand ?? (_changeTitleCommand = new DelegateCommand<string>(ChangeTitle, CanChangeTitle));

        private bool _isActive;
        /// <summary>
        /// Gets or sets a value indicating whether the object is active.
        /// </summary>
        public bool IsActive
        {
            get { return _isActive; }
            set
            {
                 _isActive = value;
                 OnIsActiveChanged();
            }
        }

        /// <summary>
        /// Notifies that the value for <see cref="IsActive"/> property has changed.
        /// </summary>
        public event EventHandler IsActiveChanged;

        /// <summary>
        /// Notifies that the value for <see cref="IsActive"/> property has changed.
        /// </summary>
        protected virtual void OnIsActiveChanged() => IsActiveChanged?.Invoke(this, new EventArgs());

	}
}
```

<p>
<img src="https://res.cloudinary.com/du6bjt9gj/image/upload/v1681157130/codegenerators_u82b8y.png">
</P>

Xaml 

```xml
<Window.DataContext>
    <viewModels:ShellViewModel />
</Window.DataContext>
<Grid>
    <StackPanel HorizontalAlignment="Center" VerticalAlignment="Center">
        <TextBlock Text="{Binding Title}" FontSize="40" />
        <TextBlock Text="{Binding CompleteTitle}" FontSize="40" />
        <Button Command="{Binding ChangeTitleCommand}" CommandParameter="New title!">Change title</Button>
        <CheckBox IsChecked="{Binding CanChange}" Content="Can change title?"/>
    </StackPanel>
</Grid>
```