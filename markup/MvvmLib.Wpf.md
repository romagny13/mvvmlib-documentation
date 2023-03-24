# MvvmLib.Wpf

* Mvvm: **ViewModelLocator**, **BindingProxy**, etc.
* Navigation: **NavigationService**, **ConfigurableNavigationService**
* Data: **ListCollectionViewEx**, **PagedList**, **PagedSource** and commands **ListCollectionViewCommands**, **PagedSourceCommands**
* Interactivity: **CallMethodAction**, **SelectorSelectedItemsSyncBehavior**, **EventToCommandBehavior**, etc.
* Controls: **AnimatingContentControl**, **TransitioningContentControl**, **TransitioningItemsControl**: allow to animate content. **NavigatableContentControl**
* Expressions: allows to create filters with Linq expressions.
* Common: **MvvmUtils**


## ViewModelLocator

> Allows to resolve ViewModels for Views with **AutoWireViewModel**. 

Default **convention**:

* Views in `Views` namespace
* View models in `ViewModels` namespace
* View model name: 
    * _view name + "ViewModel"_ (example: ShellViewModel for Shell)
    * Or if the view name ends by "View": _view name + "Model"_ (example: NavigationViewModel for NavigationView)

### Change the convention

Example with "View" and "ViewModel" namespaces

```cs
ViewModelLocationProvider.ChangeConvention((viewType) =>
{
  var viewFullName = viewType.FullName;
  viewFullName = viewFullName.Replace(".View.", ".ViewModel."); // <= 
  var suffix = viewFullName.EndsWith("View") ? "Model" : "ViewModel";
  var viewModelFullName = string.Format(CultureInfo.InvariantCulture, "{0}{1}", viewFullName, suffix);
  var viewModelType = viewType.Assembly.GetType(viewModelFullName);

  return viewModelType;
});
```

### Register a custom View Model for a view

```cs
ViewModelLocationProvider.Register<ShellViewModel, CustomViewModel>();
```

Or with factory


```cs
ViewModelLocationProvider.Register<ShellViewModel>(() => new CustomViewModel());
```


### AutoWireViewModel Attached property (Window, UserControl)

Example:


```xml
<Window x:Class="Sample.Views.Shell"
        ...
         xmlns:ml="http://mvvmlib.com/"
         ml:ViewModelLocator.AutoWireViewModel="True">
```


## Navigation

### NavigationService

Easy to use / customize, Mvvm support, injectable, etc.

| Method | Description |
| --- | --- |
| Navigate | Navigation by Uri or uri string to a registered view (or viewmodel)
| Replace| Allows to replace previous navigation entry
| MoveTo | Move to index or content
| MoveToFirst | Move to first page
| MoveToLast | Move to last page
| MoveToPrevious | Move to previous page
| MoveToNext | Move to next page
| Clear | clear history and content
| Sync | Allows to sync with another navigation service

| Property | Description |
| --- | --- |
| CanMoveToPrevious | Checks if can move to previous entry
| CanMoveToNext | Checks if can move to next entry
| Journal | The navigation journal

| Event | Description |
| --- | --- |
| CanMoveToPreviousChanged | Raised when the value changed
| CanMoveToNextChanged | Raised when the value changed
| ContentChanged | Raised on content changed
| Navigated | Raised on navigation end
| Navigating | Raised at beginning of navigation
| NavigationFailed | Raised on navigation failed (cancelled with activation guard for example)

Note: 1 navigation service per UIElement (or use NavigationBehavior for custom behaviors)

Bindable navigation service. Exemple with ContentControl

```xml
 <ContentControl Content="{Binding NavigationService.Content}"></ContentControl>
```

Register the navigation service with a IoC Container and inject in ViewModel

```cs
public class ShellViewModel
{
    public ShellViewModel(INavigationService navigationService)
    {
        NavigationService = navigationService;
        NavigationCommands = new NavigationServiceCommands(NavigationService);
        NavigationService.Navigate("HomeView");
    }

    public INavigationService NavigationService { get; }
    public NavigationServiceCommands NavigationCommands { get; }
}
```
Bind commands in view

```xml
<Button Command="{Binding NavigationCommands.MoveToFirstCommand}" Width="50">&lt;&lt;</Button>
<Button Command="{Binding NavigationCommands.MoveToPreviousCommand}" Width="50" Content="&lt;"></Button>
<Button Command="{Binding NavigationCommands.MoveToNextCommand}" Width="50" Content="&gt;"></Button>
<Button Command="{Binding NavigationCommands.MoveToLastCommand}" Width="50">&gt;&gt;</Button>
<Button Command="{Binding NavigationCommands.NavigateCommand}" CommandParameter="HomeView">Home</Button>
<!-- with parameter -->
<Button Command="{Binding NavigationCommands.NavigateCommand}" CommandParameter="ViewA?id=sample-id">View A (Guards)</Button>>
```

In code

```cs
NavigationService.Navigate("ViewA");
// With parameters 
NavigationService.Navigate("ViewA", new NavigateParameters
{
    { "id", 10 }
});
// note: the query parameters are added to the NavigateParameters
NavigationService.Navigate("ViewA?tag=sample", new NavigateParameters
{
    { "id", 10 }
});
// ViewModel
NavigationService.Navigate("ViewAViewModel");
```

