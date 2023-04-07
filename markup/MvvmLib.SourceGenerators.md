# MvvmLib.SourceGenerators

> Allows to generate code with [Source Generators](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview) 


Attributes:

* INPC attribute on class to implement INotifyPropertyChanged
* SetProperty attribute on fields
* OnPropertyChanged attribute on fields with target Property Name or empty for self
* Command attribute on methods for add DelegateCommand with CanExecute for CanExecute method
* RaiseCanExecuteChanged attribute on fields with target command property name

Sample

```cs
using MvvmLib.SourceGenerators;

namespace SourceGeneratorsSample.ViewModels
{
    [INPC]
    public partial class ShellViewModel
    {
        [SetProperty]
        [OnPropertyChanged(PropertyName = nameof(CompleteTitle))]
        string _title;

        [SetProperty]
        [RaiseCanExecuteChanged(nameof(ChangeTitleCommand))]
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
using System;
using System.ComponentModel;
using System.Runtime.CompilerServices;
using MvvmLib.Commands;
namespace SourceGeneratorsSample.ViewModels
{
	partial class ShellViewModel : INotifyPropertyChanged
	{
		public string Title
		{
			get { return _title; }
			set
			{
				if(SetProperty(ref _title, value))
				{

					OnPropertyChanged("CompleteTitle");
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

        public event PropertyChangedEventHandler PropertyChanged;

        protected virtual bool SetProperty<T>(ref T storage, T value, [CallerMemberName] string propertyName = null)
        {
            if (Equals(storage, value))
                return false;

            storage = value;
            OnPropertyChanged(propertyName);
            return true;
        }

        protected virtual void OnPropertyChanged(string propertyName)
        {
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
        }
    }
}
```

<p>
<img src="https://res.cloudinary.com/du6bjt9gj/image/upload/v1680900700/sourcegenerators_lw3blh.png">
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

Custom Command configuration

Sample with Prism

```cs
private const string PrismCommandsNamespace = "Prism.Commands";

[Command(Namespace = PrismCommandsNamespace)]
private void ChangeTitle(string title)
{
    Title = title;
}
```
Note: "DelegateCommand" can be changed to "RelayCommand" for example with TypeName.