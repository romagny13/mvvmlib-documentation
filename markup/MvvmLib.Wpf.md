# MvvmLib.Wpf

* Mvvm: **ViewModelLocator**, **BindingProxy**, etc.
* Navigation: **NavigationService**, **ConfigurableNavigationService**
* Data: **ListCollectionViewEx**, **PagedList**, **PagedSource** and commands **ListCollectionViewCommands**, **PagedSourceCommands**
* Interactivity: **CallMethodeAction**, **SelectorSelectedItemsSyncBehavior**, **EventToCommandBehavior**,**EventToMethodBehavior**, etc.
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

#### ConfigurableNavigationService

Easy to configure for navigation (do not require to use a bootstrapper)

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

#### NavigationService with a Bootstrapper

Examples

With `MvvmLib.Unity`

```cs
public class Bootstrapper : UnityBootstrapperBase
{
    protected override void RegisterTypes()
    {
        // Services
        Container.RegisterSingleton<INavigationService, PrismNavigationService>();
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
        Services.AddSingleton<INavigationService, PrismNavigationService>();
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

## ContainerLocator

Is used by MvvmLib to resolve all dependencies and is configured by the bootstrapper in background.

Manually (without Bootstrapper), example with Unity:

```cs
// Container is IUnityContainer
ContainerLocator.SetContainerLocationProvider(() => new UnityContainerLocationProvider(Container))
```

Resolve a dependency

```cs
ContainerLocator.Current.Resolve(typeof(Shell));
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

    public void CanActivate(NavigateContext navigationContext, Action<bool> continuationCallback)
    {
        var can = MessageBox.Show($"Activate {nameof(ViewAViewModel)}?", "Confirmation", MessageBoxButton.OKCancel) == MessageBoxResult.OK;
        continuationCallback(can);
    }

    public void CanDeactivate(NavigateContext navigationContext, Action<bool> continuationCallback)
    {
        var can = MessageBox.Show($"Deactivate {nameof(ViewAViewModel)}?", "Confirmation", MessageBoxButton.OKCancel) == MessageBoxResult.OK;
        continuationCallback(can);
    }

    // returns always the same View/ViewModel if the pageKey of the uri is ViewA
    public bool IsNavigationTarget(NavigateContext navigationContext) => navigationContext.PageKey == "ViewA";

    public void OnNavigatedFrom(NavigateContext navigationContext)  { }

    public void OnNavigatedTo(NavigateContext navigationContext)
    {
        var id = navigationContext.Parameters.GetValue<string>("id");
    }
}
```

### Easy to customize Navigation Service

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
        var callback = new Action<bool>(t =>
        {
            if (t == false)
                continuationCallback(false);
            else
                continuationCallback(t);
        });

        if (currentContent is IConfirmNavigationRequest)
            ((IConfirmNavigationRequest)currentContent).ConfirmNavigationRequest(context.AsPrism(), callback);
        else if (currentContent is FrameworkElement element && element.DataContext is IConfirmNavigationRequest)
            ((IConfirmNavigationRequest)element.DataContext).ConfirmNavigationRequest(context.AsPrism(), callback);
        else
            callback(true);
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
    INavigationJournal _journal;
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

## Controls


### AnimatingContentControl

> Content Control that allows to animate on content change. 

2 Storyboards : 

* EntranceAnimation 
* ExitAnimation
* Simultaneous (boolean) allows to play simultaneously the animations.
* CanAnimateOnLoad: allows to cancel animation on load

```xml
<mvvmLib:AnimatingContentControl mvvmLib:NavigationManager.SourceName="Main">
    <mvvmLib:AnimatingContentControl.EntranceAnimation>
        <Storyboard>
            <!-- Target "CurrentContentPresenter"  -->
            <DoubleAnimation Storyboard.TargetName="CurrentContentPresenter" 
                             Storyboard.TargetProperty="(UIElement.RenderTransform).(TransformGroup.Children)[3].(TranslateTransform.X)"
                             From="400" To="0" Duration="0:0:0.4"  />
        </Storyboard>
    </mvvmLib:AnimatingContentControl.EntranceAnimation>
    <mvvmLib:AnimatingContentControl.ExitAnimation>
        <Storyboard>
            <!-- Target "CurrentContentPresenter" or with Simultaneous "PreviousContentPresenter" -->
            <DoubleAnimation Storyboard.TargetName="CurrentContentPresenter" 
                             Storyboard.TargetProperty="(UIElement.RenderTransform).(TransformGroup.Children)[3].(TranslateTransform.X)"
                             From="0" To="400" Duration="0:0:0.4"  />
        </Storyboard>
    </mvvmLib:AnimatingContentControl.ExitAnimation>