Replace: example remove LoginView from navigation journal

```cs
public class LoginViewModel : ISupportNavigation
{
    private readonly IAuth _auth;
    private readonly INavigationService _navigationService;
    private DelegateCommand _loginCommand;
    private string _returnUrl;
    private INavigateParameters _parameters;

    public LoginViewModel(IAuth auth, INavigationService navigationService)
    {
        _auth = auth;
        _navigationService = navigationService;
    }

    public DelegateCommand LoginCommand
    {
        get
        {
            if (_loginCommand == null)
                _loginCommand = new DelegateCommand(Login);
            return _loginCommand;
        }
    }

    public bool IsNavigationTarget(NavigateContext context) => false;

    public void OnNavigatedFrom(NavigateContext context) { }

    public void OnNavigatedTo(NavigateContext context)
    {
        _returnUrl = context.Parameters.GetValue<string>("returnUrl");
        _parameters = context.GetOriginalParameters();
    }

    private void Login()
    {
        _auth.Login();
        _navigationService.Replace(_returnUrl, _parameters);
    }
}
```

Example: try to navigate to a protected resource and redirect to LoginView

```cs
public class ProtectedViewModel : BindableBase, ISupportNavigation, ISupportActivationGuard
{
    private readonly IAuth _auth;

    public ProtectedViewModel(IAuth auth)
    {
        _auth = auth;
    }

    public bool IsNavigationTarget(NavigateContext context) => false;

    public void OnNavigatedFrom(NavigateContext context)
    { }

    public void OnNavigatedTo(NavigateContext context)
    {
        var id = context.Parameters.GetValue<string>("id");
        // etc.
    }

    public void CanActivate(NavigateContext context, Action<bool> continuationCallback)
    {
        if (!_auth.IsLogged)
        {
            context.NavigationService.Navigate("LoginView?returnUrl=ProtectedView", context.Parameters);
            continuationCallback(false);
        }
        else
            continuationCallback(true);
    }
}
```

### Registering views (and view models) for navigation

`ConfigurableNavigationService` is easy to configure (do not require to use a bootstrapper)

```cs
var navigationService = new ConfigurableNavigationService
{
    PageAssociations = new Dictionary<string, Type>
    {
        { "HomeView", typeof(HomeView) },
        { "ViewA", typeof(ViewA) },
        // etc.
    }
};
```

Or use Scrutor to find and register all views of a namespace

```cs
var navigationService = new ConfigurableNavigationService();
navigationService.RegisterViewsInExactNamespaceOf<Shell>(); // or RegisterViewsInNamespaceOf
```

`NavigationService`: it's better to use a Bootstrapper (to register views for navigation) ...

## Bootstrapper

Examples

With `MvvmLib.Unity`

```cs
public class Bootstrapper : UnityBootstrapperBase
{
    protected override void RegisterTypes()
    {
        // Services
        Container.RegisterSingleton<INavigationService, NavigationService>();
        Container.RegisterSingleton<IAuth, FakeAuth>();

        // ViewModels
        Container.RegisterForNavigation<ViewBViewModel>();
        Container.RegisterViewModelsInExactNamespaceOf<ShellViewModel>();

        // Register For Navigation
        Container.RegisterForNavigationInNamespaceOf<Shell>(); // here
    }

    protected override Window CreateShell() => Container.Resolve<Shell>();
}
```


With `MvvmLib.Microsoft.DependencyInjection.Extensions`

```cs
public class Bootstrapper : MicrosoftDependencyInjectionBootstrapperBase
{
    protected override void RegisterTypes()
    {
        // Services
        Services.AddSingleton<INavigationService, NavigationService>();
        Services.AddSingleton<IAuth, FakeAuth>();
        // ViewModels
        Services.RegisterForNavigation<ViewBViewModel>();
        Services.RegisterViewModelsInExactNamespaceOf<ShellViewModel>();

        // Register For Navigation
        Services.RegisterForNavigationInNamespaceOf<Shell>(); // here
    }

    protected override Window CreateShell() => ContainerLocator.Current.Resolve<Shell>();
}
```

With `MvvmLib.IoC.Extensions`: all can be auto resolved

```cs
public class Bootstrapper : InjectorBootstrapperBase
{
    protected override void RegisterTypes()
    {
    }

    protected override Window CreateShell() => ContainerLocator.Current.Resolve<Shell>();
}
```

App.xaml: Replace StartupUri by Startup event

```xml
<Application x:Class="WpfApp1.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:local="clr-namespace:WpfApp1"
             Startup="Application_Startup">
    <Application.Resources>
         
    </Application.Resources>
</Application>
```

In code behind

```cs
public partial class App : Application
{
    private void Application_Startup(object sender, StartupEventArgs e)
    {
        new Bootstrapper().Run();
    }
}
```

### Create a Login Screen or a SplashScreen

```cs
public class Bootstrapper : UnityBootstrapperBase
{
    // etc.
    
    protected override Window CreateShell() => Container.Resolve<Shell>();

    protected override void ShowShell()
    {
        var window = new LoginOrSplashWindow();
        if (window.ShowDialog() == true)
            base.ShowShell();
        else
            Application.Current.Shutdown();
    }
}
```

Note: With a SplashScreenManager, `BeforeCreatingShell` method can be used.


## ContainerLocator

Is used by MvvmLib to resolve all dependencies and is configured by the bootstrapper in background.

Manually (without Bootstrapper), example with Unity:

```cs
// Container is IUnityContainer
ContainerLocator.SetContainerLocationProvider(() => new UnityContainerLocationProvider(Container))
```

Resolve a dependency

```cs
var shell = ContainerLocator.Current.Resolve<Shell>();
```

### Mvvm Support

Navigation

* ISupportNavigation: IsNavigationTarget to manage the view resolved, OnNavigatedFrom and OnNavigatedTo methods
* ISupportJournal: to not persist a view in journal
* ISupportLoaded with ViewModel.EnableLoaded attached property on a View
* ISupportActivation: notified when a view is active/ selected

Guards:

* ISupportActivationGuard
* ISupportDeactivationGuard


Example confirm  navigation

```cs
public class ViewAViewModel : BindableBase, ISupportNavigation, ISupportActivationGuard, ISupportDeactivationGuard
{
    // etc.

    public void CanActivate(NavigateContext context, Action<bool> continuationCallback)
    {
        var can = MessageBox.Show($"Activate {nameof(ViewAViewModel)}?", "Confirmation", MessageBoxButton.OKCancel) == MessageBoxResult.OK;
        continuationCallback(can);
    }

    public void CanDeactivate(NavigateContext context, Action<bool> continuationCallback)
    {
        var can = MessageBox.Show($"Deactivate {nameof(ViewAViewModel)}?", "Confirmation", MessageBoxButton.OKCancel) == MessageBoxResult.OK;
        continuationCallback(can);
    }

    // returns always the same View/ViewModel if the pageKey of the uri is View (Usefull for Master Details Scenario for example)
    public bool IsNavigationTarget(NavigateContext context) => context.PageKey == "ViewA";

    public void OnNavigatedFrom(NavigateContext context)  { }

    public void OnNavigatedTo(NavigateContext context)
    {
        var id = context.Parameters.GetValue<string>("id");
    }
}
```

### Handle Loaded from ViewModel

Add the `EnabledLoaded` attached property on the View

```xml
<Window
        ...
        xmlns:ml="http://mvvmlib.com/"
        ml:ViewModel.EnableLoaded="True" />
```

Implement `ISupportLoaded`

```cs
public class ShellViewModel : ISupportLoaded
{
    public void OnLoaded()
    {
        // do something
    }
}
```


## Change the Mvvm interfaces used by the Navigation Service

Example: replace to use Prism interfaces

```cs
public class PrismNavigationService : NavigationService
{
    #region Mvvm

    protected override NavigateContext CreateContext(Uri uri, INavigateParameters parameters, NavigateMode mode)
    {
        var context = new PrismNavigateContext(uri, parameters, mode)
        {
            NavigationService = this
        };
        return context;
    }

    protected override void TryAutoWireViewModel(object content)
    {
        if (content is FrameworkElement view && view.DataContext is null && PrismViewModelLocator.GetAutoWireViewModel(view) is null)
            PrismViewModelLocator.SetAutoWireViewModel(view, true);
    }

    protected override bool IsNavigationTarget(object content, NavigateContext context)
    {
        bool isNavigationTarget = false;
        MvvmUtils.ViewAndViewModelAction<INavigationAware>(content, ina => { isNavigationTarget = ina.IsNavigationTarget(context.AsPrism()); });
        return isNavigationTarget;
    }

    protected override void CanNavigate(object currentContent, object content, NavigateContext context, Action<bool> continuationCallback)
    {
        if (currentContent is IConfirmNavigationRequest)
            ((IConfirmNavigationRequest)currentContent).ConfirmNavigationRequest(context.AsPrism(), continuationCallback);
        else if (currentContent is FrameworkElement element && element.DataContext is IConfirmNavigationRequest)
            ((IConfirmNavigationRequest)element.DataContext).ConfirmNavigationRequest(context.AsPrism(), continuationCallback);
        else
            continuationCallback(true);
    }

    protected override bool PersistInJournal(object content)
    {
        bool persist = true;
        MvvmHelpers.ViewAndViewModelAction<IJournalAware>(content, ija => { persist &= ija.PersistInHistory(); });
        return persist;
    }

    protected override void SetActive(object content, bool isActive)
    {
        MvvmUtils.ViewAndViewModelAction<IActiveAware>(content, iaa => iaa.IsActive = isActive);
    }

    protected override void OnNavigatedFrom(object content, NavigateContext context)
    {
        MvvmUtils.ViewAndViewModelAction<INavigationAware>(content, x => x.OnNavigatedFrom(context.AsPrism()));
    }

    protected override void OnNavigatedTo(object content, NavigateContext context)
    {
        MvvmUtils.ViewAndViewModelAction<INavigationAware>(content, ina => ina.OnNavigatedTo(context.AsPrism()));
    }

    #endregion
}

public class PrismNavigateContext : NavigateContext
{
    public PrismNavigateContext(Uri uri, INavigateParameters parameters, NavigateMode mode) : base(uri, parameters, mode)
    {
        NavigationContext = CreatePrismNavigationContext();
    }

    private NavigationContext CreatePrismNavigationContext()
    {
        var parameters = Parameters.ToPrism();
        var navigationContext = new MvvmLibNavigationContext(this, Uri, parameters);
        return navigationContext;
    }

    public NavigationContext NavigationContext { get; }
}

public static class NavigateContextExtensions
{
    public static NavigationContext AsPrism(this NavigateContext navigateContext)
    {
        if (navigateContext is PrismNavigateContext prismNavigateContext)
            return prismNavigateContext.NavigationContext;
        return null;
    }
}

public static class INavigateParametersExtensions
{
    public static NavigationParameters ToPrism(this INavigateParameters parameters)
    {
        var navigationParameters = new NavigationParameters();
        foreach (var parameter in parameters)
            navigationParameters.Add(parameter.Key, parameter.Value);
        return navigationParameters;
    }
}

public class MvvmLibNavigationContext : NavigationContext
{
    public MvvmLibNavigationContext(NavigateContext context, Uri uri, NavigationParameters navigationParameters)
        : base(null, uri, navigationParameters)
    {
        Context = context;
        GetNavigationParameters(navigationParameters);
    }

    public INavigationService MvvmLibNavigationService => Context.NavigationService;
    public NavigateContext Context { get; }

    private void GetNavigationParameters(NavigationParameters navigationParameters)
    {
        if (navigationParameters != null)
        {
            foreach (KeyValuePair<string, object> navigationParameter in navigationParameters)
            {
                Parameters.Add(navigationParameter.Key, navigationParameter.Value);
            }
        }
    }
}

public static class NavigationContextExtensions
{
    public static T As<T>(this NavigationContext navigationContext) where T: NavigationContext
    {
        if (navigationContext is T objAsT)
            return objAsT;
        return default;
    }
}
```