</mvvmLib:AnimatingContentControl>
```

Or Simultaneous

```xml
  <mvvmLib:AnimatingContentControl Content="{Binding Navigation.Current}" 
                                    Simultaneous="True"
                                    IsCancelled="{Binding IsCancelled}">
    <mvvmLib:AnimatingContentControl.EntranceAnimation>
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
    </mvvmLib:AnimatingContentControl.EntranceAnimation>
    <mvvmLib:AnimatingContentControl.ExitAnimation>
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
    </mvvmLib:AnimatingContentControl.ExitAnimation>
</mvvmLib:AnimatingContentControl>
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
<mvvmLib:AnimatingContentControl Content="{Binding Navigation.Current}" 
                                 EntranceAnimation="{StaticResource EntranceAnimation1}"
                                 ExitAnimation="{StaticResource ExitAnimation1}">
</mvvmLib:AnimatingContentControl>
```

Other sample: Change Animations dynamically and controlling when the animation is played with "CanAnimate". For a Schedule view for example.

```xml
<mvvmLib:AnimatingContentControl  Content="{Binding Navigation.Current}" 
                                  CanAnimate="{Binding CanAnimate, Mode=OneWay}" 
                                  CanAnimateOnLoad="False"
                                  EntranceAnimation="{Binding EntranceAnimation, Mode=OneWay}"
                                  ExitAnimation="{Binding ExitAnimation, Mode=OneWay}">

    <mvvmLib:Interaction.Triggers>
        <mvvmLib:EventTrigger EventName="AnimationCompleted">
            <mvvmLib:CallMethodAction TargetObject="{Binding}" MethodName="CompleteChangingScheduleMode"/>
        </mvvmLib:EventTrigger>
    </mvvmLib:Interaction.Triggers>

</mvvmLib:AnimatingContentControl>
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
<mvvmLib:TransitioningContentControl x:Name="TransitioningContentControl1" Margin="0,20">
        <mvvmLib:TransitioningContentControl.EntranceTransition>
            <Storyboard>
                <DoubleAnimation Storyboard.TargetName="ContentPresenter" 
                                    Storyboard.TargetProperty="(UIElement.RenderTransform).(TransformGroup.Children)[0].(ScaleTransform.ScaleY)" 
                                    From="0" To="1" Duration="0:0:0.6">
                    <DoubleAnimation.EasingFunction>
                        <ExponentialEase EasingMode="EaseInOut"/>
                    </DoubleAnimation.EasingFunction>
                </DoubleAnimation>
            </Storyboard>
        </mvvmLib:TransitioningContentControl.EntranceTransition>
        <mvvmLib:TransitioningContentControl.ExitTransition>
            <Storyboard>
                <DoubleAnimation Storyboard.TargetName="ContentPresenter" 
                                    Storyboard.TargetProperty="(UIElement.Opacity)" 
                                    From="1" To="0" Duration="0:0:2"/>
            </Storyboard>