## Create a custom Navigation Service

Example with NavigationBehaviors and binding for Selectors (ListBox mutliple, TabControl, etc.)

```cs
public class CustomNavigationService : NavigationService, ICustomNavigationService
{
    private INavigationJournal _journal;
    private List<INavigationBehavior> _behaviors;

    public CustomNavigationService()
    {
        Views = new ObservableCollection<object>();
        ActiveViews = new ObservableCollection<object>();
        _behaviors = new List<INavigationBehavior>();
        _journal = new CustomJournal();
    }

    public ObservableCollection<object> Views { get; }
    public ObservableCollection<object> ActiveViews { get; }

    public override INavigationJournal Journal => _journal;

    public IEnumerable<INavigationBehavior> Behaviors => _behaviors;

    protected override void Activate(object content)
    {
        // single and multiple
        if (Views.IndexOf(content) == -1)
            Views.Insert(Journal.CurrentEntryIndex, content);

        // multiple
        if (ActiveViews.IndexOf(content) == -1)
            ActiveViews.Insert(Journal.CurrentEntryIndex, content);

        base.Activate(content);
    }

    public virtual void Attach(INavigationBehavior behavior)
    {
        if (behavior is null)
            throw new ArgumentNullException(nameof(behavior));

        behavior.NavigationService = this;
        behavior.Attach();
        _behaviors.Add(behavior);
    }

    public void Remove(object content)
    {
        base.Remove(content, r =>
        {
            // multiple
            ActiveViews.Remove(content);

            // single
            Views.Remove(content);
        });
    }

    public override void Clear()
    {
        Views.Clear();
        ActiveViews.Clear();
        base.Clear();
    }
}

public class SyncStackPanelBehavior : NavigationBehavior<StackPanel>
{
    public SyncStackPanelBehavior(StackPanel associatedObject) : base(associatedObject)
    {
    }

    protected CustomNavigationService CustomNavigationService => NavigationService as CustomNavigationService;

    protected override void OnAttach()
    {

        CustomNavigationService.Views.CollectionChanged += (s, e) =>
        {
            switch (e.Action)
            {
                case NotifyCollectionChangedAction.Add:
                    {
                        foreach (var item in e.NewItems)
                        {
                            if (item is FrameworkElement element)
                            {
                                if (TargetElement.Children.IndexOf(element) == -1)
                                    TargetElement.Children.Add(EnsureNewView(element));
                            }
                        }
                    }
                    break;
            }
        };

    }

    protected virtual FrameworkElement EnsureNewView(object content)
    {
        if (content is FrameworkElement)
        {
            var newContent = ContainerLocator.Current.Resolve(content.GetType()) as FrameworkElement;
            // if no AutoWireViewModel attached property on view or dataContext not defined at construction
            MvvmUtils.TryAutoWireViewModel(newContent);
            return newContent;
        }
        return null;
    }

    protected override void OnDetach()
    { }
}

// Example: custom navigation journal: insert each entry at beginning
public class CustomJournal : NavigationJournal
{
    protected override int GetNewEntryPosition() => 0;

    protected override void RemoveEntries(int startIndex)
    {
        while (EntriesInternal.Count < startIndex)
            EntriesInternal.RemoveAt(startIndex);
    }
}
```

Binding: Example ListBox Multiple

```xml
<ListBox ItemsSource="{Binding NavigationService.Views}" 
            SelectionMode="Multiple"
            SelectedItem="{Binding NavigationService.Content}"  
            Grid.Row="1">
    <ml:Interaction.Behaviors>
        <ml:SelectorSelectedItemsSyncBehavior ActiveItems="{Binding NavigationService.ActiveViews}" />
    </ml:Interaction.Behaviors>
</ListBox>
```

Inject the navigation service in MainWindow to add the behavior

```cs
public partial class MainWindow : Window
{
    public MainWindow(ICustomNavigationService navigationService)
    {
        InitializeComponent();

        navigationService.Attach(new SyncStackPanelBehavior(Panel));
    }
}
```

## Data

### ListCollectionViewEx

Is a ListCollectionView with shortcuts and usefull methods

```cs
public class ListCollectionViewSampleViewModel
{
    public ListCollectionViewSampleViewModel()
    {
        var users = new List<User>
        {
            new User { Id = 1, FirstName = "First.1", LastName ="Last.1", Age = 30, Role = UserRole.Admin },
            new User { Id = 2, FirstName = "First.2", LastName ="Last.2", Age = 40, Role = UserRole.User },
            new User { Id = 3, FirstName = "First.3", LastName ="Last.3", Age = 50, Role = UserRole.SuperAdmin },
            // ...
        };
        View = new ListCollectionViewEx<User>(users);

        Commands = new ListCollectionViewCommands(View)
        {
            Delete = Delete,
            Save = Save,
            Sort = new Action<string, ListSortDirection>((p, d) => View.Sort(p, d)),
            Filter = Filter
        };
    }

    public ListCollectionViewEx<User> View { get; }
    public ListCollectionViewCommands Commands { get; }

    private void Filter(string args)
    {
        if (string.IsNullOrWhiteSpace(args))
            View.ClearFilter();
        else
        {
            var splits = SplitText(args);
            if (splits.Length > 0)
            {
                var compositeFilter = new CompositeFilter<User>(LogicalOperator.Or);
                foreach (var split in splits)
                {
                    compositeFilter.AddFilter(new PropertyFilter<User>("FirstName", PredicateOperator.Contains, split));
                    compositeFilter.AddFilter(new PropertyFilter<User>("LastName", PredicateOperator.Contains, split));
                }
                View.Filter = compositeFilter.Filter;
            }
        }
    }

    private string[] SplitText(string text) => text.Split(' ', ',');

    private void Save()
    {
        try
        {
            if (View.IsAddingNew)
            {
                var current = View.CurrentAddItem as User;
                current.ValidateAll();
                if (!current.HasErrors)
                {
                    // save to db...
                    View.CommitNew();
                }
            }
            else if (View.IsEditingItem)
            {
                var current = View.CurrentEditItem as User;
                current.ValidateAll();
                if (!current.HasErrors)
                {
                    // save to db ..
                    View.CommitEdit();
                }
            }
        }
        catch (Exception ex)
        { }
    }

    private void Delete()
    {
        var current = View.Current;
        if (MessageBox.Show($"Delete {current.FirstName}?", "Confirmation", MessageBoxButton.OKCancel) == MessageBoxResult.OK)
        {
            try
            {
                // remove from db ...
                View.Remove(current);
                // eventAggregator.GetEvent<NotificationMessageEvent>().Publish($"{name} removed!");
            }
            catch (Exception ex)
            {
                MessageBox.Show($"A problem occured:{ex.Message}");
            }
        }
    }
}

public class User : Validatable, IEditableObject
{
    private User oldUser = null;

    public void BeginEdit()
    {
        oldUser = new User
        {
            Id = Id,
            FirstName = FirstName,
            LastName = LastName,
            ImagePath = ImagePath,
            Age = Age,
            Role = Role
        };
    }

    public void CancelEdit()
    {
        if (oldUser == null)
            throw new InvalidOperationException();

        FirstName = oldUser.FirstName;
        LastName = oldUser.LastName;
        ImagePath = oldUser.ImagePath;
        Age = oldUser.age;
        Role = oldUser.role;
    }

    public void EndEdit()
    {
        oldUser = null;
    }

    private string firstName;
    [Required]
    public string FirstName
    {
        get { return firstName; }
        set { SetProperty(ref firstName, value); }
    }

    // etc.
}
```

### PagedList (IEnumerable T extension method)

```cs
int pageNumber = 1;
int pageSize = 50;
var pagedList = Users.ToPagedList<User>(pageNumber, pageSize);
```

Binding

```xml
<ListBox ItemsSource="{Binding PagedList}"></ListBox>
```

### PagedSource


```cs
public class PagedSourceViewModel : BindableBase
{
    public PagedSourceViewModel()
    {
        Users = new ObservableCollection<User>();
        PopulateUsers();
        PagedSource = new PagedSource(Users);
        Commands = new PagedSourceCommands(PagedSource);

        AddNewItemCommand = new DelegateCommand(AddItem);
    }

    public ObservableCollection<User> Users { get; }
    public PagedSource PagedSource { get; }
    public PagedSourceCommands Commands { get; }
    public ICommand AddNewItemCommand { get; }

    private void AddItem()
    {
        var user = new User();
        PagedSource.View.AddNewItem(user);
        PagedSource.MoveToLastPage();
        PagedSource.PageView.MoveCurrentToLast();
    }

    // etc.
}
```