</mvvmLib:TransitioningContentControl.ExitTransition>
```

### TransitioningItemsControl

> ItemsControl that allows to animate on item insertion and deletion. 

The "ControlledAnimation" avoid to set the target and the target property of the storyboard. The TargetPropertyType is a shortcut. But it's possible to target explicitly the target property of the storyboard with "TargetProperty" dependency property.

```xml
<mvvmLib:TransitioningItemsControl ItemsSource="{Binding MyItems}" 
                                   TransitionClearHandling="Transition"
                                   IsCancelled="{Binding IsCancelled}">
    <mvvmLib:TransitioningItemsControl.EntranceAnimation>
        <mvvmLib:ParallelAnimation>

            <mvvmLib:ControlledAnimation TargetPropertyType="TranslateX">
                <DoubleAnimation From="200" To="0"  Duration="0:0:2"/>
            </mvvmLib:ControlledAnimation>

        </mvvmLib:ParallelAnimation>
    </mvvmLib:TransitioningItemsControl.EntranceAnimation>

    <mvvmLib:TransitioningItemsControl.ExitAnimation>
        <mvvmLib:ParallelAnimation>
            <mvvmLib:ControlledAnimation TargetPropertyType="TranslateX">
                <DoubleAnimation From="0" To="200" Duration="0:0:2"/>
            </mvvmLib:ControlledAnimation>
        </mvvmLib:ParallelAnimation>
    </mvvmLib:TransitioningItemsControl.ExitAnimation>
</mvvmLib:TransitioningItemsControl>
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

## Triggers and TriggerActions

Example:

```xml
<Button x:Name="Button1" Content="Click!">
    <mvvmLib:Interaction.Triggers>
        <mvvmLib:EventTrigger EventName="Click">
            <mvvmLib:CallMethodAction TargetObject="{Binding}" MethodName="SayHello" Parameter="My parameter" />
            <mvvmLib:InvokeCommandAction Command="{Binding MyCommand}" CommandParameter="My parameter"/>
            <mvvmLib:ChangePropertyAction TargetObject="{Binding ElementName=Button1}" PropertyPath="Foreground" Value="Red"/>
        </mvvmLib:EventTrigger>
    </mvvmLib:Interaction.Triggers>
</Button>
```

DataTrigger sample

```xml
<TextBlock x:Name="TextBlock1" Text="{Binding MyValue}">
    <mvvmLib:Interaction.Triggers>
        <mvvmLib:DataTrigger Binding="{Binding MyValue}" Value="My value">
            <mvvmLib:ChangePropertyAction TargetObject="{Binding ElementName=TextBlock1}" PropertyPath="Foreground" Value="Red"/>
        </mvvmLib:DataTrigger>
        <mvvmLib:DataTrigger Binding="{Binding MyValue}" Comparison="NotEqual" Value="My value">
            <mvvmLib:ChangePropertyAction TargetObject="{Binding ElementName=TextBlock1}" PropertyPath="Foreground" Value="Blue"/>
        </mvvmLib:DataTrigger>
    </mvvmLib:Interaction.Triggers>
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

Easy to create our owns Triggers (inherit from TriggerBase) and TriggerActions (inherit from TriggerAction and implement Invoke method).

CallMethodAction

```xml
<Button Content="Test">
    <ml:Interaction.Triggers>
        <ml:EventTrigger EventName="Click">
            <ml:CallMethodAction TargetObject="{Binding}" MethodName="MyMethod" Parameter="MvvmLib!"/>
        </ml:EventTrigger>
    </ml:Interaction.Triggers>
</Button>
```

ChangePropertyAction

```xml
<Button Content="Test" HorizontalAlignment="Left" Margin="5">
    <ml:Interaction.Triggers>
        <ml:EventTrigger EventName="Click">
            <ml:InvokeCommandAction Command="{Binding MessageCommand}" CommandParameter="Marie!"/>
        </ml:EventTrigger>
    </ml:Interaction.Triggers>
</Button>
```

InvokeCommandAction

```xml
<Button Content="Test" HorizontalAlignment="Left" Margin="5">
    <ml:Interaction.Triggers>
        <ml:EventTrigger EventName="Click">
            <ml:ChangePropertyAction TargetObject="{Binding}" PropertyPath="MyProperty" Value="New Value!"/>
        </ml:EventTrigger>
    </ml:Interaction.Triggers>
</Button>
```

## Behaviors

EventToCommandBehavior

```xml
<Button Content="Click event">
    <mvvmLib:NavigationInteraction.Behaviors>
        <mvvmLib:EventToCommandBehavior EventName="Click" Command="{Binding MessageCommand}" CommandParameter="World"/>
    </mvvmLib:NavigationInteraction.Behaviors>
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
                <CheckBox IsChecked="{Binding IsActive}" />
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
        <mvvmLib:BindingProxy x:Key="Proxy"  Data="{Binding}"/>
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