Binding

```xml
<!-- commands -->
<Button Command="{Binding Commands.MoveCurrentToFirstCommand}" >
    <controls:MaterialDesignIcon Icon="SkipPrevious" Brush="#2980b9"/>
</Button>

<!-- currentItem -->
<ContentControl Content="{Binding PagedSource.PageView.CurrentItem, Mode=OneWay}"></ContentControl>
```

It's possible to create a `DataPager` and the methods of the PagedSource

* MoveToFirstPage
* MoveToPreviousPage
* MoveToNextPage
* MoveToLastPage
* MoveToPage

## Controls

### AnimatingContentControl

> Content Control that allows to animate on content change. 

2 Storyboards : 

* EntranceAnimation 
* ExitAnimation
* Simultaneous (boolean) allows to play simultaneously the animations.
* CanAnimateOnLoad: allows to cancel animation on load

```xml
<ml:AnimatingContentControl ml:NavigationManager.SourceName="Main">
    <ml:AnimatingContentControl.EntranceAnimation>
        <Storyboard>
            <!-- Target "CurrentContentPresenter"  -->
            <DoubleAnimation Storyboard.TargetName="CurrentContentPresenter" 
                             Storyboard.TargetProperty="(UIElement.RenderTransform).(TransformGroup.Children)[3].(TranslateTransform.X)"
                             From="400" To="0" Duration="0:0:0.4"  />
        </Storyboard>
    </ml:AnimatingContentControl.EntranceAnimation>
    <ml:AnimatingContentControl.ExitAnimation>
        <Storyboard>
            <!-- Target "CurrentContentPresenter" or with Simultaneous "PreviousContentPresenter" -->
            <DoubleAnimation Storyboard.TargetName="CurrentContentPresenter" 
                             Storyboard.TargetProperty="(UIElement.RenderTransform).(TransformGroup.Children)[3].(TranslateTransform.X)"
                             From="0" To="400" Duration="0:0:0.4"  />
        </Storyboard>
    </ml:AnimatingContentControl.ExitAnimation>
</ml:AnimatingContentControl>
```

Or Simultaneous

```xml
  <ml:AnimatingContentControl Content="{Binding Navigation.Current}" 
                                    Simultaneous="True"
                                    IsCancelled="{Binding IsCancelled}">
    <ml:AnimatingContentControl.EntranceAnimation>
        <Storyboard>
            <DoubleAnimation  Storyboard.TargetName="CurrentContentPresenter"
                                Storyboard.TargetProperty="(UIElement.RenderTransform).(TransformGroup.Children)[3].(TranslateTransform.X)"
                                From="{Binding ElementName=ThisControl,Path=ActualWidth,FallbackValue=400}" To="0" 
                                Duration="{Binding ElementName=DuractionComboBox,Path=SelectedItem}">
                <DoubleAnimation.EasingFunction>
                    <SineEase EasingMode="EaseInOut" />
                </DoubleAnimation.EasingFunction>
            </DoubleAnimation>
        </Storyboard>
    </ml:AnimatingContentControl.EntranceAnimation>
    <ml:AnimatingContentControl.ExitAnimation>
        <Storyboard>
            <DoubleAnimation  Storyboard.TargetName="PreviousContentPresenter"
                                Storyboard.TargetProperty="(UIElement.RenderTransform).(TransformGroup.Children)[3].(TranslateTransform.X)"
                                From="0" To="{Binding ElementName=ThisControl,Path=ActualWidth,FallbackValue=400}" 
                                Duration="{Binding ElementName=DuractionComboBox,Path=SelectedItem}">
                <DoubleAnimation.EasingFunction>
                    <SineEase EasingMode="EaseInOut" />
                </DoubleAnimation.EasingFunction>
            </DoubleAnimation>
        </Storyboard>
    </ml:AnimatingContentControl.ExitAnimation>
</ml:AnimatingContentControl>
```

Other sample: animations in resources


```xml
<UserControl.Resources>
    <Storyboard x:Key="EntranceAnimation1">
        <DoubleAnimation Storyboard.TargetName="CurrentContentPresenter" 
                         Storyboard.TargetProperty="(UIElement.RenderTransform).(TransformGroup.Children)[3].(TranslateTransform.X)"
                         From="400" To="0" Duration="0:0:0.4"  />
    </Storyboard>

    <Storyboard x:Key="ExitAnimation1">
        <DoubleAnimation Storyboard.TargetName="CurrentContentPresenter" 
                         Storyboard.TargetProperty="(UIElement.RenderTransform).(TransformGroup.Children)[3].(TranslateTransform.X)"
                         From="0" To="-400" Duration="0:0:0.4"  />
    </Storyboard>
</UserControl.Resources>
```

```xml
<ml:AnimatingContentControl Content="{Binding Navigation.Current}" 
                                 EntranceAnimation="{StaticResource EntranceAnimation1}"
                                 ExitAnimation="{StaticResource ExitAnimation1}">
</ml:AnimatingContentControl>
```

Other sample: Change Animations dynamically and controlling when the animation is played with "CanAnimate". For a Schedule view for example.

```xml
<ml:AnimatingContentControl  Content="{Binding Navigation.Current}" 
                                  CanAnimate="{Binding CanAnimate, Mode=OneWay}" 
                                  CanAnimateOnLoad="False"
                                  EntranceAnimation="{Binding EntranceAnimation, Mode=OneWay}"
                                  ExitAnimation="{Binding ExitAnimation, Mode=OneWay}">

    <ml:Interaction.Triggers>
        <ml:EventTrigger EventName="AnimationCompleted">
            <ml:CallMethodAction TargetObject="{Binding}" MethodName="CompleteChangingScheduleMode"/>
        </ml:EventTrigger>
    </ml:Interaction.Triggers>

</ml:AnimatingContentControl>
```


### TransitioningContentControl

> Allows to play a transition on loaded.

2 Storyboards:

* EntranceTransition: played when control loaded (or explicitly with "DoEnter")
* ExitTransition: played explicitly with "DoLeave" or IsLeaving dependency property (for example played when the user click on a tab close button)

Other methods:

* CancelTransition
* Reset: reset the render transform property and opacity + cancel transition

```xml
<ml:TransitioningContentControl x:Name="TransitioningContentControl1" Margin="0,20">
        <ml:TransitioningContentControl.EntranceTransition>
            <Storyboard>
                <DoubleAnimation Storyboard.TargetName="ContentPresenter" 
                                    Storyboard.TargetProperty="(UIElement.RenderTransform).(TransformGroup.Children)[0].(ScaleTransform.ScaleY)" 
                                    From="0" To="1" Duration="0:0:0.6">
                    <DoubleAnimation.EasingFunction>
                        <ExponentialEase EasingMode="EaseInOut"/>
                    </DoubleAnimation.EasingFunction>
                </DoubleAnimation>
            </Storyboard>
        </ml:TransitioningContentControl.EntranceTransition>
        <ml:TransitioningContentControl.ExitTransition>
            <Storyboard>
                <DoubleAnimation Storyboard.TargetName="ContentPresenter" 
                                    Storyboard.TargetProperty="(UIElement.Opacity)" 
                                    From="1" To="0" Duration="0:0:2"/>
            </Storyboard>
</ml:TransitioningContentControl.ExitTransition>
```

### TransitioningItemsControl

> ItemsControl that allows to animate on item insertion and deletion. 

The "ControlledAnimation" avoid to set the target and the target property of the storyboard. The TargetPropertyType is a shortcut. But it's possible to target explicitly the target property of the storyboard with "TargetProperty" dependency property.

```xml
<ml:TransitioningItemsControl ItemsSource="{Binding MyItems}" 
                                   TransitionClearHandling="Transition"
                                   IsCancelled="{Binding IsCancelled}">
    <ml:TransitioningItemsControl.EntranceAnimation>
        <ml:ParallelAnimation>

            <ml:ControlledAnimation TargetPropertyType="TranslateX">
                <DoubleAnimation From="200" To="0"  Duration="0:0:2"/>
            </ml:ControlledAnimation>

        </ml:ParallelAnimation>
    </ml:TransitioningItemsControl.EntranceAnimation>

    <ml:TransitioningItemsControl.ExitAnimation>
        <ml:ParallelAnimation>
            <ml:ControlledAnimation TargetPropertyType="TranslateX">
                <DoubleAnimation From="0" To="200" Duration="0:0:2"/>
            </ml:ControlledAnimation>
        </ml:ParallelAnimation>
    </ml:TransitioningItemsControl.ExitAnimation>
</ml:TransitioningItemsControl>
```

### NavigatableContentControl

It's a control that allows to use navigation service méthods (like a Frame). 

Example with mahApps.Metro, sets the content of the <a href="https://mahapps.com/docs/controls/HamburgerMenu" target="_blank">HamburgerMenu</a>

```cs
public partial class Shell
{
    public Shell(IConfigurableNavigationService navigationService)
    {
        InitializeComponent();

        var navigatableContentControl = new NavigatableContentControl();
        navigatableContentControl.NavigationService = navigationService;
        HamburgerMenuControl.Content = navigatableContentControl;
    }
}
```

Example 2 xaml

```xml
<!-- xmlns:ml="http://mvvmlib.com/" -->
<ml:NavigatableContentControl x:Name="NavigatableContentControl"/>
```

## Interactivity

### Triggers and TriggerActions

EventTrigger

```xml
<Button x:Name="Button1" Content="Click!">
    <ml:Interaction.Triggers>
        <ml:EventTrigger EventName="Click">
            <ml:CallMethodAction TargetObject="{Binding}" MethodName="SayHello" Parameter="My parameter" />
            <ml:InvokeCommandAction Command="{Binding MyCommand}" CommandParameter="My parameter"/>
            <ml:ChangePropertyAction TargetObject="{Binding ElementName=Button1}" PropertyPath="Foreground" Value="Red"/>
        </ml:EventTrigger>
    </ml:Interaction.Triggers>
</Button>
```

DataTrigger

```xml
<TextBlock x:Name="TextBlock1" Text="{Binding MyValue}">
    <ml:Interaction.Triggers>
        <ml:DataTrigger Binding="{Binding MyValue}" Value="My value">
            <ml:ChangePropertyAction TargetObject="{Binding ElementName=TextBlock1}" PropertyPath="Foreground" Value="Red"/>
        </ml:DataTrigger>
        <ml:DataTrigger Binding="{Binding MyValue}" Comparison="NotEqual" Value="My value">
            <ml:ChangePropertyAction TargetObject="{Binding ElementName=TextBlock1}" PropertyPath="Foreground" Value="Blue"/>
        </ml:DataTrigger>
    </ml:Interaction.Triggers>
</TextBlock>
```

Triggers:

* EventTrigger
* DataTrigger

TriggerActions:

* CallMethodAction
* InvokeCommandAction
* ChangePropertyAction
* GoToStateAction

It's possible to create custom Triggers (inherit from TriggerBase) and TriggerActions (inherit from TriggerAction and implement Invoke method).

### Behaviors

EventToCommandBehavior

```xml
<Button Content="Click event">
    <ml:NavigationInteraction.Behaviors>
        <ml:EventToCommandBehavior EventName="Click" Command="{Binding MessageCommand}" CommandParameter="World"/>
    </ml:NavigationInteraction.Behaviors>
</Button>
```

SelectorSelectedItemsSyncBehavior


```xml
<ListBox ItemsSource="{Binding Users}" 
            SelectionMode="Multiple"
            SelectedItem="{Binding SelectedUser}"
            MinHeight="100">
    <ml:Interaction.Behaviors>
        <ml:SelectorSelectedItemsSyncBehavior ActiveItems="{Binding SelectedUsers}"/>
    </ml:Interaction.Behaviors>
    <ListBox.ItemTemplate>
        <DataTemplate>
            <TextBlock Text="{Binding Name}" />
        </DataTemplate>
    </ListBox.ItemTemplate>
</ListBox>
```

TreeViewSelectedItemChangedBehavior 

```xml
<TreeView ItemsSource="{Binding Families}" MinHeight="100">
    <TreeView.Resources>
        <HierarchicalDataTemplate DataType="{x:Type viewModels:Family}" ItemsSource="{Binding Members}">
            <StackPanel Orientation="Horizontal">
                <TextBlock Text="{Binding Name}" />
                <TextBlock Text=" [" Foreground="Blue" />
                <TextBlock Text="{Binding Members.Count}" Foreground="Blue" />
                <TextBlock Text="]" Foreground="Blue" />
            </StackPanel>
        </HierarchicalDataTemplate>
        <DataTemplate DataType="{x:Type viewModels:FamilyMember}">
            <StackPanel Orientation="Horizontal">
                <TextBlock Text="{Binding Name}" />
                <TextBlock Text=" (" Foreground="Green" />
                <TextBlock Text="{Binding Age}" Foreground="Green" />
                <TextBlock Text=" years)" Foreground="Green" />
            </StackPanel>
        </DataTemplate>
    </TreeView.Resources>
    <ml:Interaction.Behaviors>
        <ml:TreeViewSelectedItemChangedBehavior 
            ExpandSelected="True" 
            SelectedItem="{Binding ActiveFamilyOrMember}" />
    </ml:Interaction.Behaviors>
</TreeView>
```

## BindingProxy

Example:

```xml
<DataGrid x:Name="DataGrid1" ItemsSource="{Binding CollectionView}" AutoGenerateColumns="False" IsReadOnly="True">
    <DataGrid.Resources>
        <!-- 1. Adds the Proxy in control or window resources-->
        <ml:BindingProxy x:Key="Proxy"  Data="{Binding}"/>
    </DataGrid.Resources>
    <DataGrid.Columns>
        <DataGridTextColumn Binding="{Binding FirstName}" Width="*">
            <DataGridTextColumn.Header>
                <Grid>
                    <Grid.ColumnDefinitions>
                        <ColumnDefinition/>
                        <ColumnDefinition Width="Auto" />
                    </Grid.ColumnDefinitions>

                    <TextBlock Text="Name" />

                    <local:DropDownButton Grid.Column="1">
                        <local:DropDownButton.DropDownContent>
                            <Grid>
                                <Grid.RowDefinitions>
                                    <RowDefinition />
                                    <RowDefinition Height="Auto"/>
                                </Grid.RowDefinitions>

                                <!-- code -->

                                <StackPanel Orientation="Horizontal" Grid.Row="1">
                                    <!-- 2. Use the Proxy as Source and bind with The Data dependency property -->
                                    <Button Content="Filter" Command="{Binding Data.FilterFirstNameCommand, Source={StaticResource Proxy}}" />
                                </StackPanel>
                            </Grid>
                        </local:DropDownButton.DropDownContent>
                    </local:DropDownButton>
                </Grid>
            </DataGridTextColumn.Header>
        </DataGridTextColumn>

        <!-- other columns -->
    </DataGrid.Columns>
</DataGrid>
```
