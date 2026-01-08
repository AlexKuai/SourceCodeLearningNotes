# MVC

## 源码涉及的核心类型

- ApplicationPart  
- IApplicationPartTypeProvider  
- AssemblyPart  
- IApplicationFeatureProvider\<\>  
- ControllerFeature  
- ControllerFeatureProvider  
- ApplicationPartManager  
- ApplicationPartFactory  
- DefaultApplicationPartFactory  
- ActionDescriptor  
- ControllerActionDescriptor  
- ActionDescriptorProviderContext  
- IActionDescriptorProvider  
- ControllerActionDescriptorProvider  
- ApplicationModel  
- ControllerModel  
- ActionModel  
- SelectorModel  
- IApplicationModelConvention  
- ApplicationModelProviderContext  
- IApplicationModelProvider  
- DefaultApplicationModelProvider  
- ApiBehaviorApplicationModelProvider  
- ApplicationModelFactory  
- IActionDescriptorChangeProvider  
- ActionDescriptorChangeProvider  
- IActionDescriptorCollectionProvider  
- ActionDescriptorCollectionProvider  
- DefaultActionDescriptorCollectionProvider  
- ControllerActionEndpointConventionBuilder  
- ActionEndpointDataSourceBase  
- ControllerActionEndpointDataSource  
- ActionEndpointFactory  
- ControllerEndpointRouteBuilderExtensions  
- ControllerActionEndpointDataSourceFactory  
- ActionContext  
- ControllerContext  
- IRequestDelegateFactory  
- ControllerRequestDelegateFactory  
- IActionInvoker  
- ResourceInvoker  
- ControllerActionInvoker  
- ProblemDetails  
- ProblemDetailsClientErrorFactory  
- DefaultProblemDetailsFactory  
- IFilterMetadata  
- IOrderedFilter  
- IFilterFactory  
- TypeFilterAttribute  
- ClientErrorResultFilterFactory  
- ClientErrorResultFilter  
- ModelStateInvalidFilterFactory  
- ModelStateInvalidFilter  

## MVC 应用程序模型

- ApplicationPart

```C#
// 应用程序组成部分
public abstract class ApplicationPart
{
    public abstract string Name { get; }
}
```

- IApplicationPartTypeProvider

```C#
// 应用程序组成部分类型提供器接口
public interface IApplicationPartTypeProvider
{
    // 返回类型集合
    IEnumerable<TypeInfo> Types { get; }
}
```

- AssemblyPart

```C#
// 表示应用程序组成部分中的程序集部分
public class AssemblyPart : ApplicationPart, IApplicationPartTypeProvider
{
    public AssemblyPart(Assembly assembly)
    {
        Assembly = assembly ?? throw new ArgumentNullException(nameof(assembly));
    }
 
    public Assembly Assembly { get; }
 
    public override string Name => Assembly.GetName().Name!;
 
    // 返回程序集中定义的所有类型
    public IEnumerable<TypeInfo> Types => Assembly.DefinedTypes;
}
```

- IApplicationFeatureProvider\<\>

```C#
// 应用程序特性提供器接口
// 实现的 IApplicationFeatureProvider 属于空接口，作为标记接口使用
public interface IApplicationFeatureProvider<TFeature> : IApplicationFeatureProvider
{
    // 利用 ApplicationPart 集合配置 TFeature 特性
    void PopulateFeature(IEnumerable<ApplicationPart> parts, TFeature feature);
}
```

- ControllerFeature

```C#
// 控制器特性
public class ControllerFeature
{
    // 控制器类型集合
    // 用于收集应用程序内所有满足控制器要求的类型
    public IList<TypeInfo> Controllers { get; } = new List<TypeInfo>();
}
```

- ControllerFeatureProvider

```C#
// 控制器特性提供器，实现 IApplicationFeatureProvider<ControllerFeature>
public class ControllerFeatureProvider : IApplicationFeatureProvider<ControllerFeature>
{
    // 表示控制器类型的名称后缀
    private const string ControllerTypeNameSuffix = "Controller";
 
    // 从 ApplicationPart 集合中筛选出实现 IApplicationPartTypeProvider 的 ApplicationPart
    // 然后遍历这些 ApplicationPart.Types 集合收集满足控制器要求的类型添加到 ControllerFeature.Controllers 集合中
    public void PopulateFeature(
        IEnumerable<ApplicationPart> parts,
        ControllerFeature feature)
    {
        foreach (var part in parts.OfType<IApplicationPartTypeProvider>())
        {
            foreach (var type in part.Types)
            {
                if (IsController(type) && !feature.Controllers.Contains(type))
                {
                    feature.Controllers.Add(type);
                }
            }
        }
    }
 
    // 满足控制器要求的类型必须同时达成以下条件：
    // 1. 是类
    // 2. 不是抽象类
    // 3. 是公开的
    // 4. 不是泛型类
    // 5. 类型上没有绑定 NonControllerAttribute 特性
    // 6. 类型上绑定 ControllerAttribute 特性或类型名称以 "Controller" 结尾（忽略大小写）
    protected virtual bool IsController(TypeInfo typeInfo)
    {
        if (!typeInfo.IsClass)
        {
            return false;
        }
 
        if (typeInfo.IsAbstract)
        {
            return false;
        }
 
        if (!typeInfo.IsPublic)
        {
            return false;
        }
 
        if (typeInfo.ContainsGenericParameters)
        {
            return false;
        }
 
        if (typeInfo.IsDefined(typeof(NonControllerAttribute)))
        {
            return false;
        }
 
        if (!typeInfo.Name.EndsWith(ControllerTypeNameSuffix, StringComparison.OrdinalIgnoreCase) &&
            !typeInfo.IsDefined(typeof(ControllerAttribute)))
        {
            return false;
        }
 
        return true;
    }
}
```

- ApplicationPartManager

```C#
// 应用程序组成部分管理器
// 用于管理 MVC 中多个应用程序组成部分和应用程序特性提供器
public class ApplicationPartManager
{
    // 应用程序特性提供器集合
    public IList<IApplicationFeatureProvider> FeatureProviders { get; } =
        new List<IApplicationFeatureProvider>();

    // 应用程序组成部分集合
    public IList<ApplicationPart> ApplicationParts { get; } = new List<ApplicationPart>();

    // 填充应用程序的默认组成部分
    // 本质是利用 IWebHostEnvironment.ApplicationName 表示的应用程序名称作为入口程序集名称
    internal void PopulateDefaultParts(string entryAssemblyName)
    {
        // 得到入口程序集绑定的所有 ApplicationPartAttribute 特性中指定的程序集以及所有引用的程序集
        var assemblies = GetApplicationPartAssemblies(entryAssemblyName);
 
        // 避免重复添加相同程序集
        var seenAssemblies = new HashSet<Assembly>();
 
        foreach (var assembly in assemblies)
        {
            if (!seenAssemblies.Add(assembly))
            {
                continue;
            }

            // 通过程序集创建特定的 ApplicationPartFactory
            var partFactory = ApplicationPartFactory.GetApplicationPartFactory(assembly);
            // 如果是 DefaultApplicationPartFactory 则创建的是 AssemblyPart 集合
            foreach (var applicationPart in partFactory.GetApplicationParts(assembly))
            {
                ApplicationParts.Add(applicationPart);
            }
        }
    }

    // 得到入口程序集绑定的所有 ApplicationPartAttribute 特性中指定的程序集及其引用的程序集
    // 包含如下程序集：
    // 1. 入口程序集和其引用的程序集
    // 2. 特性指定的程序集以及这些程序集引用的程序集
    private static IEnumerable<Assembly> GetApplicationPartAssemblies(string entryAssemblyName)
    {
        var entryAssembly = Assembly.Load(new AssemblyName(entryAssemblyName));
 
        var assembliesFromAttributes = entryAssembly
            .GetCustomAttributes<ApplicationPartAttribute>()
            .Select(name => Assembly.Load(name.AssemblyName))
            .OrderBy(assembly => assembly.FullName, StringComparer.Ordinal)
            .SelectMany(GetAssemblyClosure);
 
        return GetAssemblyClosure(entryAssembly)
            .Concat(assembliesFromAttributes);
    }

    // 利用提供的 TFeature 泛型实参筛选对应的 IApplicationFeatureProvider<TFeature> 配置 TFeature 特性
    public void PopulateFeature<TFeature>(TFeature feature)
    {
        if (feature == null)
        {
            throw new ArgumentNullException(nameof(feature));
        }

        // 遍历所有 ApplicationPart 填充 TFeature 特性
        foreach (var provider in FeatureProviders.OfType<IApplicationFeatureProvider<TFeature>>())
        {
            provider.PopulateFeature(ApplicationParts, feature);
        }
    }
}
```

- ApplicationPartFactory

```C#
// 表示 ApplicationPart 的抽象工厂
public abstract class ApplicationPartFactory
{
    // 子类重写该方法，实现根据给定的程序集得到 ApplicationPart 集合
    public abstract IEnumerable<ApplicationPart> GetApplicationParts(Assembly assembly);
 
    // 静态工具方法
    // 根据给定的程序集创建 ApplicationPartFactory
    public static ApplicationPartFactory GetApplicationPartFactory(Assembly assembly)
    {
        ArgumentNullException.ThrowIfNull(assembly);

        // 检查程序集是否绑定了 ProvideApplicationPartFactoryAttribute 特性
        var provideAttribute = assembly.GetCustomAttribute<ProvideApplicationPartFactoryAttribute>();
        // 如果没有则创建 DefaultApplicationPartFactory
        if (provideAttribute == null)
        {
            return DefaultApplicationPartFactory.Instance;
        }

        // 否则利用 ProvideApplicationPartFactoryAttribute.GetFactoryType 方法返回的类型反射得到对应的 ApplicationPartFactory
        var type = provideAttribute.GetFactoryType();
        if (!typeof(ApplicationPartFactory).IsAssignableFrom(type))
        {
            throw new InvalidOperationException(Resources.FormatApplicationPartFactory_InvalidFactoryType(
                type,
                nameof(ProvideApplicationPartFactoryAttribute),
                typeof(ApplicationPartFactory)));
        }
 
        return (ApplicationPartFactory)Activator.CreateInstance(type)!;
    }
}
```

- DefaultApplicationPartFactory

```C#
// 默认的 ApplicationPart 工厂
// 用于创建 AssemblyPart 封装对应的程序集
public class DefaultApplicationPartFactory : ApplicationPartFactory
{
    public static DefaultApplicationPartFactory Instance { get; } = new DefaultApplicationPartFactory();
    
    // 创建 AssemblyPart 封装对应的程序集并返回
    public static IEnumerable<ApplicationPart> GetDefaultApplicationParts(Assembly assembly)
    {
        ArgumentNullException.ThrowIfNull(assembly);
 
        yield return new AssemblyPart(assembly);
    }
 
    // 重写 GetApplicationParts 方法
    public override IEnumerable<ApplicationPart> GetApplicationParts(Assembly assembly)
    {
        return GetDefaultApplicationParts(assembly);
    }
}
```

- ActionDescriptor

```C#
// Action 方法描述符
// 注意：
// 每个 Action 方法基于特性路由可以对应多个 ActionDescriptor
public class ActionDescriptor
{
    public ActionDescriptor()
    {
        Id = Guid.NewGuid().ToString();
        Properties = new Dictionary<object, object?>();
        RouteValues = new Dictionary<string, string?>(StringComparer.OrdinalIgnoreCase);
    }
 
    public string Id { get; }
 
    // 路由值字典
    public IDictionary<string, string?> RouteValues { get; set; }
 
    // 特性路由信息
    public AttributeRouteInfo? AttributeRouteInfo { get; set; }
 
    // Action 约束集合
    public IList<IActionConstraintMetadata>? ActionConstraints { get; set; }
 
    // 终结点元数据集合
    public IList<object> EndpointMetadata { get; set; } = Array.Empty<ParameterDescriptor>();
 
    // 参数描述符集合
    public IList<ParameterDescriptor> Parameters { get; set; } = Array.Empty<ParameterDescriptor>();
 
    public IList<ParameterDescriptor> BoundProperties { get; set; } = Array.Empty<ParameterDescriptor>();
 
    // 过滤器描述符集合
    public IList<FilterDescriptor> FilterDescriptors { get; set; } = Array.Empty<FilterDescriptor>();
 
    // 显示名称
    public virtual string? DisplayName { get; set; }
 
    public IDictionary<object, object?> Properties { get; set; } = default!;
 
    internal IFilterMetadata[]? CachedReusableFilters { get; set; }
}
```

- ControllerActionDescriptor

```C#
// 表示基于控制器的 Action 方法描述符
public class ControllerActionDescriptor : ActionDescriptor
{
    public string ControllerName { get; set; } = default!;
 
    public virtual string ActionName { get; set; } = default!;
 
    public MethodInfo MethodInfo { get; set; } = default!;
 
    public TypeInfo ControllerTypeInfo { get; set; } = default!;
 
    internal EndpointFilterDelegate? FilterDelegate { get; set; }

    internal ControllerActionInvokerCacheEntry? CacheEntry { get; set; }
 
    public override string? DisplayName
    {
        get
        {
            if (base.DisplayName == null && ControllerTypeInfo != null && MethodInfo != null)
            {
                base.DisplayName = string.Format(
                    CultureInfo.InvariantCulture,
                    "{0}.{1} ({2})",
                    TypeNameHelper.GetTypeDisplayName(ControllerTypeInfo),
                    MethodInfo.Name,
                    ControllerTypeInfo.Assembly.GetName().Name);
            }
 
            return base.DisplayName!;
        }
 
        set
        {
            ArgumentNullException.ThrowIfNull(value);
 
            base.DisplayName = value;
        }
    }
}
```

- ActionDescriptorProviderContext

```C#
// 用于从多个 IActionDescriptorProvider 中收集 ActionDescriptor 时的上下文
public class ActionDescriptorProviderContext
{
    // 用于收集 ActionDescriptor 的集合
    public IList<ActionDescriptor> Results { get; } = new List<ActionDescriptor>();
}
```

- IActionDescriptorProvider

```C#
// ActionDescriptor 提供器接口
// 注意：
// 当存在多个 IActionDescriptorProvider 时会按 Order 属性值升序依次执行 OnProvidersExecuting 方法和降序执行 OnProvidersExecuted 方法
public interface IActionDescriptorProvider
{
    // 用于决定执行顺序的属性，值越小优先级越高
    int Order { get; }

    // 前处理方法
    void OnProvidersExecuting(ActionDescriptorProviderContext context);

    // 后处理方法
    void OnProvidersExecuted(ActionDescriptorProviderContext context);
}
```

- ControllerActionDescriptorProvider

```C#
// 基于控制器的 ActionDescriptor 提供器
internal sealed class ControllerActionDescriptorProvider : IActionDescriptorProvider
{
    private readonly ApplicationPartManager _partManager;
    private readonly ApplicationModelFactory _applicationModelFactory;
 
    public ControllerActionDescriptorProvider(
        ApplicationPartManager partManager,
        ApplicationModelFactory applicationModelFactory)
    {
        ArgumentNullException.ThrowIfNull(partManager);
        ArgumentNullException.ThrowIfNull(applicationModelFactory);

        _partManager = partManager;
        _applicationModelFactory = applicationModelFactory;
    }
 
    // 保证比较高的优先级
    public int Order => -1000;
 
    // 前处理方法用于创建所有的 ControllerActionDescriptor 并添加到上下文的 Results 集合中
    public void OnProvidersExecuting(ActionDescriptorProviderContext context)
    {
        ArgumentNullException.ThrowIfNull(context);
 
        foreach (var descriptor in GetDescriptors())
        {
            context.Results.Add(descriptor);
        }
    }
 
    // 后处理方法将所有 ControllerActionDescriptor 的 RouteValues 属性表示的路由值字典对齐
    public void OnProvidersExecuted(ActionDescriptorProviderContext context)
    {
        // 遍历每个 ControllerActionDescriptor 收集路由值字典的键（忽略大小写去重）
        var keys = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
        for (var i = 0; i < context.Results.Count; i++)
        {
            var action = context.Results[i];
            foreach (var key in action.RouteValues.Keys)
            {
                keys.Add(key);
            }
        }
 
        // 对齐每个 ControllerActionDescriptor 的路由值字典，如果对应的键不存在则添加一个 null 值
        // 目的是在后续创建 RouteEndpointBuilder 时为 RoutePattern 的 RequiredValues 属性赋值，用于路由系统使用 RouteEndpoint 做路径匹配或链接生成
        for (var i = 0; i < context.Results.Count; i++)
        {
            var action = context.Results[i];
            foreach (var key in keys)
            {
                if (!action.RouteValues.ContainsKey(key))
                {
                    action.RouteValues.Add(key, null);
                }
            }
        }
    }
 
    // 得到 ControllerActionDescriptor 集合
    internal IEnumerable<ControllerActionDescriptor> GetDescriptors()
    {
        // 得到所有控制器类型
        var controllerTypes = GetControllerTypes();
        // 利用 ApplicationModelFactory 解析所有控制器类型创建 ApplicationModel
        var application = _applicationModelFactory.CreateApplicationModel(controllerTypes);
        // 利用 ApplicationModel 构建 ControllerActionDescriptor 集合并返回
        // 最终 ControllerActionDescriptor 的数量取决于特性路由的几个关键概念（定义型、静默型以及是否属于 Http 方法限定）
        return ControllerActionDescriptorBuilder.Build(application);
    }
    
    // 得到所有控制器类型
    private IEnumerable<TypeInfo> GetControllerTypes()
    {
        var feature = new ControllerFeature();
        _partManager.PopulateFeature(feature);
 
        return feature.Controllers;
    }
}
```

- ApplicationModel

```C#
// 表示应用程序模型
public class ApplicationModel : IPropertyModel, IFilterModel, IApiExplorerModel
{
    public ApplicationModel()
    {
        ApiExplorer = new ApiExplorerModel();
        Controllers = new List<ControllerModel>();
        Filters = new List<IFilterMetadata>();
        Properties = new Dictionary<object, object?>();
    }
 
    // 全局 ApiExplorerModel
    // 作用于整个应用程序的所有 Action 方法，用于是否可以发现 API 以及分组
    // 可以通过 ControllerModel 或 ActionModel 的 ApiExplorer 属性覆盖
    public ApiExplorerModel ApiExplorer { get; set; }
 
    // 应用程序的 ControllerModel 集合
    // 表示应用程序内的所有控制器模型
    public IList<ControllerModel> Controllers { get; }
 
    // 全局 IFilterMetadata 集合
    // 作用于整个应用程序的所有 Action 方法，用于提供全局过滤器
    public IList<IFilterMetadata> Filters { get; }
 
    // 全局的共享字典
    // 作用于整个应用程序的所有 Action 方法
    public IDictionary<object, object?> Properties { get; }
}
```

- ControllerModel

```C#
// 控制器模型
public class ControllerModel : ICommonModel, IFilterModel, IApiExplorerModel
{
    public ControllerModel(
        TypeInfo controllerType,
        IReadOnlyList<object> attributes)
    {
        ArgumentNullException.ThrowIfNull(controllerType);
        ArgumentNullException.ThrowIfNull(attributes);
 
        ControllerType = controllerType;
 
        Actions = new List<ActionModel>();
        ApiExplorer = new ApiExplorerModel();
        Attributes = new List<object>(attributes);
        ControllerProperties = new List<PropertyModel>();
        Filters = new List<IFilterMetadata>();
        Properties = new Dictionary<object, object?>();
        RouteValues = new Dictionary<string, string?>(StringComparer.OrdinalIgnoreCase);
        Selectors = new List<SelectorModel>();
    }
    
    // 控制器模型下的所有 ActionModel 模型
    public IList<ActionModel> Actions { get; }
 
    // 控制器的 ApiExplorerModel
    public ApiExplorerModel ApiExplorer { get; set; }
    
    // 当前控制器模型所属的 ApplicationModel
    public ApplicationModel? Application { get; set; }
 
    // 绑定在当前控制器类型上的所有特性
    public IReadOnlyList<object> Attributes { get; }
 
    MemberInfo ICommonModel.MemberInfo => ControllerType;
 
    string ICommonModel.Name => ControllerName;
 
    // 控制器名称
    public string ControllerName { get; set; } = default!;
    
    // 控制器的类型
    public TypeInfo ControllerType { get; }
 
    // 控制器的 PropertyModel 集合
    public IList<PropertyModel> ControllerProperties { get; }
 
    // 控制器的 IFilterMetadata 集合
    // 从当前绑定的特性集合中过滤出实现 IFilterMetadata 的特性
    public IList<IFilterMetadata> Filters { get; }
 
    // 控制器的路由值字典
    public IDictionary<string, string?> RouteValues { get; }
 
    // 控制器的共享字典
    public IDictionary<object, object?> Properties { get; }
 
    // 控制器的 SelectorModel 集合
    // 从当前绑定的特性集合中过滤出实现 IRouteTemplateProvider 的特性
    public IList<SelectorModel> Selectors { get; }
 
    public string DisplayName
    {
        get
        {
            var controllerType = TypeNameHelper.GetTypeDisplayName(ControllerType);
            var controllerAssembly = ControllerType.Assembly.GetName().Name;
            return $"{controllerType} ({controllerAssembly})";
        }
    }
}
```

- ActionModel

```C#
// 方法模型
public class ActionModel : ICommonModel, IFilterModel, IApiExplorerModel
{
    public ActionModel(
        MethodInfo actionMethod,
        IReadOnlyList<object> attributes)
    {
        ArgumentNullException.ThrowIfNull(actionMethod);
        ArgumentNullException.ThrowIfNull(attributes);
 
        ActionMethod = actionMethod;
 
        ApiExplorer = new ApiExplorerModel();
        Attributes = new List<object>(attributes);
        Filters = new List<IFilterMetadata>();
        Parameters = new List<ParameterModel>();
        RouteValues = new Dictionary<string, string?>(StringComparer.OrdinalIgnoreCase);
        Properties = new Dictionary<object, object?>();
        Selectors = new List<SelectorModel>();
    }

    // 方法的 MethodInfo
    public MethodInfo ActionMethod { get; }
 
    // 方法名称
    public string ActionName { get; set; } = default!;
 
    // 方法的 ApiExplorerModel
    public ApiExplorerModel ApiExplorer { get; set; }
 
    // 绑定在当前方法上的所有特性
    public IReadOnlyList<object> Attributes { get; }
    
    // 当前方法模型所属的 ControllerModel
    public ControllerModel Controller { get; set; } = default!;
 
    // 方法的 IFilterMetadata 集合
    // 从当前绑定的特性集合中过滤出实现 IFilterMetadata 的特性
    public IList<IFilterMetadata> Filters { get; }
 
    // 方法的 ParameterModel 集合
    public IList<ParameterModel> Parameters { get; }
 
    public IOutboundParameterTransformer? RouteParameterTransformer { get; set; }
 
    // 方法的路由值字典
    public IDictionary<string, string?> RouteValues { get; }
 
    // 方法的共享字典
    public IDictionary<object, object?> Properties { get; }
 
    MemberInfo ICommonModel.MemberInfo => ActionMethod;
 
    string ICommonModel.Name => ActionName;
 
    // 方法的 SelectorModel 集合
    // 从当前绑定的特性集合中过滤出实现 IRouteTemplateProvider 的特性
    public IList<SelectorModel> Selectors { get; }
 
    public string DisplayName
    {
        get
        {
            if (Controller == null)
            {
                return ActionMethod.Name;
            }
 
            var controllerType = TypeNameHelper.GetTypeDisplayName(Controller.ControllerType);
            var controllerAssembly = Controller?.ControllerType.Assembly.GetName().Name;
            return $"{controllerType}.{ActionMethod.Name} ({controllerAssembly})";
        }
    }
}
```

- SelectorModel

```C#
// 选择器模型
// 表示控制器或方法上绑定的实现了 IRouteTemplateProvider 的特性
public class SelectorModel
{
    public SelectorModel()
    {
        ActionConstraints = new List<IActionConstraintMetadata>();
        EndpointMetadata = new List<object>();
    }
 
    // 特性路由模型
    // 每个 SelectorModel 都会对应一个 IRouteTemplateProvider 特性
    public AttributeRouteModel? AttributeRouteModel { get; set; }
 
    // Action 约束元数据集合
    public IList<IActionConstraintMetadata> ActionConstraints { get; }
 
    // 终结点元数据集合
    public IList<object> EndpointMetadata { get; }
}
```

- IApplicationModelConvention

```C#
// ApplicationModel 约定配置接口
public interface IApplicationModelConvention
{
    // 针对 ApplicationModel 的配置方法
    void Apply(ApplicationModel application);
}
```

- ApplicationModelProviderContext

```C#
// 用于从多个 IApplicationModelProvider 中收集 ApplicationModel 中的 ControllerModel、 ActionModel、SelectorModel、ParameterModel、PropertyModel 等模型时的上下文
public class ApplicationModelProviderContext
{
    public ApplicationModelProviderContext(IEnumerable<TypeInfo> controllerTypes)
    {
        ArgumentNullException.ThrowIfNull(controllerTypes);
 
        ControllerTypes = controllerTypes;
    }
 
    // 用于构建应用程序模型的控制器类型集合
    public IEnumerable<TypeInfo> ControllerTypes { get; }
 
    // 应用程序模型
    public ApplicationModel Result { get; } = new ApplicationModel();
}
```

- IApplicationModelProvider

```C#
// 应用模型提供器接口
// 注意：
// 当存在多个 IApplicationModelProvider 时会按 Order 属性值升序依次执行 OnProvidersExecuting 方法和降序执行 OnProvidersExecuted 方法
public interface IApplicationModelProvider
{
    // 用于决定执行顺序的属性，值越小优先级越高
    int Order { get; }
 
    // 前处理方法
    void OnProvidersExecuting(ApplicationModelProviderContext context);
 
    // 后处理方法
    void OnProvidersExecuted(ApplicationModelProviderContext context);
}
```

- DefaultApplicationModelProvider

```C#
// 默认的 ApplicationModel 提供器
// 用于创建 ApplicationModel 中的 ControllerModel、ActionModel、SelectorModel、ParameterModel、PropertyModel 等模型
internal class DefaultApplicationModelProvider : IApplicationModelProvider
{
    private readonly MvcOptions _mvcOptions;
    private readonly IModelMetadataProvider _modelMetadataProvider;

    public DefaultApplicationModelProvider(
        IOptions<MvcOptions> mvcOptionsAccessor,
        IModelMetadataProvider modelMetadataProvider)
    {
        _mvcOptions = mvcOptionsAccessor.Value;
        _modelMetadataProvider = modelMetadataProvider;
 
        _supportsAllRequests = _ => true;
        _supportsNonGetRequests = context => !HttpMethods.IsGet(context.HttpContext.Request.Method);
    }

    // 保证比较高的优先级
    public int Order => -1000;

    // 前处理方法
    // 为 ApplicationModel 创建 ControllerModel、ActionModel、SelectorModel、ParameterModel、PropertyModel 等模型
    public void OnProvidersExecuting(ApplicationModelProviderContext context)
    {
        ArgumentNullException.ThrowIfNull(context);

        // 从 MvcOptions.Filters 属性中将过滤器（实现 IFilterMetadata）转移到 ApplicationModel.Filters 集合中
        // 这些就是作用于整个应用程序中所有 Action 的全局过滤器
        // 注意：
        // MvcOptions.Filters 属性在添加过滤器类型时，实际使用 TypeFilterAttribute （实现 IFilterMetadata）这个特性封装实际的过滤器类型（同样实现 IFilterMetadata）
        // TypeFilterAttribute 实现了 IFilterFactory 接口，实际的过滤器就是通过实现的 CreateInstance 方法创建的，并且 IFilterFactory.CreateInstance 方法接受一个 IServiceProvider 类型的参数，可以在创建实际过滤器时提供依赖服务的解析
        foreach (var filter in _mvcOptions.Filters)
        {
            context.Result.Filters.Add(filter);
        }
 
        // 从控制器类型创建对应的 ControllerModel 并添加到 ApplicationModel.Controllers 集合中
        foreach (var controllerType in context.ControllerTypes)
        {
            var controllerModel = CreateControllerModel(controllerType);
            if (controllerModel == null)
            {
                continue;
            }
 
            context.Result.Controllers.Add(controllerModel);
            controllerModel.Application = context.Result;

            // 创建 PropertyModel 并添加到 ControllerModel.ControllerProperties 集合中
            foreach (var propertyHelper in PropertyHelper.GetProperties(controllerType.AsType()))
            {
                var propertyInfo = propertyHelper.Property;
                var propertyModel = CreatePropertyModel(propertyInfo);
                if (propertyModel != null)
                {
                    propertyModel.Controller = controllerModel;
                    controllerModel.ControllerProperties.Add(propertyModel);
                }
            }
 
            //  创建 ActionModel 并添加到 ControllerModel.Actions 集合中
            foreach (var methodInfo in controllerType.AsType().GetMethods())
            {
                var actionModel = CreateActionModel(controllerType, methodInfo);
                if (actionModel == null)
                {
                    continue;
                }
 
                actionModel.Controller = controllerModel;
                controllerModel.Actions.Add(actionModel);
 
                //  创建 ParameterModel 并添加到 ActionModel.Parameters 集合中
                foreach (var parameterInfo in actionModel.ActionMethod.GetParameters())
                {
                    var parameterModel = CreateParameterModel(parameterInfo);
                    if (parameterModel != null)
                    {
                        parameterModel.Action = actionModel;
                        actionModel.Parameters.Add(parameterModel);
                    }
                }
            }
        }
    }

    // 后处理方法
    public void OnProvidersExecuted(ApplicationModelProviderContext context)
    {
        // 不做任何处理
    }

    // 创建 ControllerModel
    internal static ControllerModel CreateControllerModel(TypeInfo typeInfo)
    {
        ArgumentNullException.ThrowIfNull(typeInfo);
 
        var currentTypeInfo = typeInfo;
        var objectTypeInfo = typeof(object).GetTypeInfo();
 
        IRouteTemplateProvider[] routeAttributes;
 
        do
        {
            // 得到控制器上绑定的实现了 IRouteTemplateProvider 的特性
            // 注意：
            // 这里不从基类继承特性
            routeAttributes = currentTypeInfo
                .GetCustomAttributes(inherit: false)
                .OfType<IRouteTemplateProvider>()
                .ToArray();

            // 存在则中断
            if (routeAttributes.Length > 0)
            {
                break;
            }

            // 如果没有找到则继续检查基类
            // 直到找到 object 类型为止
            currentTypeInfo = currentTypeInfo.BaseType!.GetTypeInfo();
        }
        while (currentTypeInfo != objectTypeInfo);

        // 得到控制器上绑定的所有的特性（包括基类上的）
        var attributes = typeInfo.GetCustomAttributes(inherit: true);
 
        var filteredAttributes = new List<object>();
        foreach (var attribute in attributes)
        {
            if (attribute is IRouteTemplateProvider)
            {
                // 排除实现了 IRouteTemplateProvider 的特性
            }
            else
            {
                // 只添加其他特性
                filteredAttributes.Add(attribute);
            }
        }
        
        // 合并特性，将收集的实现 IRouteTemplateProvider 的特性添加到集合末尾
        filteredAttributes.AddRange(routeAttributes);
 
        attributes = filteredAttributes.ToArray();

        // 利用控制器类型和特性集合创建 ControllerModel
        var controllerModel = new ControllerModel(typeInfo, attributes);

        // 根据 ControllerModel 中绑定的实现 IRouteTemplateProvider 的特性创建 SelectorModel，并添加到 ControllerModel.Selectors 集合中
        AddRange(controllerModel.Selectors, CreateSelectors(attributes));
    
        controllerModel.ControllerName =
            typeInfo.Name.EndsWith("Controller", StringComparison.OrdinalIgnoreCase) ?
                typeInfo.Name.Substring(0, typeInfo.Name.Length - "Controller".Length) :
                typeInfo.Name;
 
        // 根据 ControllerModel 中绑定的实现 IFilterMetadata 的特性创建 FilterModel，并添加到 ControllerModel.Filters 集合中
        AddRange(controllerModel.Filters, attributes.OfType<IFilterMetadata>());

        // 根据 ControllerModel 中绑定的实现 IRouteValueProvider 的特性填充路由值字典
        foreach (var routeValueProvider in attributes.OfType<IRouteValueProvider>())
        {
            controllerModel.RouteValues.Add(routeValueProvider.RouteKey, routeValueProvider.RouteValue);
        }

        // 根据 ControllerModel 中绑定的实现 IApiDescriptionVisibilityProvider 的特性配置控制器的可见性（针对控制器下所有方法）
        var apiVisibility = attributes.OfType<IApiDescriptionVisibilityProvider>().FirstOrDefault();
        if (apiVisibility != null)
        {
            controllerModel.ApiExplorer.IsVisible = !apiVisibility.IgnoreApi;
        }
 
        // 根据 ControllerModel 中绑定的实现 IApiDescriptionGroupNameProvider 的特性配置控制器的分组名称
        var apiGroupName = attributes.OfType<IApiDescriptionGroupNameProvider>().FirstOrDefault();
        if (apiGroupName != null)
        {
            controllerModel.ApiExplorer.GroupName = apiGroupName.GroupName;
        }
 
        // 如果控制器类型自身实现了 IAsyncActionFilter 或 IActionFilter，则创建 ControllerActionFilter 作为 IFileterMetadata 添加到 ControllerModel.Filters 集合中
        // ControllerActionFilter 作为过滤器时会在管线执行时，利用控制器调用过滤器方法
        if (typeof(IAsyncActionFilter).GetTypeInfo().IsAssignableFrom(typeInfo) ||
            typeof(IActionFilter).GetTypeInfo().IsAssignableFrom(typeInfo))
        {
            controllerModel.Filters.Add(new ControllerActionFilter());
        }
        // 如果控制器类型自身实现了 IAsyncResultFilter 或 IResultFilter，则创建 ControllerResultFilter 作为 IFileterMetadata 添加到 ControllerModel.Filters 集合中
        // ControllerResultFilter 作为过滤器时会在管线执行时，利用控制器调用过滤器方法
        if (typeof(IAsyncResultFilter).GetTypeInfo().IsAssignableFrom(typeInfo) ||
            typeof(IResultFilter).GetTypeInfo().IsAssignableFrom(typeInfo))
        {
            controllerModel.Filters.Add(new ControllerResultFilter());
        }
 
        return controllerModel;
    }

    // 创建 ActionModel
    internal ActionModel? CreateActionModel(
        TypeInfo typeInfo,
        MethodInfo methodInfo)
    {
        ArgumentNullException.ThrowIfNull(typeInfo);
        ArgumentNullException.ThrowIfNull(methodInfo);
 
        // 选定的方法需要同时满足的条件
        // 1. 不能是编译器生成的特殊方法名，比如属性的 get/set 方法
        // 2. 方法不能绑定 NonActionAttribute 特性
        // 3. 重写的方法不能是 object 类型中定义的 virtual 方法
        // 4. 不能是实现了 IDisposable 接口的 Dispose 方法
        // 5. 不能是静态方法
        // 6. 不能是抽象方法
        // 7. 不能是构造函数
        // 8. 不能是泛型方法
        if (!IsAction(typeInfo, methodInfo))
        {
            return null;
        }
 
        // 得到方法上绑定的所有特性（包括基类被重写方法继承的特性）
        var attributes = methodInfo.GetCustomAttributes(inherit: true);

        // 利用 MethodInfo 和特性集合创建 ActionModel
        var actionModel = new ActionModel(methodInfo, attributes);
 
        // 根据 Action 方法上绑定的实现 IFilterMetadata 的特性创建 FilterModel，并添加到 ActionModel.Filters 集合中
        AddRange(actionModel.Filters, attributes.OfType<IFilterMetadata>());
 
        var actionName = attributes.OfType<ActionNameAttribute>().FirstOrDefault();
        if (actionName?.Name != null)
        {
            actionModel.ActionName = actionName.Name;
        }
        else
        {
            actionModel.ActionName = CanonicalizeActionName(methodInfo.Name);
        }
 
        // 根据 ActionModel 中绑定的实现 IApiDescriptionVisibilityProvider 的特性配置方法的可见性
        var apiVisibility = attributes.OfType<IApiDescriptionVisibilityProvider>().FirstOrDefault();
        if (apiVisibility != null)
        {
            actionModel.ApiExplorer.IsVisible = !apiVisibility.IgnoreApi;
        }
 
        // 根据 ActionModel 中绑定的实现 IApiDescriptionGroupNameProvider 的特性配置方法的分组名称
        var apiGroupName = attributes.OfType<IApiDescriptionGroupNameProvider>().FirstOrDefault();
        if (apiGroupName != null)
        {
            actionModel.ApiExplorer.GroupName = apiGroupName.GroupName;
        }
 
        // 根据 ActionModel 中绑定的实现 IRouteValueProvider 的特性填充路由值字典
        foreach (var routeValueProvider in attributes.OfType<IRouteValueProvider>())
        {
            actionModel.RouteValues.Add(routeValueProvider.RouteKey, routeValueProvider.RouteValue);
        }
 
        var currentMethodInfo = methodInfo;
 
        IRouteTemplateProvider[] routeAttributes;
 
        // 从方法上收集实现了 IRouteTemplateProvider 的特性
        while (true)
        {
            // 注意：
            // 这里不从基类被重写方法继承特性
            routeAttributes = currentMethodInfo
                .GetCustomAttributes(inherit: false)
                .OfType<IRouteTemplateProvider>()
                .ToArray();

            // 存在则中断
            if (routeAttributes.Length > 0)
            {
                break;
            }

            // 继续检查基类中被重写的方法
            var nextMethodInfo = currentMethodInfo.GetBaseDefinition();
            if (currentMethodInfo == nextMethodInfo)
            {
                break;
            }

            currentMethodInfo = nextMethodInfo;
        }
 
        var applicableAttributes = new List<object>(routeAttributes.Length);
        foreach (var attribute in attributes)
        {
            if (attribute is IRouteTemplateProvider)
            {
                // 排除实现了 IRouteTemplateProvider 的特性
            }
            else
            {
                // 只添加其他特性
                applicableAttributes.Add(attribute);
            }
        }
 
        // 合并特性，将收集的实现 IRouteTemplateProvider 的特性添加到集合末尾
        applicableAttributes.AddRange(routeAttributes);
        // 根据 ActionModel 中绑定的实现 IRouteTemplateProvider 的特性创建 SelectorModel，并添加到 ActionModel.Selectors 集合中
        AddRange(actionModel.Selectors, CreateSelectors(applicableAttributes));

        // 添加返回值类型的元数据到 ActionModel.Selectors 集合中的每个 SelectorModel 中的 EndpointMetadata 集合中
        AddReturnTypeMetadata(actionModel.Selectors, methodInfo);
 
        return actionModel;
    }

    // 创建 ParameterModel
    internal ParameterModel CreateParameterModel(ParameterInfo parameterInfo)
    {
        ArgumentNullException.ThrowIfNull(parameterInfo);
 
        // 得到参数上绑定的所有特性（包括基类被重写方法参数继承的特性）
        var attributes = parameterInfo.GetCustomAttributes(inherit: true);
 
        // 遍历所有特性解析参数的绑定信息确定绑定源
        // BindingInfo 优先读取特性列表中的绑定信息，再从 ModelMetadata 获取绑定信息作为补充
        // ModelMetadata 通过实现 IModelMetadataProvider 的 DefaultModelMetadataProvider 创建 DefaultModelMetadata
        // DefaultModelMetadata 内部提供绑定信息依赖 BindingMetadata
        // BindingMetadata 通过实现 ICompositeMetadataDetailsProvider 的 DefaultCompositeMetadataDetailsProvider 调用 CreateBindingMetadata 方法创建
        // CreateBindingMetadata 方法会遍历所有注入的 IBindingMetadataProvider（实现 IMetadataDetailsProvider） 来创建 BindingMetadata
        // 所以可以通过自定义 IBindingMetadataProvider 来定制参数的绑定信息
        BindingInfo? bindingInfo;
        if (_modelMetadataProvider is ModelMetadataProvider modelMetadataProviderBase)
        {
            var modelMetadata = modelMetadataProviderBase.GetMetadataForParameter(parameterInfo);
            bindingInfo = BindingInfo.GetBindingInfo(attributes, modelMetadata);
        }
        else
        {
            bindingInfo = BindingInfo.GetBindingInfo(attributes);
        }

        // 利用 ParameterInfo、特性集合、绑定信息创建 ParameterModel
        var parameterModel = new ParameterModel(parameterInfo, attributes)
        {
            ParameterName = parameterInfo.Name!,
            BindingInfo = bindingInfo
        };
 
        return parameterModel;
    }

    // 创建 SelectorModel 集合
    // 具体创建多少个 SelectorModel 遵循以下规则：
    // 1. 特性路由需要是实现 IRouteTemplateProvider 的特性
    // 2. 特性路由分为定义型和静默型两种以及是否有 Http 方法限定（试下 IActionHttpMethodProvider）
    // 3. 定义型并且有 Http 方法限定的，每个特性创建一个 SelectorModel
    // 4. 静默型并且有 Http 方法限定的，首先合并所有特性
    // 5. 如果存在定义型并且没有 Http 方法限定的，则和合并后的特性一起创建一个 SelectorModel
    // 6. 否则使用方法名作为默认路由模板和合并后的特性一起创建一个 SelectorModel
    private static IList<SelectorModel> CreateSelectors(IList<object> attributes)
    {
        var routeProviders = new List<IRouteTemplateProvider>();
 
        var createSelectorForSilentRouteProviders = false;
        foreach (var attribute in attributes)
        {
            if (attribute is IRouteTemplateProvider routeTemplateProvider)
            {
                if (IsSilentRouteAttribute(routeTemplateProvider))
                {
                    createSelectorForSilentRouteProviders = true;
                }
                else
                {
                    routeProviders.Add(routeTemplateProvider);
                }
            }
        }
 
        foreach (var routeProvider in routeProviders)
        {
            if (!(routeProvider is IActionHttpMethodProvider))
            {
                createSelectorForSilentRouteProviders = false;
                break;
            }
        }
 
        var selectorModels = new List<SelectorModel>();
        if (routeProviders.Count == 0 && !createSelectorForSilentRouteProviders)
        {
            selectorModels.Add(CreateSelectorModel(route: null, attributes: attributes));
        }
        else
        {
            foreach (var routeProvider in routeProviders)
            {
                var filteredAttributes = new List<object>();
                foreach (var attribute in attributes)
                {
                    if (ReferenceEquals(attribute, routeProvider))
                    {
                        filteredAttributes.Add(attribute);
                    }
                    else if (InRouteProviders(routeProviders, attribute))
                    {
                    }
                    else if (
                        routeProvider is IActionHttpMethodProvider &&
                        attribute is IActionHttpMethodProvider)
                    {
                    }
                    else
                    {
                        filteredAttributes.Add(attribute);
                    }
                }
 
                selectorModels.Add(CreateSelectorModel(routeProvider, filteredAttributes));
            }
 
            if (createSelectorForSilentRouteProviders)
            {
                var filteredAttributes = new List<object>();
                foreach (var attribute in attributes)
                {
                    if (!InRouteProviders(routeProviders, attribute))
                    {
                        filteredAttributes.Add(attribute);
                    }
                }
 
                selectorModels.Add(CreateSelectorModel(route: null, attributes: filteredAttributes));
            }
        }
 
        return selectorModels;
    }

    // 创建 SelectorModel
    private static SelectorModel CreateSelectorModel(IRouteTemplateProvider? route, IList<object> attributes)
    {
        var selectorModel = new SelectorModel();
        if (route != null)
        {
            selectorModel.AttributeRouteModel = new AttributeRouteModel(route);
        }
 
        // 将特性集合中实现了 IActionConstraintMetadata 的特性添加到 SelectorModel.ActionConstraints 集合中
        AddRange(selectorModel.ActionConstraints, attributes.OfType<IActionConstraintMetadata>());
        // 将特性集合添加到 SelectorModel.EndpointMetadata 表示的终结点元数据集合中
        AddRange(selectorModel.EndpointMetadata, attributes);
 
        // 检查是否存在实现了 IActionHttpMethodProvider 的特性
        var httpMethods = attributes
            .OfType<IActionHttpMethodProvider>()
            .SelectMany(a => a.HttpMethods)
            .Distinct(StringComparer.OrdinalIgnoreCase)
            .ToArray();
 
        if (httpMethods.Length > 0)
        {
            // 添加 HttpMethodActionConstraint 到 SelectorModel.ActionConstraints 集合中
            selectorModel.ActionConstraints.Add(new HttpMethodActionConstraint(httpMethods));
            // 添加 HttpMethodMetadata 到 SelectorModel.EndpointMetadata 集合中
            selectorModel.EndpointMetadata.Add(new HttpMethodMetadata(httpMethods));
        }
 
        return selectorModel;
    }
}
```

- ApiBehaviorApplicationModelProvider

```C#
// 基于 Api 行为的 ApplicationModel 提供器
// 针对绑定了 ApiControllerAttribute 特性的控制器
// 注意：
// 标记了 ApiControllerAttribute 特性的控制器的所有 Action 方法必须使用特性路由
// 即在类型或方法上绑定实现了 IRouteTemplateProvider 的特性
internal sealed class ApiBehaviorApplicationModelProvider : IApplicationModelProvider
{
    public ApiBehaviorApplicationModelProvider(
        IOptions<ApiBehaviorOptions> apiBehaviorOptions,
        IModelMetadataProvider modelMetadataProvider,
        IServiceProvider serviceProvider)
    {
        var options = apiBehaviorOptions.Value;
 
        ActionModelConventions = new List<IActionModelConvention>()
        {
            // 用于配置默认的 Action 可见
            // 注意：
            // 只有在控制器和方法都没有绑定 IApiDescriptionVisibilityProvider 特性的情况下生效
            new ApiVisibilityConvention(),
        };
 
        if (!options.SuppressMapClientErrors)
        {
            // 为每个 ActionModel 的 Action.Filters 集合添加 ClientErrorResultFilterFactory 并最终创建 ClientErrorResultFilter 结果过滤器来处理 4xx 状态码的友好响应
            ActionModelConventions.Add(new ClientErrorResultFilterConvention());
        }
 
        if (!options.SuppressModelStateInvalidFilter)
        {
            // 为每个 ActionModel 的 Action.Filters 集合添加 ModelStateInvalidFilterFactory 并最终创建 ModelStateInvalidFilter 动作过滤器来处理模型验证失败的情况
            ActionModelConventions.Add(new InvalidModelStateFilterConvention());
        }
 
        if (!options.SuppressConsumesConstraintForFormFileParameters)
        {
            // 为每个包含 IFormFile 或 IFormFileCollection 参数的 ActionModel 添加 ConsumesAttribute 资源过滤器，用来检查请求的 Content-Type 是否为 multipart/form-data 媒体类型，如果不是则返回 415 状态码
            ActionModelConventions.Add(new ConsumesConstraintForFormFileParameterConvention());
        }
 
        // 错误响应类型配置
        var defaultErrorType = options.SuppressMapClientErrors ? typeof(void) : typeof(ProblemDetails);
        var defaultErrorTypeAttribute = new ProducesErrorResponseTypeAttribute(defaultErrorType);
        ActionModelConventions.Add(new ApiConventionApplicationModelConvention(defaultErrorTypeAttribute));
 
        // 检查方法参数上的绑定源是否配置，如果没有则配置默认的模型绑定源
        if (!options.SuppressInferBindingSourcesForParameters)
        {
            var serviceProviderIsService = serviceProvider.GetService<IServiceProviderIsService>();
            var convention = options.DisableImplicitFromServicesParameters || serviceProviderIsService is null ?
                new InferParameterBindingInfoConvention(modelMetadataProvider) :
                new InferParameterBindingInfoConvention(modelMetadataProvider, serviceProviderIsService);
            ActionModelConventions.Add(convention);
        }
    }
 
    // +100 的目的是降低优先级，保证 ApiBehaviorApplicationModelProvider 在 DefaultApplicationModelProvider 之后执行
    // 因为需要等待 DefaultApplicationModelProvider 创建好所有模型之后才能使用预添加的 IActionModelConvention 对 ActionModel 进行配置
    public int Order => -1000 + 100;
 
    public List<IActionModelConvention> ActionModelConventions { get; }
 
    // 前处理方法
    public void OnProvidersExecuting(ApplicationModelProviderContext context)
    {
        // 遍历 ApplicationModel.Controllers 集合
        foreach (var controller in context.Result.Controllers)
        {
            // 跳过没有绑定 ApiControllerAttribute 特性的控制器
            // ApiBehaviorApplicationModelProvider 是提供给 WebApi 控制器的专属配置
            if (!IsApiController(controller))
            {
                continue;
            }
 
            foreach (var action in controller.Actions)
            {
                // 控制器或方法上必须确保配置了特性路由
                EnsureActionIsAttributeRouted(action);

                // 遍历 IActionModelConventions 集合配置 ActionModel
                foreach (var convention in ActionModelConventions)
                {
                    convention.Apply(action);
                }
            }
        }
    }

    // 后处理方法
    public void OnProvidersExecuted(ApplicationModelProviderContext context)
    {
        // 不做任何处理
    }
    
    // 确保控制器或方法上配置了特性路由
    private static void EnsureActionIsAttributeRouted(ActionModel actionModel)
    {
        if (!IsAttributeRouted(actionModel.Controller.Selectors) &&
            !IsAttributeRouted(actionModel.Selectors))
        {
            var message = Resources.FormatApiController_AttributeRouteRequired(
                 actionModel.DisplayName,
                nameof(ApiControllerAttribute));
            throw new InvalidOperationException(message);
        }
 
        static bool IsAttributeRouted(IList<SelectorModel> selectorModel)
        {
            for (var i = 0; i < selectorModel.Count; i++)
            {
                if (selectorModel[i].AttributeRouteModel != null)
                {
                    return true;
                }
            }
 
            return false;
        }
    }
    
    // 检查控制器类型上是否绑定实现了 IApiBehaviorMetadata 的特性
    // 比如 ApiControllerAttribute 就实现了 IApiBehaviorMetadata
    private static bool IsApiController(ControllerModel controller)
    {
        if (controller.Attributes.OfType<IApiBehaviorMetadata>().Any())
        {
            return true;
        }

        // 或者从类型定义所在的程序集上检查是否绑定了实现 IApiBehaviorMetadata 的特性
        var controllerAssembly = controller.ControllerType.Assembly;
        var assemblyAttributes = controllerAssembly.GetCustomAttributes();
        return assemblyAttributes.OfType<IApiBehaviorMetadata>().Any();
    }
}
```

- ApplicationModelFactory

```C#
// ApplicationModel 工厂
internal sealed class ApplicationModelFactory
{
    private readonly IApplicationModelProvider[] _applicationModelProviders;
    private readonly IList<IApplicationModelConvention> _conventions;
 
    public ApplicationModelFactory(
        IEnumerable<IApplicationModelProvider> applicationModelProviders,
        IOptions<MvcOptions> options)
    {
        ArgumentNullException.ThrowIfNull(applicationModelProviders);
        ArgumentNullException.ThrowIfNull(options);
 
        // 对注入的 IApplicationModelProvider 集合升序排序
        _applicationModelProviders = applicationModelProviders.OrderBy(p => p.Order).ToArray();
        
        // 从 MvcOptions.Conventions 属性中得到 IApplicationModelConvention 集合
        _conventions = options.Value.Conventions;
    }

    // 应用 IApplicationModelProvider 集合配置 ApplicationModel
    public ApplicationModel CreateApplicationModel(IEnumerable<TypeInfo> controllerTypes)
    {
        ArgumentNullException.ThrowIfNull(controllerTypes);
 
        var context = new ApplicationModelProviderContext(controllerTypes);
 
        for (var i = 0; i < _applicationModelProviders.Length; i++)
        {
            _applicationModelProviders[i].OnProvidersExecuting(context);
        }

        for (var i = _applicationModelProviders.Length - 1; i >= 0; i--)
        {
            _applicationModelProviders[i].OnProvidersExecuted(context);
        }
 
        // 应用 IApplicationModelConvention 集合配置 ApplicationModel
        ApplicationModelConventions.ApplyConventions(context.Result, _conventions);
 
        return context.Result;
    }
}
```

- IActionDescriptorChangeProvider

```C#
// 与 IActionDescriptorProvider 配套的变动令牌提供器接口
// 用于在 IActionDescriptorCollectionProvider 保存的 ActionDescriptor 集合变化时发出通知
public interface IActionDescriptorChangeProvider
{
    // 返回变动令牌，用于注册回调，在发生变动时触发
    IChangeToken GetChangeToken();
}
```

- ActionDescriptorChangeProvider

```C#
// IActionDescriptorChangeProvider 的默认实现
public class ActionDescriptorChangeProvider : IActionDescriptorChangeProvider
{
    public ActionDescriptorChangeProvider(WellKnownChangeToken changeToken)
    {
        ChangeToken = changeToken;
    }
 
    public WellKnownChangeToken ChangeToken { get; }
 
    public IChangeToken GetChangeToken()
    {
        if (ChangeToken.TokenSource.IsCancellationRequested)
        {
            var changeTokenSource = new CancellationTokenSource();
            return new CancellationChangeToken(changeTokenSource.Token);
        }
 
        return new CancellationChangeToken(ChangeToken.TokenSource.Token);
    }
}
```

- IActionDescriptorCollectionProvider

```C#
// IActionDescriptorProvider 集合提供器接口
public interface IActionDescriptorCollectionProvider
{
    // 聚合多个 IActionDescriptorProvider 提供的 ActionDescriptor 集合
    ActionDescriptorCollection ActionDescriptors { get; }
}
```

- ActionDescriptorCollectionProvider

```C#
// IActionDescriptorCollectionProvider 的默认实现
public abstract class ActionDescriptorCollectionProvider : IActionDescriptorCollectionProvider
{
    // ActionDescriptor 集合
    public abstract ActionDescriptorCollection ActionDescriptors { get; }
 
    // 返回变动令牌，用于回调注册，在发生变动时触发
    public abstract IChangeToken GetChangeToken();
}
```

- DefaultActionDescriptorCollectionProvider

```C#
// 继承 ActionDescriptorCollectionProvider
internal sealed partial class DefaultActionDescriptorCollectionProvider : ActionDescriptorCollectionProvider
{
    private readonly IActionDescriptorProvider[] _actionDescriptorProviders;
    private readonly IActionDescriptorChangeProvider[] _actionDescriptorChangeProviders;
    private readonly ILogger _logger;
 
    private readonly object _lock;
    private ActionDescriptorCollection? _collection;
    private IChangeToken? _changeToken;
    private CancellationTokenSource? _cancellationTokenSource;
    private int _version;
 
    public DefaultActionDescriptorCollectionProvider(
        IEnumerable<IActionDescriptorProvider> actionDescriptorProviders,
        IEnumerable<IActionDescriptorChangeProvider> actionDescriptorChangeProviders,
        ILogger<DefaultActionDescriptorCollectionProvider> logger)
    {
        // 对注入的 IActionDescriptorProvider 集合升序排序
        _actionDescriptorProviders = actionDescriptorProviders
            .OrderBy(p => p.Order)
            .ToArray();
 
        _actionDescriptorChangeProviders = actionDescriptorChangeProviders.ToArray();
 
        _lock = new object();
 
        _logger = logger;
 
        // 合并多个 IChangeToken 封装为 CompositeChangeToken 并注册回调 UpdateCollection 方法
        // 用于更新 ActionDescriptor 集合
        ChangeToken.OnChange(
            GetCompositeChangeToken,
            UpdateCollection);
    }
 
    // 合并多个 IActionDescriptorProvider 提供的 ActionDescriptor 集合
    public override ActionDescriptorCollection ActionDescriptors
    {
        get
        {
            // 初始化 ActionDescriptor 集合
            Initialize();
            Debug.Assert(_collection != null);
            Debug.Assert(_changeToken != null);
 
            return _collection;
        }
    }
 
    // 返回变动令牌，用于回调注册，在 ActionDescriptor 集合发生变动时触发
    public override IChangeToken GetChangeToken()
    {
        // 初始化 ActionDescriptor 集合
        Initialize();
        Debug.Assert(_collection != null);
        Debug.Assert(_changeToken != null);
 
        return _changeToken;
    }
 
    // 合并多个 IActionDescriptorChangeProvider 提供的 IChangeToken 封装为 CompositeChangeToken
    private IChangeToken GetCompositeChangeToken()
    {
        if (_actionDescriptorChangeProviders.Length == 1)
        {
            return _actionDescriptorChangeProviders[0].GetChangeToken();
        }
 
        var changeTokens = new IChangeToken[_actionDescriptorChangeProviders.Length];
        for (var i = 0; i < _actionDescriptorChangeProviders.Length; i++)
        {
            changeTokens[i] = _actionDescriptorChangeProviders[i].GetChangeToken();
        }
 
        return new CompositeChangeToken(changeTokens);
    }
 
    // 初始化 ActionDescriptor 集合
    private void Initialize()
    {
        // 需要保证线程同步，双重检查锁定
        // 初始化只会调用 UpdateCollection 方法一次，用于创建 ActionDescriptor 集合，其他时候都是通过触发变动令牌回调来更新集合
        if (_collection == null)
        {
            lock (_lock)
            {
                if (_collection == null)
                {
                    UpdateCollection();
                }
            }
        }
    }
    
    // 更新 ActionDescriptor 集合
    private void UpdateCollection()
    {
        lock (_lock)
        {
            // 创建上下文
            var context = new ActionDescriptorProviderContext();
 
            // 升序执行每个 IActionDescriptorProvider 的前处理方法
            for (var i = 0; i < _actionDescriptorProviders.Length; i++)
            {
                _actionDescriptorProviders[i].OnProvidersExecuting(context);
            }

            // 降序执行每个 IActionDescriptorProvider 的后处理方法
            for (var i = _actionDescriptorProviders.Length - 1; i >= 0; i--)
            {
                _actionDescriptorProviders[i].OnProvidersExecuted(context);
            }
 
            if (context.Results.Count == 0)
            {
                Log.NoActionDescriptors(_logger);
            }

            var oldCancellationTokenSource = _cancellationTokenSource;

            // 重新创建 ActionDescriptorCollection
            _collection = new ActionDescriptorCollection(
                new ReadOnlyCollection<ActionDescriptor>(context.Results),
                _version++);

            // 利用 CancellationTokenSource 创建一个新的 CancellationChangeToken
            _cancellationTokenSource = new CancellationTokenSource();
            _changeToken = new CancellationChangeToken(_cancellationTokenSource.Token);

            // 利用旧的 CancellationChangeToken 触发回调
            oldCancellationTokenSource?.Cancel();
        }
    }
 
    public static partial class Log
    {
        [LoggerMessage(
            EventId = 1,
            EventName = "NoActionDescriptors",
            Level = LogLevel.Information,
            Message = "No action descriptors found. This may indicate an incorrectly configured application or missing application parts. To learn more, visit https://aka.ms/aspnet/mvc/app-parts")]
        public static partial void NoActionDescriptors(ILogger logger);
    }
}
```

## 终结点

- ControllerActionEndpointConventionBuilder

```C#
// 针对控制器 ActionDescriptor 构建终结点时的 RouteEndpointBuilder 约定配置类
// 是与 ControllerActionEndpointDataSource 配套的 IEndpointConventionBuilder 实现
public sealed class ControllerActionEndpointConventionBuilder : IEndpointConventionBuilder
{
    private readonly object _lock;
    private readonly List<Action<EndpointBuilder>> _conventions;
    private readonly List<Action<EndpointBuilder>> _finallyConventions;
 
    // 注意：
    // 两个配置集合都是来自于外部
    internal ControllerActionEndpointConventionBuilder(object @lock, List<Action<EndpointBuilder>> conventions, List<Action<EndpointBuilder>> finallyConventions)
    {
        _lock = @lock;
        _conventions = conventions;
        _finallyConventions = finallyConventions;
    }
 
    public void Add(Action<EndpointBuilder> convention)
    {
        ArgumentNullException.ThrowIfNull(convention);
 
        lock (_lock)
        {
            _conventions.Add(convention);
        }
    }
 
    public void Finally(Action<EndpointBuilder> finalConvention)
    {
        ArgumentNullException.ThrowIfNull(nameof(finalConvention));
 
        lock (_lock)
        {
            _finallyConventions.Add(finalConvention);
        };
    }
}
```

- ActionEndpointDataSourceBase

```C#
// 针对 ActionDescriptor 的 EndpointDataSource 基类
internal abstract class ActionEndpointDataSourceBase : EndpointDataSource, IDisposable
{
    private readonly IActionDescriptorCollectionProvider _actions;
 
    protected readonly object Lock = new object();
 
    protected readonly List<Action<EndpointBuilder>> Conventions;
    protected readonly List<Action<EndpointBuilder>> FinallyConventions;
 
    private List<Endpoint>? _endpoints;
    private CancellationTokenSource? _cancellationTokenSource;
    private IChangeToken? _changeToken;
    private IDisposable? _disposable;
 
    public ActionEndpointDataSourceBase(IActionDescriptorCollectionProvider actions)
    {
        // 注入 IActionDescriptorCollectionProvider 服务
        // 默认的实现类型是 DefaultActionDescriptorCollectionProvider
        _actions = actions;
 
        // 注意：
        // 这两个配置集合就是提供给 ControllerActionEndpointConventionBuilder 的外部集合
        Conventions = new List<Action<EndpointBuilder>>();
        FinallyConventions = new List<Action<EndpointBuilder>>();
    }
 
    // 返回终结点集合
    public override IReadOnlyList<Endpoint> Endpoints
    {
        get
        {
            // 初始化终结点集合
            Initialize();
            Debug.Assert(_changeToken != null);
            Debug.Assert(_endpoints != null);
            return _endpoints;
        }
    }
 
    // 创建终结点集合
    // 不同的 ActionEndpointDataSource 实现类会有不同的实现方式
    // 比如 ControllerActionEndpointDataSource 会利用 ControllerActionDescriptor 创建终结点
    protected abstract List<Endpoint> CreateEndpoints(
        RoutePattern? groupPrefix,
        IReadOnlyList<ActionDescriptor> actions,
        IReadOnlyList<Action<EndpointBuilder>> conventions,
        IReadOnlyList<Action<EndpointBuilder>> groupConventions,
        IReadOnlyList<Action<EndpointBuilder>> finallyConventions,
        IReadOnlyList<Action<EndpointBuilder>> groupFinallyConventions);
    
    // 订阅 ActionDescriptorCollectionProvider 提供的变动令牌
    // 用以在 IActionDescriptorCollectionProvider 保存的 ActionDescriptor 集合变化时通知更新终结点集合
    protected void Subscribe()
    {
        if (_actions is ActionDescriptorCollectionProvider collectionProviderWithChangeToken)
        {
            _disposable = ChangeToken.OnChange(
                collectionProviderWithChangeToken.GetChangeToken,
                UpdateEndpoints);
        }
    }
 
    // 返回变动令牌，用于回调注册，在 Endpoint 集合发生变动时触发回调，通知重新获取 Endpoint 集合
    public override IChangeToken GetChangeToken()
    {
        // 初始化创建终结点集合
        Initialize();
        Debug.Assert(_changeToken != null);
        Debug.Assert(_endpoints != null);
        return _changeToken;
    }
 
    public void Dispose()
    {
        _disposable?.Dispose();
        _disposable = null;
    }
 
    private void Initialize()
    {
        // 需要保证线程同步，双重检查锁定
        // 初始化只会调用 UpdateEndpoints 方法一次，用于创建 Endpoint 集合，其他时候都是通过触发变动令牌回调来更新集合
        if (_endpoints == null)
        {
            lock (Lock)
            {
                if (_endpoints == null)
                {
                    UpdateEndpoints();
                }
            }
        }
    }
 
    // 更新 Endpoint 集合
    private void UpdateEndpoints()
    {
        lock (Lock)
        {
            // 利用 IActionDescriptorCollectionProvider 中的 ActionDescriptor 集合创建 Endpoint 集合
            var endpoints = CreateEndpoints(
                groupPrefix: null,
                _actions.ActionDescriptors.Items,
                conventions: Conventions,
                groupConventions: Array.Empty<Action<EndpointBuilder>>(),
                finallyConventions: FinallyConventions,
                groupFinallyConventions: Array.Empty<Action<EndpointBuilder>>());

            var oldCancellationTokenSource = _cancellationTokenSource;
 
            _endpoints = endpoints;

            // 利用 CancellationTokenSource 创建一个新的 CancellationChangeToken
            _cancellationTokenSource = new CancellationTokenSource();
            _changeToken = new CancellationChangeToken(_cancellationTokenSource.Token);

            // 利用旧的 CancellationChangeToken 触发回调
            oldCancellationTokenSource?.Cancel();
        }
    }
}
```

- ControllerActionEndpointDataSource

```C#
// 基于控制器的 ActionEndpointDataSource
// 是与 ControllerActionEndpointConventionBuilder 配套的 EndpointDataSource 子类
internal sealed class ControllerActionEndpointDataSource : ActionEndpointDataSourceBase
{
    private readonly ActionEndpointFactory _endpointFactory;
    private readonly OrderedEndpointsSequenceProvider _orderSequence;
    private readonly List<ConventionalRouteEntry> _routes;
 
    public ControllerActionEndpointDataSource(
        ControllerActionEndpointDataSourceIdProvider dataSourceIdProvider,
        IActionDescriptorCollectionProvider actions,
        ActionEndpointFactory endpointFactory,
        OrderedEndpointsSequenceProvider orderSequence)
        : base(actions)
    {
        _endpointFactory = endpointFactory;
 
        // 从 1 开始生成的自增 Id
        DataSourceId = dataSourceIdProvider.CreateId();
        _orderSequence = orderSequence;

        // 用来收集基于约定的路由
        _routes = new List<ConventionalRouteEntry>();
 
        // 创建默认 ControllerActionEndpointConventionBuilder
        DefaultBuilder = new ControllerActionEndpointConventionBuilder(Lock, Conventions, FinallyConventions);
 
        // 注册回调，监听 ActionDescriptor 集合的变动，用于更新 Endpoint 集合
        Subscribe();
    }
 
    public int DataSourceId { get; }
 
    public ControllerActionEndpointConventionBuilder DefaultBuilder { get; }

    public bool CreateInertEndpoints { get; set; }
 
    // 提供添加约定路由的入口方法，用于创建基于约定路由的 ConventionalRouteEntry
    public ControllerActionEndpointConventionBuilder AddRoute(
        string routeName,
        string pattern,
        RouteValueDictionary? defaults,
        IDictionary<string, object?>? constraints,
        RouteValueDictionary? dataTokens)
    {
        lock (Lock)
        {
            var conventions = new List<Action<EndpointBuilder>>();
            var finallyConventions = new List<Action<EndpointBuilder>>();
            // 添加的 ConventionalRouteEntry 从 0 开始编号
            _routes.Add(new ConventionalRouteEntry(routeName, pattern, defaults, constraints, dataTokens, _orderSequence.GetNext(), conventions, finallyConventions));
            // 返回一个新的 ControllerActionEndpointConventionBuilder
            return new ControllerActionEndpointConventionBuilder(Lock, conventions, finallyConventions);
        }
    }
    
    // 重写 CreateEndpoints 抽象方法
    // 利用 ControllerActionDescriptor 创建 Endpoint 集合
    // 每个 ControllerActionDescriptor 都会创建一个 Endpoint 
    protected override List<Endpoint> CreateEndpoints(
        RoutePattern? groupPrefix,
        IReadOnlyList<ActionDescriptor> actions,
        IReadOnlyList<Action<EndpointBuilder>> conventions,
        IReadOnlyList<Action<EndpointBuilder>> groupConventions,
        IReadOnlyList<Action<EndpointBuilder>> finallyConventions,
        IReadOnlyList<Action<EndpointBuilder>> groupFinallyConventions)
    {
        var endpoints = new List<Endpoint>();
        var keys = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
 
        // 确保路由名称唯一
        var routeNames = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
 
        // 遍历 ActionDescriptor 集合
        // 利用 ActionEndpointFactory 创建 Endpoint
        for (var i = 0; i < actions.Count; i++)
        {
            // 只处理 ControllerActionDescriptor 类型的 ActionDescriptor
            if (actions[i] is ControllerActionDescriptor action)
            {
                // 创建 Endpoint
                _endpointFactory.AddEndpoints(endpoints,
                                              routeNames,
                                              action,
                                              _routes,
                                              conventions: conventions,
                                              groupConventions: groupConventions,
                                              finallyConventions: finallyConventions,
                                              groupFinallyConventions: groupFinallyConventions,
                                              CreateInertEndpoints,
                                              groupPrefix: groupPrefix);

                // 如果存在基于约定的路由
                if (_routes.Count > 0)
                {
                    // 先从每个 ControllerActionDescriptor 的路由值字典中收集键
                    foreach (var kvp in action.RouteValues)
                    {
                        keys.Add(kvp.Key);
                    }
                }
            }
        }
 
        // 遍历基于约定的路由，创建出站 RoutePattern 用于构建 RouteEndpoint，提供生成链接使用
        for (var i = 0; i < _routes.Count; i++)
        {
            var route = _routes[i];
            _endpointFactory.AddConventionalLinkGenerationRoute(
                endpoints,
                routeNames,
                keys,
                route,
                groupConventions: groupConventions,
                conventions: conventions,
                finallyConventions: finallyConventions,
                groupFinallyConventions: groupFinallyConventions,
                groupPrefix: groupPrefix);
        }
 
        return endpoints;
    }
}
```

- ActionEndpointFactory

```C#
// 基于 ActionDescriptor 的 Endpoint 工厂
internal sealed class ActionEndpointFactory
{
    private readonly RoutePatternTransformer _routePatternTransformer;
    private readonly RequestDelegate _requestDelegate;
    private readonly IRequestDelegateFactory[] _requestDelegateFactories;
    private readonly IServiceProvider _serviceProvider;
 
    public ActionEndpointFactory(RoutePatternTransformer routePatternTransformer,
                                IEnumerable<IRequestDelegateFactory> requestDelegateFactories,
                                IServiceProvider serviceProvider)
    {
        ArgumentNullException.ThrowIfNull(routePatternTransformer);
 
        _routePatternTransformer = routePatternTransformer;
        _requestDelegate = CreateRequestDelegate();
        // 注入多个 IRequestDelegateFactory 服务
        _requestDelegateFactories = requestDelegateFactories.ToArray();
        _serviceProvider = serviceProvider;
    }

    // 实际创建 Endpoint 的方法
    public void AddEndpoints(
        List<Endpoint> endpoints,
        HashSet<string> routeNames,
        ActionDescriptor action,
        IReadOnlyList<ConventionalRouteEntry> routes,
        IReadOnlyList<Action<EndpointBuilder>> conventions,
        IReadOnlyList<Action<EndpointBuilder>> groupConventions,
        IReadOnlyList<Action<EndpointBuilder>> finallyConventions,
        IReadOnlyList<Action<EndpointBuilder>> groupFinallyConventions,
        bool createInertEndpoints,
        RoutePattern? groupPrefix = null)
    {
        ArgumentNullException.ThrowIfNull(nameof(endpoints));
        ArgumentNullException.ThrowIfNull(nameof(routeNames));
        ArgumentNullException.ThrowIfNull(nameof(action));
        ArgumentNullException.ThrowIfNull(nameof(routes));
        ArgumentNullException.ThrowIfNull(nameof(conventions));
        ArgumentNullException.ThrowIfNull(nameof(groupConventions));
        ArgumentNullException.ThrowIfNull(nameof(finallyConventions));
        ArgumentNullException.ThrowIfNull(nameof(groupFinallyConventions));

        // 检查 ActionDescriptor 是否为约定路由
        if (action.AttributeRouteInfo?.Template == null)
        {
            // 遍历 ConventionalRouteEntry 集合
            foreach (var route in routes)
            {
                // 利用 ActionDescriptor 中的路由值字典匹配 RoutePattern
                // 如果匹配上则返回拷贝的 RoutePattern 并使用 ActionDescriptor 中的路由值字典填充 RoutePattern.RequiredValues 字典
                // 注意：
                // 匹配规则就是根据原始 RoutePattern 中的路由模式参数名查找 ActionDescriptor.RouteValues 字典中是否存在对应的键并存在值
                var updatedRoutePattern = _routePatternTransformer.SubstituteRequiredValues(route.Pattern, action.RouteValues);
                // 没有匹配上则跳过继续
                if (updatedRoutePattern == null)
                {
                    continue;
                }
 
                // 合并路由前缀
                updatedRoutePattern = RoutePatternFactory.Combine(groupPrefix, updatedRoutePattern);

                // 利用 ActionDescriptor 创建 RequestDelegate
                var requestDelegate = CreateRequestDelegate(action, route.DataTokens) ?? _requestDelegate;
 
                // 创建 RouteEndpointBuilder
                var builder = new RouteEndpointBuilder(requestDelegate, updatedRoutePattern, route.Order)
                {
                    DisplayName = action.DisplayName,
                    ApplicationServices = _serviceProvider,
                };
                // 配置 RouteEndpointBuilder
                AddActionDataToBuilder(
                    builder,
                    routeNames,
                    action,
                    route.RouteName,
                    route.DataTokens,
                    suppressLinkGeneration: true,
                    suppressPathMatching: false,
                    groupConventions: groupConventions,
                    conventions: conventions,
                    perRouteConventions: route.Conventions,
                    groupFinallyConventions: groupFinallyConventions,
                    finallyConventions: finallyConventions,
                    perRouteFinallyConventions: route.FinallyConventions);
                // 创建 RouteEndpoint 并添加到集合中
                endpoints.Add(builder.Build());
            }
        }
        else
        {
            // 利用 ActionDescriptor 创建 RequestDelegate
            var requestDelegate = CreateRequestDelegate(action) ?? _requestDelegate;
            // 利用 ActionDescriptor.AttributeRouteInfo 创建 RoutePattern
            var attributeRoutePattern = RoutePatternFactory.Parse(action.AttributeRouteInfo.Template);
 
            // 利用 ActionDescriptor 中的路由值字典填充 RoutePattern.RequiredValues 字典
            var (resolvedRoutePattern, resolvedRouteValues) = ResolveDefaultsAndRequiredValues(action, attributeRoutePattern);
 
            // 利用 ActionDescriptor 中的路由值字典匹配 RoutePattern
            // 如果匹配上则返回拷贝的 RoutePattern 并使用 ActionDescriptor 中的路由值字典填充 RoutePattern.RequiredValues 字典
            // 注意：
            // 匹配规则就是根据原始 RoutePattern 中的路由模式参数名查找 ActionDescriptor.RouteValues 字典中是否存在对应的键并存在值
            var updatedRoutePattern = _routePatternTransformer.SubstituteRequiredValues(resolvedRoutePattern, resolvedRouteValues);
            if (updatedRoutePattern == null)
            {
                var formattedRouteKeys = string.Join(", ", resolvedRouteValues.Keys.Select(k => $"'{k}'"));
                throw new InvalidOperationException(
                    $"Failed to update the route pattern '{resolvedRoutePattern.RawText}' with required route values. " +
                    $"This can occur when the route pattern contains parameters with reserved names such as: {formattedRouteKeys} " +
                    $"and also uses route constraints such as '{{action:int}}'. " +
                    "To fix this error, choose a different parameter name.");
            }
 
            // 合并路由前缀
            updatedRoutePattern = RoutePatternFactory.Combine(groupPrefix, updatedRoutePattern);
 
            // 创建 RouteEndpointBuilder
            var builder = new RouteEndpointBuilder(requestDelegate, updatedRoutePattern, action.AttributeRouteInfo.Order)
            {
                DisplayName = action.DisplayName,
                ApplicationServices = _serviceProvider,
            };
            // 配置 RouteEndpointBuilder
            AddActionDataToBuilder(
                builder,
                routeNames,
                action,
                action.AttributeRouteInfo.Name,
                dataTokens: null,
                action.AttributeRouteInfo.SuppressLinkGeneration,
                action.AttributeRouteInfo.SuppressPathMatching,
                groupConventions: groupConventions,
                conventions: conventions,
                perRouteConventions: Array.Empty<Action<EndpointBuilder>>(),
                groupFinallyConventions: groupFinallyConventions,
                finallyConventions: finallyConventions,
                perRouteFinallyConventions: Array.Empty<Action<EndpointBuilder>>());
            // 创建 Endpoint 并添加到集合中
            endpoints.Add(builder.Build());
        }
    }

    // 配置 RouteEndpointBuilder
    // 主要工作就是将 ActionDescriptor 中保存的各种特性转移到 RouteEndpointBuilder 的元数据集合中
    private static void AddActionDataToBuilder(
        EndpointBuilder builder,
        HashSet<string> routeNames,
        ActionDescriptor action,
        string? routeName,
        RouteValueDictionary? dataTokens,
        bool suppressLinkGeneration,
        bool suppressPathMatching,
        IReadOnlyList<Action<EndpointBuilder>> groupConventions,
        IReadOnlyList<Action<EndpointBuilder>> conventions,
        IReadOnlyList<Action<EndpointBuilder>> perRouteConventions,
        IReadOnlyList<Action<EndpointBuilder>> groupFinallyConventions,
        IReadOnlyList<Action<EndpointBuilder>> finallyConventions,
        IReadOnlyList<Action<EndpointBuilder>> perRouteFinallyConventions)
    {
        for (var i = 0; i < groupConventions.Count; i++)
        {
            groupConventions[i](builder);
        }
 
        var controllerActionDescriptor = action as ControllerActionDescriptor;

        // 如果时 ControllerActionDescriptor 则检查方法的参数类型和返回值类型是否实现了 IEndpointMetadataProvider 接口
        // 如果实现了则调用对应的静态方法 PopulateMetadata 将元数据添加到 EndpointBuilder 的元数据集合中
        if (controllerActionDescriptor?.MethodInfo is not null)
        {
            EndpointMetadataPopulator.PopulateMetadata(controllerActionDescriptor.MethodInfo, builder);
        }
 
        // 将 ActionDescriptor.EndpointMetadata 中的元数据全部转移到 EndpointBuilder 的元数据集合中
        if (action.EndpointMetadata != null)
        {
            foreach (var d in action.EndpointMetadata)
            {
                builder.Metadata.Add(d);
            }
        }
 
        // 将 ActionDescriptor 自身添加到 EndpointBuilder 的元数据集合中
        builder.Metadata.Add(action);

        // 将路由名称添加到 EndpointBuilder 的元数据集合中
        // 目的是作为终结点名称
        if (routeName != null &&
            !suppressLinkGeneration &&
            routeNames.Add(routeName) &&
            builder.Metadata.OfType<IEndpointNameMetadata>().LastOrDefault()?.EndpointName == null)
        {
            builder.Metadata.Add(new EndpointNameMetadata(routeName));
        }
 
        // 将路由数据令牌添加到 EndpointBuilder 的元数据集合中
        if (dataTokens != null)
        {
            builder.Metadata.Add(new DataTokensMetadata(dataTokens));
        }

        // 将路由名称添加到 EndpointBuilder 的元数据集合中
        builder.Metadata.Add(new RouteNameMetadata(routeName));
 
        // 将 ActionDescriptor.FilterDescriptors 中的 IFilterMetadata 过滤器添加到 EndpointBuilder 的元数据集合中
        if (action.FilterDescriptors != null && action.FilterDescriptors.Count > 0)
        {
            foreach (var filter in action.FilterDescriptors.OrderBy(f => f, FilterDescriptorOrderComparer.Comparer).Select(f => f.Filter))
            {
                builder.Metadata.Add(filter);
            }
        }
 
        // 遍历 ActionDescriptor.ActionConstraints 集合
        if (action.ActionConstraints != null && action.ActionConstraints.Count > 0)
        {
            foreach (var actionConstraint in action.ActionConstraints)
            {
                if (actionConstraint is HttpMethodActionConstraint httpMethodActionConstraint &&
                    !builder.Metadata.OfType<HttpMethodMetadata>().Any())
                {
                    // 如果存在 HttpMethodActionConstraint 则 EndpointBuilder 的元数据集合中必须存在 HttpMethodMetadata
                    // 目的是用于终结点匹配
                    builder.Metadata.Add(new HttpMethodMetadata(httpMethodActionConstraint.HttpMethods));
                }
                else if (actionConstraint is ConsumesAttribute consumesAttribute &&
                    !builder.Metadata.OfType<AcceptsMetadata>().Any())
                {
                    // 如果存在 ConsumesAttribute 则 EndpointBuilder 的元数据集合中必须存在 AcceptsMetadata
                    // 目的是用于终结点匹配
                    builder.Metadata.Add(new AcceptsMetadata(consumesAttribute.ContentTypes.ToArray()));
                }
                else if (!builder.Metadata.Contains(actionConstraint))
                {
                    // 其他类型的 IActionConstraintMetadata 元数据直接添加到 EndpointBuilder 的元数据集合中
                    builder.Metadata.Add(actionConstraint);
                }
            }
        }
 
        // 将 SuppressLinkGenerationMetadata 添加到 EndpointBuilder 元数据集合中
        // 注意：
        // 基于约定的入站路由创建的 RouteEndpoint 禁止用于生成链接
        if (suppressLinkGeneration)
        {
            builder.Metadata.Add(new SuppressLinkGenerationMetadata());
        }
 
        // 将 SuppressMatchingMetadata 添加到 EndpointBuilder 元数据集合中
        if (suppressPathMatching)
        {
            builder.Metadata.Add(new SuppressMatchingMetadata());
        }
 
        // 使用约定配置配置 EndpointBuilder
        for (var i = 0; i < conventions.Count; i++)
        {
            conventions[i](builder);
        }
 
        for (var i = 0; i < perRouteConventions.Count; i++)
        {
            perRouteConventions[i](builder);
        }
 
        if (builder.FilterFactories.Count > 0 && controllerActionDescriptor is not null)
        {
            var routeHandlerFilters = builder.FilterFactories;
 
            EndpointFilterDelegate del = static invocationContext =>
            {
                var controllerInvocationContext = (ControllerEndpointFilterInvocationContext)invocationContext;
                return controllerInvocationContext.ActionDescriptor.CacheEntry!.InnerActionMethodExecutor.Execute(controllerInvocationContext);
            };
 
            var context = new EndpointFilterFactoryContext
            {
                MethodInfo = controllerActionDescriptor.MethodInfo,
                ApplicationServices = builder.ApplicationServices,
            };
 
            var initialFilteredInvocation = del;
 
            for (var i = routeHandlerFilters.Count - 1; i >= 0; i--)
            {
                var filterFactory = routeHandlerFilters[i];
                del = filterFactory(context, del);
            }
 
            controllerActionDescriptor.FilterDelegate = ReferenceEquals(del, initialFilteredInvocation) ? null : del;
        }
 
        foreach (var perRouteFinallyConvention in perRouteFinallyConventions)
        {
            perRouteFinallyConvention(builder);
        }
 
        foreach (var finallyConvention in finallyConventions)
        {
            finallyConvention(builder);
        }
 
        foreach (var groupFinallyConvention in groupFinallyConventions)
        {
            groupFinallyConvention(builder);
        }
    }

    // 利用 ActionDescriptor 创建 RequestDelegate
    private RequestDelegate? CreateRequestDelegate(ActionDescriptor action, RouteValueDictionary? dataTokens = null)
    {
        // 遍历 IRequestDelegateFactory 集合
        foreach (var factory in _requestDelegateFactories)
        {
            // 每个 IRequestDelegateFactory 内部会根据 ActionDescriptor 的实际类型决定是否由自己创建对应的 RequestDelegate
            // 如果不是则返回 null，继续交由下一个 IRequestDelegateFactory 处理
            var requestDelegate = factory.CreateRequestDelegate(action, dataTokens);
            if (requestDelegate != null)
            {
                return requestDelegate;
            }
        }
 
        return null;
    }
}
```

## 注册终结点

- ControllerEndpointRouteBuilderExtensions

```C#
// 提供用来注册路由系统终结点的扩展方法
public static class ControllerEndpointRouteBuilderExtensions
{
    // 注册特性路由终结点
    public static ControllerActionEndpointConventionBuilder MapControllers(this IEndpointRouteBuilder endpoints)
    {
        ArgumentNullException.ThrowIfNull(endpoints);

        // 确保 IServiceCollection.AddControllers 扩展方法被调用
        EnsureControllerServices(endpoints);

        // 创建 ControllerActionEndpointDataSource
        return GetOrCreateDataSource(endpoints).DefaultBuilder;
    }

    // 注册约定路由终结点
    public static ControllerActionEndpointConventionBuilder MapControllerRoute(
        this IEndpointRouteBuilder endpoints,
        string name,
        string pattern,
        object? defaults = null,
        object? constraints = null,
        object? dataTokens = null)
    {
        ArgumentNullException.ThrowIfNull(endpoints);
 
        // 确保 IServiceCollection.AddControllers 扩展方法被调用
        EnsureControllerServices(endpoints);
 
        // 创建 ControllerActionEndpointDataSource
        var dataSource = GetOrCreateDataSource(endpoints);
        // 添加约定路由模板
        return dataSource.AddRoute(
            name,
            pattern,
            new RouteValueDictionary(defaults),
            new RouteValueDictionary(constraints),
            new RouteValueDictionary(dataTokens));
    }

    // 注册默认的约定路由系统终结点
    public static ControllerActionEndpointConventionBuilder MapDefaultControllerRoute(this IEndpointRouteBuilder endpoints)
    {
        ArgumentNullException.ThrowIfNull(endpoints);
 
        // 确保 IServiceCollection.AddControllers 扩展方法被调用
        EnsureControllerServices(endpoints);
 
        // 创建 ControllerActionEndpointDataSource
        var dataSource = GetOrCreateDataSource(endpoints);
        // 添加默认约定路由模板
        return dataSource.AddRoute(
            "default",
            "{controller=Home}/{action=Index}/{id?}",
            defaults: null,
            constraints: null,
            dataTokens: null);
    }

    // 确保 IServiceCollection.AddControllers 扩展方法被调用
    private static void EnsureControllerServices(IEndpointRouteBuilder endpoints)
    {
        var marker = endpoints.ServiceProvider.GetService<MvcMarkerService>();
        if (marker == null)
        {
            throw new InvalidOperationException(Resources.FormatUnableToFindServices(
                nameof(IServiceCollection),
                "AddControllers",
                "ConfigureServices(...)"));
        }
    }
 
    // 创建 ControllerActionEndpointDataSource
    private static ControllerActionEndpointDataSource GetOrCreateDataSource(IEndpointRouteBuilder endpoints)
    {
        var dataSource = endpoints.DataSources.OfType<ControllerActionEndpointDataSource>().FirstOrDefault();
        if (dataSource == null)
        {
            // 得到 OrderedEndpointsSequenceProviderCache 服务，用于给创建的约定路由终结点编号
            var orderProvider = endpoints.ServiceProvider.GetRequiredService<OrderedEndpointsSequenceProviderCache>();
            // 得到 ControllerActionEndpointDataSourceFactory 服务
            var factory = endpoints.ServiceProvider.GetRequiredService<ControllerActionEndpointDataSourceFactory>();
            // 利用 ControllerActionEndpointDataSourceFactory 创建 ControllerActionEndpointDataSource
            dataSource = factory.Create(orderProvider.GetOrCreateOrderedEndpointsSequenceProvider(endpoints));
            // 将 ControllerActionEndpointDataSource 添加到 IEndpointRouteBuilder.DataSources 集合中
            endpoints.DataSources.Add(dataSource);
        }
 
        return dataSource;
    }
}
```

- ControllerActionEndpointDataSourceFactory

```C#
// ControllerActionEndpointDataSource 工厂
internal sealed class ControllerActionEndpointDataSourceFactory
{
    // 用于给 ControllerActionEndpointDataSource 创建自增 Id
    private readonly ControllerActionEndpointDataSourceIdProvider _dataSourceIdProvider;
    private readonly IActionDescriptorCollectionProvider _actions;
    private readonly ActionEndpointFactory _factory;
 
    public ControllerActionEndpointDataSourceFactory(
        ControllerActionEndpointDataSourceIdProvider dataSourceIdProvider,
        IActionDescriptorCollectionProvider actions,
        ActionEndpointFactory factory)
    {
        // 注入 IActionDescriptorCollectionProvider 和 ActionEndpointFactory
        _dataSourceIdProvider = dataSourceIdProvider;
        _actions = actions;
        _factory = factory;
    }
    
    // 创建 ControllerActionEndpointDataSource
    public ControllerActionEndpointDataSource Create(OrderedEndpointsSequenceProvider orderProvider)
    {
        return new ControllerActionEndpointDataSource(_dataSourceIdProvider, _actions, _factory, orderProvider);
    }
}
```

## Action 管线构建

- ActionContext

```C#
// Action 上下文
// 本质是对 HttpContxt、ActionDescriptor、RouteData、ModelStateDictionary 四元组的封装
public class ActionContext
{
    public ActionContext()
    {
        ModelState = new ModelStateDictionary();
    }
 
    public ActionContext(ActionContext actionContext)
        : this(
            actionContext.HttpContext,
            actionContext.RouteData,
            actionContext.ActionDescriptor,
            actionContext.ModelState)
    {
    }
 
    public ActionContext(
        HttpContext httpContext,
        RouteData routeData,
        ActionDescriptor actionDescriptor)
        : this(httpContext, routeData, actionDescriptor, new ModelStateDictionary())
    {
    }
 
    public ActionContext(
        HttpContext httpContext,
        RouteData routeData,
        ActionDescriptor actionDescriptor,
        ModelStateDictionary modelState)
    {
        ArgumentNullException.ThrowIfNull(httpContext);
        ArgumentNullException.ThrowIfNull(routeData);
        ArgumentNullException.ThrowIfNull(actionDescriptor);
        ArgumentNullException.ThrowIfNull(modelState);
 
        HttpContext = httpContext;
        RouteData = routeData;
        ActionDescriptor = actionDescriptor;
        ModelState = modelState;
    }
 
    public ActionDescriptor ActionDescriptor { get; set; } = default!;
 
    public HttpContext HttpContext { get; set; } = default!;
 
    public ModelStateDictionary ModelState { get; } = default!;
 
    public RouteData RouteData { get; set; } = default!;
}
```

- ControllerContext

```C#
// 基于控制器的 Action 上下文
public class ControllerContext : ActionContext
{
    private IList<IValueProviderFactory>? _valueProviderFactories;
 
    public ControllerContext()
    {
    }
 
    public ControllerContext(ActionContext context)
        : base(context)
    {
        // 保存的 ActionDescriptor 实际类型必须是 ControllerActionDescriptor
        if (context.ActionDescriptor is not ControllerActionDescriptor)
        {
            throw new ArgumentException(Resources.FormatActionDescriptorMustBeBasedOnControllerAction(
                typeof(ControllerActionDescriptor)),
                nameof(context));
        }
    }
 
    internal ControllerContext(
        HttpContext httpContext,
        RouteData routeData,
        ControllerActionDescriptor actionDescriptor)
        : base(httpContext, routeData, actionDescriptor)
    {
    }
    
    public new ControllerActionDescriptor ActionDescriptor
    {
        get { return (ControllerActionDescriptor)base.ActionDescriptor; }
        set { base.ActionDescriptor = value; }
    }
 
    public virtual IList<IValueProviderFactory> ValueProviderFactories
    {
        get
        {
            if (_valueProviderFactories == null)
            {
                _valueProviderFactories = new List<IValueProviderFactory>();
            }
 
            return _valueProviderFactories;
        }
        set
        {
            ArgumentNullException.ThrowIfNull(value);
 
            _valueProviderFactories = value;
        }
    }
 
    private string DebuggerToString() => ActionDescriptor?.DisplayName ?? $"{{{GetType().FullName}}}";
}
```

- IRequestDelegateFactory

```C#
// RequestDelegate 工厂接口
internal interface IRequestDelegateFactory
{
    // 利用 ActionDescriptor 创建 RequestDelegate
    // 支持提供额外的路由值字典
    RequestDelegate? CreateRequestDelegate(ActionDescriptor actionDescriptor, RouteValueDictionary? dataTokens);
}
```

- ControllerRequestDelegateFactory

```C#
// 基于 ControllerActionDescriptor 的 IRequestDelegateFactory 实现
internal sealed class ControllerRequestDelegateFactory : IRequestDelegateFactory
{
    private readonly ControllerActionInvokerCache _controllerActionInvokerCache;
    private readonly IReadOnlyList<IValueProviderFactory> _valueProviderFactories;
    private readonly int _maxModelValidationErrors;
    private readonly int? _maxValidationDepth;
    private readonly int _maxModelBindingRecursionDepth;
    private readonly ILogger _logger;
    private readonly DiagnosticListener _diagnosticListener;
    private readonly IActionResultTypeMapper _mapper;
    private readonly IActionContextAccessor _actionContextAccessor;
    private readonly bool _enableActionInvokers;
 
    public ControllerRequestDelegateFactory(
        ControllerActionInvokerCache controllerActionInvokerCache,
        IOptions<MvcOptions> optionsAccessor,
        ILoggerFactory loggerFactory,
        DiagnosticListener diagnosticListener,
        IActionResultTypeMapper mapper)
        : this(controllerActionInvokerCache, optionsAccessor, loggerFactory, diagnosticListener, mapper, null)
    {
    }
 
    public ControllerRequestDelegateFactory(
        ControllerActionInvokerCache controllerActionInvokerCache,
        IOptions<MvcOptions> optionsAccessor,
        ILoggerFactory loggerFactory,
        DiagnosticListener diagnosticListener,
        IActionResultTypeMapper mapper,
        IActionContextAccessor? actionContextAccessor)
    {
        _controllerActionInvokerCache = controllerActionInvokerCache;
        _valueProviderFactories = optionsAccessor.Value.ValueProviderFactories.ToArray();
        _maxModelValidationErrors = optionsAccessor.Value.MaxModelValidationErrors;
        _maxValidationDepth = optionsAccessor.Value.MaxValidationDepth;
        _maxModelBindingRecursionDepth = optionsAccessor.Value.MaxModelBindingRecursionDepth;
        _enableActionInvokers = optionsAccessor.Value.EnableActionInvokers;
        _logger = loggerFactory.CreateLogger<ControllerActionInvoker>();
        _diagnosticListener = diagnosticListener;
        _mapper = mapper;
        _actionContextAccessor = actionContextAccessor ?? ActionContextAccessor.Null;
    }
    
    // 利用 ControllerActionDescriptor 创建 RequestDelegate
    public RequestDelegate? CreateRequestDelegate(ActionDescriptor actionDescriptor, RouteValueDictionary? dataTokens)
    {
        // 只处理 ControllerActionDescriptor 类型的 ActionDescriptor
        if (_enableActionInvokers || actionDescriptor is not ControllerActionDescriptor controller)
        {
            return null;
        }

        return context =>
        {
            RouteData routeData;

            // 针对约定路由创建 RequestDelegate 时支持提供额外的路由值字典
            // 每次请求时会将这个额外的路由值字典和实际路由值字典合并
            if (dataTokens is null or { Count: 0 })
            {
                routeData = new RouteData(context.Request.RouteValues);
            }
            else
            {
                routeData = new RouteData();
                routeData.PushState(router: null, context.Request.RouteValues, dataTokens);
            }

            // 每个 HttpContext 都会创建一个 ControllerContext
            var controllerContext = new ControllerContext(context, routeData, controller)
            {
                ValueProviderFactories = new CopyOnWriteList<IValueProviderFactory>(_valueProviderFactories)
            };
 
            controllerContext.ModelState.MaxAllowedErrors = _maxModelValidationErrors;
            controllerContext.ModelState.MaxValidationDepth = _maxValidationDepth;
            controllerContext.ModelState.MaxStateDepth = _maxModelBindingRecursionDepth;
 
            // 缓存执行管线所需的数据
            // 从 ActionDescriptor.FilterDescriptors 得到的 IFilterMetadata 过滤器集合会先进行升序排序，没有实现 IOrderedFilter 的 IFilterMetadata 过滤器则默认为 0 的顺序
            // 实现了 IFilterFactory 的 IFilterMetadata 过滤器会根据 IFilterFactory.IsReusable 属性值决定是否缓存复用，这种过滤器会在每次请求执行管线前通过反射创建过滤器实例，并利用 HttpContext.RquestServices 属性表示的范围容器提供过滤器所需的依赖服务
            // 注意：
            // 绑定在控制器或方法上的以特性形式提供的过滤器一定会缓存复用
            // 因为特性会在运行时被反序列化为具体实例，这时候这个实例是唯一的
            var (cacheEntry, filters) = _controllerActionInvokerCache.GetCachedResult(controllerContext);

            // 创建 ControllerActionInvoker
            var invoker = new ControllerActionInvoker(
                _logger,
                _diagnosticListener,
                _actionContextAccessor,
                _mapper,
                controllerContext,
                cacheEntry,
                filters);

            // 开始执行管线
            return invoker.InvokeAsync();
        };
    }
}
```

- IActionInvoker

```C#
// Action 方法执行器接口
public interface IActionInvoker
{
    // 管线入口方法
    Task InvokeAsync();
}
```

- ResourceInvoker

```C#
// Action 方法执行器基类
internal abstract partial class ResourceInvoker
{
    protected readonly DiagnosticListener _diagnosticListener;
    protected readonly ILogger _logger;
    protected readonly IActionContextAccessor _actionContextAccessor;
    protected readonly IActionResultTypeMapper _mapper;
    protected readonly ActionContext _actionContext;
    protected readonly IFilterMetadata[] _filters;
    protected readonly IList<IValueProviderFactory> _valueProviderFactories;
 
    private AuthorizationFilterContextSealed? _authorizationContext;
    private ResourceExecutingContextSealed? _resourceExecutingContext;
    private ResourceExecutedContextSealed? _resourceExecutedContext;
    private ExceptionContextSealed? _exceptionContext;
    private ResultExecutingContextSealed? _resultExecutingContext;
    private ResultExecutedContextSealed? _resultExecutedContext;
 
    protected FilterCursor _cursor;
    protected IActionResult? _result;
    protected object? _instance;
 
    public ResourceInvoker(
        DiagnosticListener diagnosticListener,
        ILogger logger,
        IActionContextAccessor actionContextAccessor,
        IActionResultTypeMapper mapper,
        ActionContext actionContext,
        IFilterMetadata[] filters,
        IList<IValueProviderFactory> valueProviderFactories)
    {
        _diagnosticListener = diagnosticListener ?? throw new ArgumentNullException(nameof(diagnosticListener));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _actionContextAccessor = actionContextAccessor ?? throw new ArgumentNullException(nameof(actionContextAccessor));
        _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
        _actionContext = actionContext ?? throw new ArgumentNullException(nameof(actionContext));
 
        _filters = filters ?? throw new ArgumentNullException(nameof(filters));
        _valueProviderFactories = valueProviderFactories ?? throw new ArgumentNullException(nameof(valueProviderFactories));
        _cursor = new FilterCursor(filters);
    }

    // 完整管线
    // 管线流入方向：授权过滤器 -> 资源过滤器（流入）-> （异常过滤器捕获范围开始）参数绑定 -> 动作过滤器（流入）-> 动作方法开始 
    // 管线流出方向：动作方法结束 -> 动作过滤器（流出）-> 异常过滤器（异常过滤器捕获范围结束） -> 结果过滤器（流入）-> 结果执行 -> 结果过滤器（流出）-> 资源过滤器（流出）

    // 不同过滤器短路管线

    // 授权过滤器短路管线
    // 同步：设置 AuthorizationFilterContext.Result 属性
    // 异步：设置 AuthorizationFilterContext.Result 属性
    // 管线流入方向：授权过滤器
    // 管线流出方向：IAlwaysRunResultFilter 过滤器（流入）-> 结果执行 -> IAlwaysRunResultFilter 过滤器（流出）

    // 资源过滤器短路管线
    // 同步：设置 ResourceExecutingContext.Result 属性
    // 异步：设置 ResourceExecutingContext.Result 属性，并跳过 ResourceExecutionDelegate 的执行，否则会抛出异常
    // 管线流入方向：授权过滤器 -> 已执行的资源过滤器（流入）
    // 管线流出方向：IAlwaysRunResultFilter 过滤器（流入）-> 结果执行 -> IAlwaysRunResultFilter 过滤器（流出）-> 已执行的资源过滤器（流出）

    // 动作过滤器短路管线
    // 同步：设置 ActionExecutingContext.Result 属性
    // 异步：设置 ActionExecutingContext.Result 属性，并跳过 ActionExecutionDelegate 的执行，否则会抛出异常
    // 管线流入方向：授权过滤器 -> 资源过滤器（流入）-> 参数绑定 -> 已执行的动作过滤器（流入）
    // 管线流出方向：已执行的动作过滤器（流出）-> 异常过滤器 -> 结果过滤器（流入）-> 结果执行 -> 结果过滤器（流出）-> 资源过滤器（流出）

    // 异常过滤器短路管线
    // 同步：通过设置 ExceptionContext.Exception 属性为 null 或 ExceptionContext.IsHandled 属性为 true
    // 异步：通过设置 ExceptionContext.Exception 属性为 null 或 ExceptionContext.IsHandled 属性为 true
    // 实际上异常过滤器不存在短路管线，因为压栈的 Scope.Exception 管线范围内的异常过滤器会逐一检查跳过执行
    // 管线流入方向：授权过滤器 -> 资源过滤器（流入）-> 参数绑定 -> 动作过滤器（流入）-> 动作方法开始
    // 管线流出方向：动作方法结束 -> 动作过滤器（流出）-> 异常过滤器 -> 结果过滤器（流入）-> 结果执行 -> 结果过滤器（流出）-> 资源过滤器（流出）

    // 结果过滤器短路管线
    // 同步：设置 ResultExecutingContext.Cancel 属性为 true
    // 异步：设置 ResultExecutingContext.Cancel 属性为 true，并跳过 ResultExecutionDelegate 的执行
    // 管线流入方向：授权过滤器 -> 资源过滤器（流入）-> 参数绑定 -> 动作过滤器（流入）-> 动作方法开始 
    // 管线流出方向：动作方法结束 -> 动作过滤器（流出）-> 异常过滤器 -> 已执行的结果过滤器（流入）-> 已执行的结果过滤器（流出）-> 资源过滤器（流出）

    // 管线入口方法
    public virtual Task InvokeAsync()
    {
        if (_diagnosticListener.IsEnabled() || _logger.IsEnabled(LogLevel.Information))
        {
            return Logged(this);
        }
 
        _actionContextAccessor.ActionContext = _actionContext;
        var scope = _logger.ActionScope(_actionContext.ActionDescriptor);
 
        Task task;
        try
        {
            // 执行过滤器管线
            task = InvokeFilterPipelineAsync();
        }
        catch (Exception exception)
        {
            return Awaited(this, Task.FromException(exception), scope);
        }

        // 如果过滤器管线没有同步完成
        if (!task.IsCompletedSuccessfully)
        {
            // InvokeAsync 方法出栈，异步执行过滤器管线并返回一个用于等待的 Task
            return Awaited(this, task, scope);
        }
 
        return ReleaseResourcesCore(scope).AsTask();

        // 异步执行过滤器管线
        static async Task Awaited(ResourceInvoker invoker, Task task, IDisposable? scope)
        {
            try
            {
                await task;
            }
            finally
            {
                await invoker.ReleaseResourcesCore(scope);
            }
        }
    }

    // 执行过滤器管线
    // 压栈 Scope.Invoker 管线范围
    // 这是最外层的管线范围
    private Task InvokeFilterPipelineAsync()
    {
        // 设置管线范围内的下一个管线执行状态
        var next = State.InvokeBegin;

        // 创建 Scope.Invoker 管线范围
        var scope = Scope.Invoker;

        // 用于保存管线范围内的当前执行过滤器
        var state = (object?)null;

        var isCompleted = false;
        try
        {
            while (!isCompleted)
            {
                // 执行管线范围内的下一个管线执行状态
                var lastTask = Next(ref next, ref scope, ref state, ref isCompleted);
                // 没有同步完成
                if (!lastTask.IsCompletedSuccessfully)
                {
                    // 异步执行管线范围的下一个管线执行状态并返回一个用于等待的 Task
                    return Awaited(this, lastTask, next, scope, state, isCompleted);
                }
            }
 
            return Task.CompletedTask;
        }
        catch (Exception ex)
        {
            return Task.FromException(ex);
        }

        static async Task Awaited(ResourceInvoker invoker, Task lastTask, State next, Scope scope, object? state, bool isCompleted)
        {
            await lastTask;

            // 循环执行管线范围内的下一个管线执行状态
            while (!isCompleted)
            {
                await invoker.Next(ref next, ref scope, ref state, ref isCompleted);
            }
        }
    }

    // 执行管线范围内的下一个管线执行状态
    private Task Next(ref State next, ref Scope scope, ref object? state, ref bool isCompleted)
    {
        switch (next)
        {
            case State.InvokeBegin:
                {
                    // 初始状态
                    // 跳转到管线范围内的 State.AuthorizationBegin 管线执行状态
                    goto case State.AuthorizationBegin;
                }
 
            case State.AuthorizationBegin:
                {
                    // 重置过滤器游标
                    _cursor.Reset();
                    // 跳转到管线范围内的 State.AuthorizationNext 管线执行状态
                    goto case State.AuthorizationNext;
                }
 
            case State.AuthorizationNext:
                {
                    // 得到授权过滤器集合并将游标移动到下一个过滤器
                    var current = _cursor.GetNextFilter<IAuthorizationFilter, IAsyncAuthorizationFilter>();
                    // 确定是否是实现了 IAsyncAuthorizationFilter 的授权过滤器
                    if (current.FilterAsync != null)
                    {
                        if (_authorizationContext == null)
                        {
                            _authorizationContext = new AuthorizationFilterContextSealed(_actionContext, _filters);
                        }
                        
                        // 设置当前执行的过滤器
                        state = current.FilterAsync;
                        // 跳转到管线范围内的 State.AuthorizationAsyncBegin 管线执行状态
                        goto case State.AuthorizationAsyncBegin;
                    }
                    else if (current.Filter != null)
                    {
                        if (_authorizationContext == null)
                        {
                            _authorizationContext = new AuthorizationFilterContextSealed(_actionContext, _filters);
                        }
 
                        // 设置当前执行的过滤器
                        state = current.Filter;
                        // 跳转到管线范围内的 State.AuthorizationSync 管线执行状态
                        goto case State.AuthorizationSync;
                    }
                    else
                    {
                        // 如果不存在授权过滤器
                        // 跳转到管线范围内的 State.AuthorizationEnd 管线执行状态
                        // 表示过滤器管线结束
                        goto case State.AuthorizationEnd;
                    }
                }
 
            case State.AuthorizationAsyncBegin:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_authorizationContext != null);
 
                    var filter = (IAsyncAuthorizationFilter)state;
                    var authorizationContext = _authorizationContext;
 
                    _diagnosticListener.BeforeOnAuthorizationAsync(authorizationContext, filter);
                    _logger.BeforeExecutingMethodOnFilter(
                        FilterTypeConstants.AuthorizationFilter,
                        nameof(IAsyncAuthorizationFilter.OnAuthorizationAsync),
                        filter);

                    // 执行 IAsyncAuthorizationFilter.OnAuthorizationAsync 方法
                    var task = filter.OnAuthorizationAsync(authorizationContext);
                    // 没有同步完成
                    if (!task.IsCompletedSuccessfully)
                    {
                        // 设置管线范围内的下一个管线执行状态为 State.AuthorizationAsyncEnd
                        next = State.AuthorizationAsyncEnd;
                        return task;
                    }

                    // 同步完成
                    // 设置管线范围内的下一个管线执行状态为 State.AuthorizationAsyncEnd
                    goto case State.AuthorizationAsyncEnd;
                }
 
            case State.AuthorizationAsyncEnd:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_authorizationContext != null);
 
                    var filter = (IAsyncAuthorizationFilter)state;
                    var authorizationContext = _authorizationContext;
 
                    _diagnosticListener.AfterOnAuthorizationAsync(authorizationContext, filter);
                    _logger.AfterExecutingMethodOnFilter(
                        FilterTypeConstants.AuthorizationFilter,
                        nameof(IAsyncAuthorizationFilter.OnAuthorizationAsync),
                        filter);

                    // 如果 AuthorizationFilterContext.Result 属性不为 null
                    // 跳转到管线范围内的 State.AuthorizationShortCircuit 管线执行状态，短路管线
                    if (authorizationContext.Result != null)
                    {
                        goto case State.AuthorizationShortCircuit;
                    }

                    // 跳转到管线范围内的 State.AuthorizationNext 管线执行状态
                    // 继续执行下一个授权过滤器
                    goto case State.AuthorizationNext;
                }
 
            case State.AuthorizationSync:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_authorizationContext != null);
 
                    var filter = (IAuthorizationFilter)state;
                    var authorizationContext = _authorizationContext;
 
                    _diagnosticListener.BeforeOnAuthorization(authorizationContext, filter);
                    _logger.BeforeExecutingMethodOnFilter(
                        FilterTypeConstants.AuthorizationFilter,
                        nameof(IAuthorizationFilter.OnAuthorization),
                        filter);

                    // 执行 IAuthorizationFilter.OnAuthorization 方法
                    filter.OnAuthorization(authorizationContext);

                    _diagnosticListener.AfterOnAuthorization(authorizationContext, filter);
                    _logger.AfterExecutingMethodOnFilter(
                        FilterTypeConstants.AuthorizationFilter,
                        nameof(IAuthorizationFilter.OnAuthorization),
                        filter);

                    // 如果 AuthorizationFilterContext.Result 属性不为 null
                    // 跳转到管线范围内的 State.AuthorizationShortCircuit 管线执行状态，短路管线
                    if (authorizationContext.Result != null)
                    {
                        goto case State.AuthorizationShortCircuit;
                    }

                    // 跳转到管线范围内的 State.AuthorizationNext 管线执行状态
                    // 继续执行下一个授权过滤器
                    goto case State.AuthorizationNext;
                }
 
            case State.AuthorizationShortCircuit:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_authorizationContext != null);
                    Debug.Assert(_authorizationContext.Result != null);
                    Log.AuthorizationFailure(_logger, (IFilterMetadata)state);

                    // 设置当前管线范围结束标志
                    isCompleted = true;
                    // 提取授权过滤器设置的结果
                    _result = _authorizationContext.Result;
                    // 执行过滤器短路管线
                    // 本质是执行 IAlwaysRunResultFilter 结果过滤器管线
                    return InvokeAlwaysRunResultFilters();
                }
 
            case State.AuthorizationEnd:
                {
                    // 授权过滤器管线正常完成
                    // 跳转到 State.ResourceBegin
                    // 准备进入资源过滤器管线
                    goto case State.ResourceBegin;
                }
 
            case State.ResourceBegin:
                {
                    // 重置过滤器游标
                    _cursor.Reset();
                    // 跳转到 State.ResourceNext
                    // 开始资源过滤器管线
                    goto case State.ResourceNext;
                }
 
            case State.ResourceNext:
                {
                    // 得到资源过滤器集合并将游标移动到下一个过滤器
                    var current = _cursor.GetNextFilter<IResourceFilter, IAsyncResourceFilter>();
                    // 优先作为 IAsyncResourceFilter 资源过滤器
                    if (current.FilterAsync != null)
                    {
                        if (_resourceExecutingContext == null)
                        {
                            _resourceExecutingContext = new ResourceExecutingContextSealed(
                                _actionContext,
                                _filters,
                                _valueProviderFactories);
                        }
 
                        state = current.FilterAsync;
                        // 跳转到 State.ResourceAsyncBegin
                        // 以异步方式执行资源过滤器
                        goto case State.ResourceAsyncBegin;
                    }
                    else if (current.Filter != null)
                    {
                        if (_resourceExecutingContext == null)
                        {
                            _resourceExecutingContext = new ResourceExecutingContextSealed(
                                _actionContext,
                                _filters,
                                _valueProviderFactories);
                        }
 
                        state = current.Filter;
                        // 跳转到 State.ResourceSyncBegin
                        // 以同步方式执行资源过滤器
                        goto case State.ResourceSyncBegin;
                    }
                    else
                    {
                        // 跳转到 State.ResourceInside
                        goto case State.ResourceInside;
                    }
                }
 
            case State.ResourceAsyncBegin:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_resourceExecutingContext != null);
 
                    var filter = (IAsyncResourceFilter)state;
                    var resourceExecutingContext = _resourceExecutingContext;
 
                    _diagnosticListener.BeforeOnResourceExecution(resourceExecutingContext, filter);
                    _logger.BeforeExecutingMethodOnFilter(
                        FilterTypeConstants.ResourceFilter,
                        nameof(IAsyncResourceFilter.OnResourceExecutionAsync),
                        filter);
 
                    // 执行 IAsyncResourceFilter.OnResourceExecutionAsync 方法
                    // 将 InvokeNextResourceFilterAwaitedAsync 方法包装为 ResourceExecutionDelegate 委托传入，可以使用异步方式执行后续资源过滤器管线
                    var task = filter.OnResourceExecutionAsync(resourceExecutingContext, InvokeNextResourceFilterAwaitedAsync);
                    // 没有同步完成
                    if (!task.IsCompletedSuccessfully)
                    {
                        // 设置管线范围内的下一个管线执行状态为 State.ResourceAsyncEnd
                        next = State.ResourceAsyncEnd;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }
 
                    // 跳转到 State.ResourceAsyncEnd
                    goto case State.ResourceAsyncEnd;
                }
 
            case State.ResourceAsyncEnd:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_resourceExecutingContext != null);
 
                    var filter = (IAsyncResourceFilter)state;
                    try
                    {
                        // 如果没有设置全局 ResourceExecutedContext，表明在某个 IAsyncResourceFilter 过滤器内跳过了 ResourceExecutionDelegate 的执行
                        // 因为只要正常通过资源过滤器流入管线就会创建 ResourceExecutedContext
                        if (_resourceExecutedContext == null)
                        {
                            _resourceExecutedContext = new ResourceExecutedContextSealed(_resourceExecutingContext, _filters)
                            {
                                // 标识管线是被取消的
                                Canceled = true,
                                Result = _resourceExecutingContext.Result,
                            };
 
                            // 如果 ResourceExecutingContext.Result 属性不为 null
                            // 跳转到 State.ResourceShortCircuit，短路管线
                            if (_resourceExecutingContext.Result != null)
                            {
                                goto case State.ResourceShortCircuit;
                            }
                        }
                    }
                    finally
                    {
                        _diagnosticListener.AfterOnResourceExecution(_resourceExecutedContext, filter);
                        _logger.AfterExecutingMethodOnFilter(
                            FilterTypeConstants.ResourceFilter,
                            nameof(IAsyncResourceFilter.OnResourceExecutionAsync),
                            filter);
                    }
                    
                    // 跳转到 State.ResourceEnd
                    goto case State.ResourceEnd;
                }
 
            case State.ResourceSyncBegin:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_resourceExecutingContext != null);
 
                    var filter = (IResourceFilter)state;
                    var resourceExecutingContext = _resourceExecutingContext;
 
                    _diagnosticListener.BeforeOnResourceExecuting(resourceExecutingContext, filter);
                    _logger.BeforeExecutingMethodOnFilter(
                        FilterTypeConstants.ResourceFilter,
                        nameof(IResourceFilter.OnResourceExecuting),
                        filter);
 
                    // 执行 IResourceFilter.OnResourceExecuting 方法
                    filter.OnResourceExecuting(resourceExecutingContext);
 
                    _diagnosticListener.AfterOnResourceExecuting(resourceExecutingContext, filter);
                    _logger.AfterExecutingMethodOnFilter(
                        FilterTypeConstants.ResourceFilter,
                        nameof(IResourceFilter.OnResourceExecuting),
                        filter);

                    // 如果 ResourceExecutingContext.Result 属性不为 null
                    // 跳转到 State.ResourceShortCircuit，短路管线
                    if (resourceExecutingContext.Result != null)
                    {
                        _resourceExecutedContext = new ResourceExecutedContextSealed(resourceExecutingContext, _filters)
                        {
                            Canceled = true,
                            Result = _resourceExecutingContext.Result,
                        };
 
                        // 跳转到 State.ResourceShortCircuit
                        goto case State.ResourceShortCircuit;
                    }

                    // 执行后续资源过滤器管线
                    var task = InvokeNextResourceFilter();
                    // 没有同步完成
                    if (!task.IsCompletedSuccessfully)
                    {
                        // 设置管线范围内的下一个管线执行状态为 State.ResourceSyncEnd
                        next = State.ResourceSyncEnd;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }
                    
                    // 跳转到 State.ResourceSyncEnd
                    goto case State.ResourceSyncEnd;
                }
 
            case State.ResourceSyncEnd:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_resourceExecutingContext != null);
                    Debug.Assert(_resourceExecutedContext != null);
 
                    var filter = (IResourceFilter)state;
                    var resourceExecutedContext = _resourceExecutedContext;
 
                    _diagnosticListener.BeforeOnResourceExecuted(resourceExecutedContext, filter);
                    _logger.BeforeExecutingMethodOnFilter(
                        FilterTypeConstants.ResourceFilter,
                        nameof(IResourceFilter.OnResourceExecuted),
                        filter);
 
                    // 执行 IResourceFilter.OnResourceExecuted 方法
                    filter.OnResourceExecuted(resourceExecutedContext);
 
                    _diagnosticListener.AfterOnResourceExecuted(resourceExecutedContext, filter);
                    _logger.AfterExecutingMethodOnFilter(
                        FilterTypeConstants.ResourceFilter,
                        nameof(IResourceFilter.OnResourceExecuted),
                        filter);
 
                    // 跳转到 State.ResourceEnd
                    goto case State.ResourceEnd;
                }
 
            case State.ResourceShortCircuit:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_resourceExecutingContext != null);
                    Debug.Assert(_resourceExecutedContext != null);
                    Log.ResourceFilterShortCircuited(_logger, (IFilterMetadata)state);
 
                    // 提取资源过滤器设置的结果
                    _result = _resourceExecutingContext.Result;
                    // 执行过滤器短路管线
                    // 本质是执行 IAlwaysRunResultFilter 结果过滤器管线
                    var task = InvokeAlwaysRunResultFilters();
                    // 没有同步完成
                    if (!task.IsCompletedSuccessfully)
                    {
                        // 设置管线范围内的下一个管线执行状态为 State.ResourceEnd
                        next = State.ResourceEnd;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }

                    // 跳转到 State.ResourceEnd
                    goto case State.ResourceEnd;
                }
 
            case State.ResourceInside:
                {
                    // 资源过滤器流入管线正常完成
                    // 准备进入异常过滤器管线
                    // 跳转到 State.ExceptionBegin
                    goto case State.ExceptionBegin;
                }
 
            case State.ExceptionBegin:
                {
                    // 重置过滤器游标
                    _cursor.Reset();
                    // 跳转到 State.ExceptionNext
                    // 开始异常过滤器管线
                    goto case State.ExceptionNext;
                }
 
            case State.ExceptionNext:
                {
                    // 得到异常过滤器集合并将游标移动到下一个过滤器
                    var current = _cursor.GetNextFilter<IExceptionFilter, IAsyncExceptionFilter>();
                    // 优先作为 IAsyncExceptionFilter 异常过滤器
                    if (current.FilterAsync != null)
                    {
                        state = current.FilterAsync;
                        // 跳转到 State.ExceptionAsyncBegin
                        goto case State.ExceptionAsyncBegin;
                    }
                    else if (current.Filter != null)
                    {
                        state = current.Filter;
                        // 跳转到 State.ExceptionSyncBegin
                        goto case State.ExceptionSyncBegin;
                    }
                    else if (scope == Scope.Exception)
                    {
                        // 跳转到 State.ExceptionInside
                        // 进入这里表明存在异常过滤器，并且已经进入异常过滤器管线范围
                        // 注意：
                        // 异常过滤器在管线的流入方向上不工作，只是构建 try...catch 块，用来捕获异常
                        // 异常捕获范围：
                        // 1. 参数绑定
                        // 2. 动作过滤器流入管线
                        // 3. 动作方法执行
                        // 4. 动作过滤器流出管线
                        // 不包括授权过滤器、资源过滤器、结果过滤器以及 IActionResult 执行
                        goto case State.ExceptionInside;
                    }
                    else
                    {
                        // 进入这里表明不存在异常过滤器
                        Debug.Assert(scope == Scope.Invoker || scope == Scope.Resource);
                        // 跳转到 State.ActionBegin
                        // 准备进入动作过滤器管线
                        goto case State.ActionBegin;
                    }
                }
 
            case State.ExceptionAsyncBegin:
                {
                    // 执行后续异常过滤器管线范围
                    var task = InvokeNextExceptionFilterAsync();
                    if (!task.IsCompletedSuccessfully)
                    {
                        // 设置管线范围内的下一个管线执行状态为 State.ExceptionAsyncResume
                        next = State.ExceptionAsyncResume;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }

                    // 跳转到 State.ExceptionAsyncResume
                    // 在管线流出方向上准备进入异常过滤器管线
                    goto case State.ExceptionAsyncResume;
                }
 
            case State.ExceptionAsyncResume:
                {
                    Debug.Assert(state != null);
 
                    var filter = (IAsyncExceptionFilter)state;
                    var exceptionContext = _exceptionContext;

                    // 如果没有发生异常或标记已处理则跳过当前异常过滤器执行
                    if (exceptionContext?.Exception != null && !exceptionContext.ExceptionHandled)
                    {
                        _diagnosticListener.BeforeOnExceptionAsync(exceptionContext, filter);
                        _logger.BeforeExecutingMethodOnFilter(
                            FilterTypeConstants.ExceptionFilter,
                            nameof(IAsyncExceptionFilter.OnExceptionAsync),
                            filter);
 
                        // 执行 IAsyncExceptionFilter.OnExceptionAsync 方法
                        var task = filter.OnExceptionAsync(exceptionContext);
                        // 没有同步完成
                        if (!task.IsCompletedSuccessfully)
                        {
                            // 设置管线范围内的下一个管线执行状态为 State.ExceptionAsyncEnd
                            next = State.ExceptionAsyncEnd;
                            // 返回 Task 用于等待管线范围完成
                            return task;
                        }
 
                        // 跳转到 State.ExceptionAsyncEnd
                        goto case State.ExceptionAsyncEnd;
                    }

                    // 跳转到 State.ExceptionEnd
                    goto case State.ExceptionEnd;
                }
 
            case State.ExceptionAsyncEnd:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_exceptionContext != null);
 
                    var filter = (IAsyncExceptionFilter)state;
                    var exceptionContext = _exceptionContext;
 
                    _diagnosticListener.AfterOnExceptionAsync(exceptionContext, filter);
                    _logger.AfterExecutingMethodOnFilter(
                        FilterTypeConstants.ExceptionFilter,
                        nameof(IAsyncExceptionFilter.OnExceptionAsync),
                        filter);
 
                    if (exceptionContext.Exception == null || exceptionContext.ExceptionHandled)
                    {
                        _logger.ExceptionFilterShortCircuited(filter);
                    }
 
                    // 跳转到 State.ExceptionEnd
                    goto case State.ExceptionEnd;
                }
 
            case State.ExceptionSyncBegin:
                {
                    // 执行后续异常过滤器管线范围
                    var task = InvokeNextExceptionFilterAsync();
                    // 没有同步完成
                    if (!task.IsCompletedSuccessfully)
                    {
                        // 设置管线范围内的下一个管线执行状态为 State.ExceptionSyncEnd
                        next = State.ExceptionSyncEnd;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }

                    // 跳转到 State.ExceptionSyncEnd
                    goto case State.ExceptionSyncEnd;
                }
 
            case State.ExceptionSyncEnd:
                {
                    Debug.Assert(state != null);
 
                    var filter = (IExceptionFilter)state;
                    var exceptionContext = _exceptionContext;
 
                    // 如果没有发生异常或标记已处理则跳过当前异常过滤器执行
                    if (exceptionContext?.Exception != null && !exceptionContext.ExceptionHandled)
                    {
                        _diagnosticListener.BeforeOnException(exceptionContext, filter);
                        _logger.BeforeExecutingMethodOnFilter(
                            FilterTypeConstants.ExceptionFilter,
                            nameof(IExceptionFilter.OnException),
                            filter);

                        // 执行 IExceptionFilter.OnException 方法
                        filter.OnException(exceptionContext);
 
                        _diagnosticListener.AfterOnException(exceptionContext, filter);
                        _logger.AfterExecutingMethodOnFilter(
                            FilterTypeConstants.ExceptionFilter,
                            nameof(IExceptionFilter.OnException),
                            filter);
 
                        if (exceptionContext.Exception == null || exceptionContext.ExceptionHandled)
                        {
                            _logger.ExceptionFilterShortCircuited(filter);
                        }
                    }

                    // 跳转到 State.ExceptionEnd
                    goto case State.ExceptionEnd;
                }
 
            case State.ExceptionInside:
                {
                    // 跳转到 State.ActionBegin
                    goto case State.ActionBegin;
                }
 
            case State.ExceptionHandled:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_exceptionContext != null);
 
                    if (_exceptionContext.Result == null)
                    {
                        _exceptionContext.Result = new EmptyResult();
                    }
 
                    _result = _exceptionContext.Result;
                    
                    // 执行结果过滤器管线范围
                    // 本质是执行 IAlwaysRunResultFilter 结果过滤器管线
                    var task = InvokeAlwaysRunResultFilters();
                    // 没有同步完成
                    if (!task.IsCompletedSuccessfully)
                    {
                        // 设置当前管道范围下一个管线执行状态为 State.ResourceInsideEnd
                        next = State.ResourceInsideEnd;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }

                    // 跳转到 State.ResourceInsideEnd
                    goto case State.ResourceInsideEnd;
                }
 
            case State.ExceptionEnd:
                {
                    var exceptionContext = _exceptionContext;
                    
                    // 如果当前处于 Scope.Exception 管线范围
                    if (scope == Scope.Exception)
                    {
                        // 设置当前管线范围结束标志
                        isCompleted = true;
                        return Task.CompletedTask;
                    }

                    // 如果 State.Exception 管线范围完成后存在异常
                    if (exceptionContext != null)
                    {
                        // 判断异常是否已被处理
                        // 满足以下条件之一
                        // 1. 标记已处理状态
                        // 2. 原始异常已被清空
                        // 3. ExceptionContext.Result 属性不为 null
                        if (exceptionContext.Result != null ||
                            exceptionContext.Exception == null ||
                            exceptionContext.ExceptionHandled)
                        {
                            // 跳转到 State.ExceptionHandled，短路管线
                            goto case State.ExceptionHandled;
                        }

                        // 重新抛出异常
                        Rethrow(exceptionContext);
                        Debug.Fail("unreachable");
                    }

                    // 执行结果过滤器管线范围
                    var task = InvokeResultFilters();
                    // 没有同步完成
                    if (!task.IsCompletedSuccessfully)
                    {
                        // 设置当前管道范围下一个管线执行状态为 State.ResourceInsideEnd
                        next = State.ResourceInsideEnd;
                        // 返回管线范围完成情况的 Task
                        return task;
                    }
                    // 跳转到 State.ResourceInsideEnd
                    goto case State.ResourceInsideEnd;
                }
 
            case State.ActionBegin:
                {
                    // 执行动作过滤器管线范围
                    var task = InvokeInnerFilterAsync();
                    // 没有同步完成
                    if (!task.IsCompletedSuccessfully)
                    {
                        // 设置当前管道范围下一个管线执行状态为 State.ActionEnd
                        next = State.ActionEnd;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }

                    // 跳转到 State.ActionEnd
                    goto case State.ActionEnd;
                }
 
            case State.ActionEnd:
                {
                    // 如果当前处于 Scope.Exception 管线范围
                    // 表明存在异常过滤器管线，并且没有发生任何异常
                    if (scope == Scope.Exception)
                    {
                        // 设置当前管线范围结束标志
                        isCompleted = true;
                        return Task.CompletedTask;
                    }
 
                    Debug.Assert(scope == Scope.Invoker || scope == Scope.Resource);
                    // 执行结果过滤器管线范围
                    var task = InvokeResultFilters();
                    // 没有同步完成
                    if (!task.IsCompletedSuccessfully)
                    {
                        // 设置当前管道范围下一个管线执行状态为 State.ResourceInsideEnd
                        next = State.ResourceInsideEnd;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }
                    // 跳转到 State.ResourceInsideEnd
                    goto case State.ResourceInsideEnd;
                }
 
            case State.ResourceInsideEnd:
                {
                    // 如果当前处于 Scope.Resource 管线范围
                    if (scope == Scope.Resource)
                    {
                        // 创建 ResourceExecutedContext
                        _resourceExecutedContext = new ResourceExecutedContextSealed(_actionContext, _filters)
                        {
                            Result = _result,
                        };
 
                        // 跳转到 State.ResourceEnd
                        goto case State.ResourceEnd;
                    }

                    // 跳转到 State.InvokeEnd
                    goto case State.InvokeEnd;
                }
 
            case State.ResourceEnd:
                {
                    // 如果当前处于 Scope.Resource 管线范围
                    if (scope == Scope.Resource)
                    {
                        // 设置当前管线范围结束标志
                        isCompleted = true;
                        return Task.CompletedTask;
                    }
 
                    Debug.Assert(scope == Scope.Invoker);
                    Rethrow(_resourceExecutedContext!);

                    // 跳转到 State.InvokeEnd
                    goto case State.InvokeEnd;
                }
 
            case State.InvokeEnd:
                {
                    // 结束管线范围
                    isCompleted = true;
                    return Task.CompletedTask;
                }
 
            default:
                throw new InvalidOperationException();
        }
    }

    // 后续资源过滤器管线入口方法
    private Task<ResourceExecutedContext> InvokeNextResourceFilterAwaitedAsync()
    {
        Debug.Assert(_resourceExecutingContext != null);
 
        // 如果在 IAsyncResourceFilter 过滤器内想通过设置 ResourceExecutingContext.Result 属性来短路管线，就应该配合跳过 ResourceExecutionDelegate 的执行
        // 否则会抛出异常
        if (_resourceExecutingContext.Result != null)
        {
            return Throw();
        }

        // 执行后续资源过滤器管线范围
        var task = InvokeNextResourceFilter();
        // 没有同步完成
        if (!task.IsCompletedSuccessfully)
        {
            // 切换为异步等待模式
            // 返回管线范围完成情况的 Task<ResourceExecutedContext>
            return Awaited(this, task);
        }
 
        Debug.Assert(_resourceExecutedContext != null);
        // 同步完成直接返回 Task<ResourceExecutedContext>
        return Task.FromResult<ResourceExecutedContext>(_resourceExecutedContext);
 
        static async Task<ResourceExecutedContext> Awaited(ResourceInvoker invoker, Task task)
        {
            await task;
 
            Debug.Assert(invoker._resourceExecutedContext != null);
            return invoker._resourceExecutedContext;
        }
#pragma warning disable CS1998
        static async Task<ResourceExecutedContext> Throw()
        {
            var message = Resources.FormatAsyncResourceFilter_InvalidShortCircuit(
                typeof(IAsyncResourceFilter).Name,
                nameof(ResourceExecutingContext.Result),
                typeof(ResourceExecutingContext).Name,
                typeof(ResourceExecutionDelegate).Name);
            throw new InvalidOperationException(message);
        }
#pragma warning restore CS1998
    }

    // 资源过滤器管线执行方法
    private Task InvokeNextResourceFilter()
    {
        try
        {
            // 创建 Scope.Resource 管线范围
            var scope = Scope.Resource;
            // 设置管线范围下一个管线执行状态为 State.ResourceNext
            var next = State.ResourceNext;
            // 用于保存管线范围内的当前执行过滤器
            var state = (object?)null;
            var isCompleted = false;
 
            while (!isCompleted)
            {
                // 执行管线的下一个状态
                var lastTask = Next(ref next, ref scope, ref state, ref isCompleted);

                // 没有同步完成
                if (!lastTask.IsCompletedSuccessfully)
                {
                    // 切换为异步等待模式
                    // 返回 Task 用于等待管线范围完成
                    return Awaited(this, lastTask, next, scope, state, isCompleted);
                }
            }
        }
        catch (Exception exception)
        {
            _resourceExecutedContext = new ResourceExecutedContextSealed(_resourceExecutingContext!, _filters)
            {
                ExceptionDispatchInfo = ExceptionDispatchInfo.Capture(exception),
            };
        }
 
        Debug.Assert(_resourceExecutedContext != null);
        return Task.CompletedTask;
 
        static async Task Awaited(ResourceInvoker invoker, Task lastTask, State next, Scope scope, object? state, bool isCompleted)
        {
            try
            {
                await lastTask;
 
                while (!isCompleted)
                {
                    await invoker.Next(ref next, ref scope, ref state, ref isCompleted);
                }
            }
            catch (Exception exception)
            {
                invoker._resourceExecutedContext = new ResourceExecutedContextSealed(invoker._resourceExecutingContext!, invoker._filters)
                {
                    ExceptionDispatchInfo = ExceptionDispatchInfo.Capture(exception),
                };
            }
 
            Debug.Assert(invoker._resourceExecutedContext != null);
        }
    }

    // 后续异常过滤器管线执行方法
    private Task InvokeNextExceptionFilterAsync()
    {
        try
        {
            // 设置管线范围下一个管线执行状态为 State.ExceptionNext
            var next = State.ExceptionNext;
            // 用于保存管线范围内的当前执行过滤器
            var state = (object?)null;
            // 创建 Scope.Exception 管线范围
            var scope = Scope.Exception;
            var isCompleted = false;
            
            while (!isCompleted)
            {
                // 执行管线的下一个状态
                var lastTask = Next(ref next, ref scope, ref state, ref isCompleted);
                // 没有同步完成
                if (!lastTask.IsCompletedSuccessfully)
                {
                    // 切换为异步等待模式
                    // 返回 Task 用于等待管线范围完成
                    return Awaited(this, lastTask, next, scope, state, isCompleted);
                }
            }
 
            return Task.CompletedTask;
        }
        catch (Exception ex)
        {
            return Task.FromException(ex);
        }
 
        static async Task Awaited(ResourceInvoker invoker, Task lastTask, State next, Scope scope, object? state, bool isCompleted)
        {
            try
            {
                await lastTask;
 
                while (!isCompleted)
                {
                    await invoker.Next(ref next, ref scope, ref state, ref isCompleted);
                }
            }
            catch (Exception exception)
            {
                // 在 Scope.Exception 管线范围内捕获异常，并将异常保存到 ExceptionContext 对象中
                invoker._exceptionContext = new ExceptionContextSealed(invoker._actionContext, invoker._filters)
                {
                    ExceptionDispatchInfo = ExceptionDispatchInfo.Capture(exception),
                };
            }
        }
    }

    // 执行 IAlwaysRunResultFilter 结果过滤器管线方法
    private Task InvokeAlwaysRunResultFilters()
    {
        try
        {
            // 设置管线范围下一个管线执行状态为 State.ResultBegin
            var next = State.ResultBegin;
            // 创建 Scope.Invoker 管线范围
            var scope = Scope.Invoker;
            // 用于保存管线范围内的当前执行过滤器
            var state = (object?)null;
            var isCompleted = false;
 
            while (!isCompleted)
            {
                // 执行管线的下一个状态
                var lastTask = ResultNext<IAlwaysRunResultFilter, IAsyncAlwaysRunResultFilter>(ref next, ref scope, ref state, ref isCompleted);
                // 没有同步完成
                if (!lastTask.IsCompletedSuccessfully)
                {
                    // 切换为异步等待模式
                    // 返回 Task 用于等待管线范围完成
                    return Awaited(this, lastTask, next, scope, state, isCompleted);
                }
            }
 
            return Task.CompletedTask;
        }
        catch (Exception ex)
        {
            return Task.FromException(ex);
        }
 
        static async Task Awaited(ResourceInvoker invoker, Task lastTask, State next, Scope scope, object? state, bool isCompleted)
        {
            await lastTask;
 
            while (!isCompleted)
            {
                await invoker.ResultNext<IAlwaysRunResultFilter, IAsyncAlwaysRunResultFilter>(ref next, ref scope, ref state, ref isCompleted);
            }
        }
    }

    // 执行所有结果过滤器管线方法
    // 实现方式与 InvokeAlwaysRunResultFilters 方法类似
    private Task InvokeResultFilters()
    {
        try
        {
            var next = State.ResultBegin;
            var scope = Scope.Invoker;
            var state = (object?)null;
            var isCompleted = false;
 
            while (!isCompleted)
            {
                var lastTask = ResultNext<IResultFilter, IAsyncResultFilter>(ref next, ref scope, ref state, ref isCompleted);
                if (!lastTask.IsCompletedSuccessfully)
                {
                    return Awaited(this, lastTask, next, scope, state, isCompleted);
                }
            }
 
            return Task.CompletedTask;
        }
        catch (Exception ex)
        {
            return Task.FromException(ex);
        }
 
        static async Task Awaited(ResourceInvoker invoker, Task lastTask, State next, Scope scope, object? state, bool isCompleted)
        {
            await lastTask;
 
            while (!isCompleted)
            {
                await invoker.ResultNext<IResultFilter, IAsyncResultFilter>(ref next, ref scope, ref state, ref isCompleted);
            }
        }
    }

    // 执行结果过滤器管线范围内的下一个管线执行状态
    private Task ResultNext<TFilter, TFilterAsync>(ref State next, ref Scope scope, ref object? state, ref bool isCompleted)
        where TFilter : class, IResultFilter
        where TFilterAsync : class, IAsyncResultFilter
    {
        var resultFilterKind = typeof(TFilter) == typeof(IAlwaysRunResultFilter) ?
            FilterTypeConstants.AlwaysRunResultFilter :
            FilterTypeConstants.ResultFilter;
 
        switch (next)
        {
            case State.ResultBegin:
                {
                    // 重置过滤器游标
                    _cursor.Reset();
                    // 跳转到 State.ResultNext
                    // 开始结果过滤器管线
                    goto case State.ResultNext;
                }
 
            case State.ResultNext:
                {
                    // 得到结果过滤器集合并将游标移动到下一个过滤器
                    var current = _cursor.GetNextFilter<TFilter, TFilterAsync>();
                    if (current.FilterAsync != null)
                    {
                        if (_resultExecutingContext == null)
                        {
                            // 创建 ResultExecutingContext 并封装全局 IActionResult 执行结果
                            _resultExecutingContext = new ResultExecutingContextSealed(_actionContext, _filters, _result!, _instance!);
                        }
 
                        state = current.FilterAsync;
                        // 跳转到 State.ResultAsyncBegin
                        goto case State.ResultAsyncBegin;
                    }
                    else if (current.Filter != null)
                    {
                        if (_resultExecutingContext == null)
                        {
                            _resultExecutingContext = new ResultExecutingContextSealed(_actionContext, _filters, _result!, _instance!);
                        }
 
                        state = current.Filter;
                        // 跳转到 State.ResultSyncBegin
                        goto case State.ResultSyncBegin;
                    }
                    else
                    {
                        // 结果过滤器流入管线完成
                        // 跳转到 State.ResultInside
                        goto case State.ResultInside;
                    }
                }
 
            case State.ResultAsyncBegin:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_resultExecutingContext != null);
 
                    var filter = (TFilterAsync)state;
                    var resultExecutingContext = _resultExecutingContext;
 
                    _diagnosticListener.BeforeOnResultExecution(resultExecutingContext, filter);
                    _logger.BeforeExecutingMethodOnFilter(
                        resultFilterKind,
                        nameof(IAsyncResultFilter.OnResultExecutionAsync),
                        filter);

                    // 执行 IAsyncResultFilter.OnResultExecutionAsync 方法
                    // 将 InvokeNextResultFilterAwaitedAsync 方法包装为 ResultExecutionDelegate 委托传入，可以使用异步方式执行后续结果过滤器管线
                    var task = filter.OnResultExecutionAsync(resultExecutingContext, InvokeNextResultFilterAwaitedAsync<TFilter, TFilterAsync>);
                    // 没有同步完成
                    if (!task.IsCompletedSuccessfully)
                    {
                        // 设置管线范围内的下一个管线执行状态为 State.ResultAsyncEnd
                        next = State.ResultAsyncEnd;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }

                    // 跳转到 State.ResultAsyncEnd
                    goto case State.ResultAsyncEnd;
                }
 
            case State.ResultAsyncEnd:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_resultExecutingContext != null);
 
                    var filter = (TFilterAsync)state;
                    var resultExecutingContext = _resultExecutingContext;
                    var resultExecutedContext = _resultExecutedContext;
 
                    // 如果在结果过滤器流入管线中被取消或进入结果过滤器流出管线之前没有正常完成
                    if (resultExecutedContext == null || resultExecutingContext.Cancel)
                    {
                        _logger.ResultFilterShortCircuited(filter);
                        _resultExecutedContext = new ResultExecutedContextSealed(
                            _actionContext,
                            _filters,
                            resultExecutingContext.Result,
                            _instance!)
                        {
                            // 标记结果过滤器管线已被取消
                            Canceled = true,
                        };
                    }
 
                    _diagnosticListener.AfterOnResultExecution(_resultExecutedContext!, filter);
                    _logger.AfterExecutingMethodOnFilter(
                        resultFilterKind,
                        nameof(IAsyncResultFilter.OnResultExecutionAsync),
                        filter);

                    // 跳转到 State.ResultEnd
                    goto case State.ResultEnd;
                }
 
            case State.ResultSyncBegin:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_resultExecutingContext != null);
 
                    var filter = (TFilter)state;
                    var resultExecutingContext = _resultExecutingContext;
 
                    _diagnosticListener.BeforeOnResultExecuting(resultExecutingContext, filter);
                    _logger.BeforeExecutingMethodOnFilter(
                        resultFilterKind,
                        nameof(IResultFilter.OnResultExecuting),
                        filter);

                    // 执行 IResultFilter.OnResultExecuting 方法
                    filter.OnResultExecuting(resultExecutingContext);
 
                    _diagnosticListener.AfterOnResultExecuting(resultExecutingContext, filter);
                    _logger.AfterExecutingMethodOnFilter(
                        resultFilterKind,
                        nameof(IResultFilter.OnResultExecuting),
                        filter);

                    // 如果在结果过滤器流入管线中被取消
                    if (_resultExecutingContext.Cancel)
                    {
                        _logger.ResultFilterShortCircuited(filter);
                        
                        // 标记结果过滤器管线已被取消
                        _resultExecutedContext = new ResultExecutedContextSealed(
                            resultExecutingContext,
                            _filters,
                            resultExecutingContext.Result,
                            _instance!)
                        {
                            Canceled = true,
                        };
 
                        goto case State.ResultEnd;
                    }
 
                    // 执行后续结果过滤器管线范围
                    var task = InvokeNextResultFilterAsync<TFilter, TFilterAsync>();
                    // 没有同步完成
                    if (!task.IsCompletedSuccessfully)
                    {
                        // 设置管线范围内的下一个管线执行状态为 State.ResultSyncEnd
                        next = State.ResultSyncEnd;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }

                    // 跳转到 State.ResultSyncEnd
                    goto case State.ResultSyncEnd;
                }
 
            case State.ResultSyncEnd:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_resultExecutingContext != null);
                    Debug.Assert(_resultExecutedContext != null);
 
                    var filter = (TFilter)state;
                    var resultExecutedContext = _resultExecutedContext;
 
                    _diagnosticListener.BeforeOnResultExecuted(resultExecutedContext, filter);
                    _logger.BeforeExecutingMethodOnFilter(
                        resultFilterKind,
                        nameof(IResultFilter.OnResultExecuted),
                        filter);

                    // 执行 IResultFilter.OnResultExecuted 方法
                    filter.OnResultExecuted(resultExecutedContext);
 
                    _diagnosticListener.AfterOnResultExecuted(resultExecutedContext, filter);
                    _logger.AfterExecutingMethodOnFilter(
                        resultFilterKind,
                        nameof(IResultFilter.OnResultExecuted),
                        filter);

                    // 跳转到 State.ResultEnd
                    goto case State.ResultEnd;
                }
 
            case State.ResultInside:
                {
                    // 尝试从 ResultExecutingContext 中得到 IActionResult 执行结果
                    if (_resultExecutingContext != null)
                    {
                        _result = _resultExecutingContext.Result;
                    }

                    // 如果全局 IActionResult 不存在，则创建一个 EmptyResult
                    if (_result == null)
                    {
                        _result = new EmptyResult();
                    }
 
                    // 执行 IActionResult
                    var task = InvokeResultAsync(_result);
                    // 没有同步完成
                    if (!task.IsCompletedSuccessfully)
                    {
                        // 设置管线范围内的下一个管线执行状态为 State.ResultEnd
                        next = State.ResultEnd;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }
 
                    goto case State.ResultEnd;
                }
 
            case State.ResultEnd:
                {
                    var result = _result;
                    // 设置当前管线范围结束标志
                    isCompleted = true;
 
                    if (scope == Scope.Result)
                    {
                        if (_resultExecutedContext == null)
                        {
                            _resultExecutedContext = new ResultExecutedContextSealed(_actionContext, _filters, result!, _instance!);
                        }
 
                        return Task.CompletedTask;
                    }
 
                    Rethrow(_resultExecutedContext!);
                    return Task.CompletedTask;
                }
 
            default:
                throw new InvalidOperationException();
        }
    }

    // 结果过滤器后续管线入口方法
    private Task<ResultExecutedContext> InvokeNextResultFilterAwaitedAsync<TFilter, TFilterAsync>()
        where TFilter : class, IResultFilter
        where TFilterAsync : class, IAsyncResultFilter
    {
        Debug.Assert(_resultExecutingContext != null);
        // 当设置了 ResultExecutingContext.Cancel 属性为 true 时，就表示需要短路管线
        // 如果此时继续调用 ResultExecutionDelegate 委托执行后续结果过滤器管线将会抛出异常
        if (_resultExecutingContext.Cancel)
        {
            return Throw();
        }

        // 执行后续结果过滤器管线范围
        var task = InvokeNextResultFilterAsync<TFilter, TFilterAsync>();
        // 没有同步完成
        if (!task.IsCompletedSuccessfully)
        {
            // 切换为异步等待模式
            // 返回管线范围完成情况的 Task<ResultExecutedContext>
            return Awaited(this, task);
        }
 
        Debug.Assert(_resultExecutedContext != null);
        return Task.FromResult<ResultExecutedContext>(_resultExecutedContext);
 
        static async Task<ResultExecutedContext> Awaited(ResourceInvoker invoker, Task task)
        {
            await task;
 
            Debug.Assert(invoker._resultExecutedContext != null);
            return invoker._resultExecutedContext;
        }
#pragma warning disable CS1998
        static async Task<ResultExecutedContext> Throw()
        {
            var message = Resources.FormatAsyncResultFilter_InvalidShortCircuit(
                typeof(IAsyncResultFilter).Name,
                nameof(ResultExecutingContext.Cancel),
                typeof(ResultExecutingContext).Name,
                typeof(ResultExecutionDelegate).Name);
 
            throw new InvalidOperationException(message);
        }
#pragma warning restore CS1998
    }

    // 子类重写该方法
    // 实现动作过滤器管线范围的执行
    protected abstract Task InvokeInnerFilterAsync();
}
```

- ControllerActionInvoker

```C#
// 基于控制器的 IActionInvoker 实现
internal partial class ControllerActionInvoker : ResourceInvoker, IActionInvoker
{
    private readonly ControllerActionInvokerCacheEntry _cacheEntry;
    private readonly ControllerContext _controllerContext;
 
    private Dictionary<string, object?>? _arguments;
 
    private ActionExecutingContextSealed? _actionExecutingContext;
    private ActionExecutedContextSealed? _actionExecutedContext;
 
    internal ControllerActionInvoker(
        ILogger logger,
        DiagnosticListener diagnosticListener,
        IActionContextAccessor actionContextAccessor,
        IActionResultTypeMapper mapper,
        ControllerContext controllerContext,
        ControllerActionInvokerCacheEntry cacheEntry,
        IFilterMetadata[] filters)
        : base(diagnosticListener, logger, actionContextAccessor, mapper, controllerContext, filters, controllerContext.ValueProviderFactories)
    {
        ArgumentNullException.ThrowIfNull(cacheEntry);
 
        _cacheEntry = cacheEntry;
        _controllerContext = controllerContext;
    }

    // 动作过滤器管线范围入口方法
    protected override Task InvokeInnerFilterAsync()
    {
        try
        {
            // 设置管线范围下一个管线执行状态为 State.ActionBegin
            var next = State.ActionBegin;
            // 创建 Scope.Invoker 管线范围
            var scope = Scope.Invoker;
            // 用于保存管线范围内的当前执行过滤器
            var state = (object?)null;
            var isCompleted = false;
 
            while (!isCompleted)
            {
                // 执行管线的下一个状态
                var lastTask = Next(ref next, ref scope, ref state, ref isCompleted);
                // 没有同步完成
                if (!lastTask.IsCompletedSuccessfully)
                {
                    // 切换为异步等待模式
                    // 返回 Task 用于等待管线范围完成
                    return Awaited(this, lastTask, next, scope, state, isCompleted);
                }
            }
 
            return Task.CompletedTask;
        }
        catch (Exception ex)
        {
            return Task.FromException(ex);
        }
 
        static async Task Awaited(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, object? state, bool isCompleted)
        {
            await lastTask;
 
            while (!isCompleted)
            {
                await invoker.Next(ref next, ref scope, ref state, ref isCompleted);
            }
        }
    }

    // 执行动作过滤器管线范围内的下一个管线执行状态
    private Task Next(ref State next, ref Scope scope, ref object? state, ref bool isCompleted)
    {
        switch (next)
        {
            case State.ActionBegin:
                {
                    var controllerContext = _controllerContext;
 
                    _cursor.Reset();
                    Log.ExecutingControllerFactory(_logger, controllerContext);

                    // 创建控制器实例
                    // 默认使用 ActivatorUtilities.CreateInstance 方法创建控制器实例，控制器构造函数的依赖服务可以由 ActionContext.HttpContext.RequestServices 属性表示的范围容器提供
                    // 如果调用 IMvcBuilder.AddControllersAsServices 扩展方法注册 IControllerActivator 服务，则控制器实例直接由 ActionContext.HttpContext.RequestServices 属性表示的范围容器提供
                    _instance = _cacheEntry.ControllerFactory(controllerContext);
                    Log.ExecutedControllerFactory(_logger, controllerContext);
 
                    _arguments = new Dictionary<string, object?>(StringComparer.OrdinalIgnoreCase);

                    // 参数绑定
                    var task = BindArgumentsAsync();
                    // 没有同步完成
                    if (task.Status != TaskStatus.RanToCompletion)
                    {
                        // 设置当前管道范围下一个管线执行状态为 State.ActionNext
                        next = State.ActionNext;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }

                    // 跳转到 State.ActionNext
                    goto case State.ActionNext;
                }
 
            case State.ActionNext:
                {
                    // 得到动作过滤器集合并将游标移动到下一个过滤器
                    var current = _cursor.GetNextFilter<IActionFilter, IAsyncActionFilter>();
                    if (current.FilterAsync != null)
                    {
                        if (_actionExecutingContext == null)
                        {
                            _actionExecutingContext = new ActionExecutingContextSealed(_controllerContext, _filters, _arguments!, _instance!);
                        }
 
                        state = current.FilterAsync;
                        // 跳转到 State.ActionAsyncBegin
                        // 以异步方式执行动作过滤器
                        goto case State.ActionAsyncBegin;
                    }
                    else if (current.Filter != null)
                    {
                        if (_actionExecutingContext == null)
                        {
                            _actionExecutingContext = new ActionExecutingContextSealed(_controllerContext, _filters, _arguments!, _instance!);
                        }
 
                        state = current.Filter;
                        // 跳转到 State.ActionSyncBegin
                        // 以同步方式执行动作过滤器
                        goto case State.ActionSyncBegin;
                    }
                    else
                    {
                        // 跳转到 State.ActionInside
                        // 动作过滤器流入管线完成
                        goto case State.ActionInside;
                    }
                }
 
            case State.ActionAsyncBegin:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_actionExecutingContext != null);
 
                    var filter = (IAsyncActionFilter)state;
                    var actionExecutingContext = _actionExecutingContext;
 
                    _diagnosticListener.BeforeOnActionExecution(actionExecutingContext, filter);
                    _logger.BeforeExecutingMethodOnFilter(
                        MvcCoreLoggerExtensions.ActionFilter,
                        nameof(IAsyncActionFilter.OnActionExecutionAsync),
                        filter);

                    // 执行 IAsyncActionFilter.OnActionExecutionAsync 方法
                    // 将 InvokeNextActionFilterAwaitedAsync 方法包装为 ActionExecutionDelegate 委托传入，可以使用异步方式执行后续动作过滤器管线
                    var task = filter.OnActionExecutionAsync(actionExecutingContext, InvokeNextActionFilterAwaitedAsync);
                    // 没有同步完成
                    if (task.Status != TaskStatus.RanToCompletion)
                    {
                        // 设置当前管道范围下一个管线执行状态为 State.ActionAsyncEnd
                        next = State.ActionAsyncEnd;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }

                    // 跳转到 State.ActionAsyncEnd
                    goto case State.ActionAsyncEnd;
                }
 
            case State.ActionAsyncEnd:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_actionExecutingContext != null);
 
                    var filter = (IAsyncActionFilter)state;
 
                    if (_actionExecutedContext == null)
                    {
                        _logger.ActionFilterShortCircuited(filter);
 
                        _actionExecutedContext = new ActionExecutedContextSealed(
                            _controllerContext,
                            _filters,
                            _instance!)
                        {
                            Canceled = true,
                            Result = _actionExecutingContext.Result,
                        };
                    }
 
                    _diagnosticListener.AfterOnActionExecution(_actionExecutedContext, filter);
                    _logger.AfterExecutingMethodOnFilter(
                        MvcCoreLoggerExtensions.ActionFilter,
                        nameof(IAsyncActionFilter.OnActionExecutionAsync),
                        filter);

                    // 跳转到 State.ActionEnd
                    goto case State.ActionEnd;
                }
 
            case State.ActionSyncBegin:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_actionExecutingContext != null);
 
                    var filter = (IActionFilter)state;
                    var actionExecutingContext = _actionExecutingContext;
 
                    _diagnosticListener.BeforeOnActionExecuting(actionExecutingContext, filter);
                    _logger.BeforeExecutingMethodOnFilter(
                        MvcCoreLoggerExtensions.ActionFilter,
                        nameof(IActionFilter.OnActionExecuting),
                        filter);

                    // 执行 IActionFilter.OnActionExecuting 方法
                    filter.OnActionExecuting(actionExecutingContext);
 
                    _diagnosticListener.AfterOnActionExecuting(actionExecutingContext, filter);
                    _logger.AfterExecutingMethodOnFilter(
                        MvcCoreLoggerExtensions.ActionFilter,
                        nameof(IActionFilter.OnActionExecuting),
                        filter);
                    
                    // 如果 ActionExecutingContext.Result 属性不为 null，则表示短路管线
                    if (actionExecutingContext.Result != null)
                    {
                        _logger.ActionFilterShortCircuited(filter);
 
                        _actionExecutedContext = new ActionExecutedContextSealed(
                            _actionExecutingContext,
                            _filters,
                            _instance!)
                        {
                            Canceled = true,
                            Result = _actionExecutingContext.Result,
                        };

                        // 跳转到 State.ActionEnd
                        goto case State.ActionEnd;
                    }

                    // 执行动作过滤器后续管线范围
                    var task = InvokeNextActionFilterAsync();
                    // 没有同步完成
                    if (task.Status != TaskStatus.RanToCompletion)
                    {
                        // 设置当前管道范围下一个管线执行状态为 State.ActionSyncEnd
                        next = State.ActionSyncEnd;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }

                    // 跳转到 State.ActionSyncEnd
                    goto case State.ActionSyncEnd;
                }
 
            case State.ActionSyncEnd:
                {
                    Debug.Assert(state != null);
                    Debug.Assert(_actionExecutingContext != null);
                    Debug.Assert(_actionExecutedContext != null);
 
                    var filter = (IActionFilter)state;
                    var actionExecutedContext = _actionExecutedContext;
 
                    _diagnosticListener.BeforeOnActionExecuted(actionExecutedContext, filter);
                    _logger.BeforeExecutingMethodOnFilter(
                        MvcCoreLoggerExtensions.ActionFilter,
                        nameof(IActionFilter.OnActionExecuted),
                        filter);

                    // 执行 IActionFilter.OnActionExecuted 方法
                    filter.OnActionExecuted(actionExecutedContext);
 
                    _diagnosticListener.AfterOnActionExecuted(actionExecutedContext, filter);
                    _logger.AfterExecutingMethodOnFilter(
                        MvcCoreLoggerExtensions.ActionFilter,
                        nameof(IActionFilter.OnActionExecuted),
                        filter);

                    // 跳转到 State.ActionEnd
                    goto case State.ActionEnd;
                }
 
            case State.ActionInside:
                {   
                    // 执行控制器动作方法
                    var task = InvokeActionMethodAsync();
                    // 没有同步完成
                    if (task.Status != TaskStatus.RanToCompletion)
                    {
                        // 设置当前管道范围下一个管线执行状态为 State.ActionEnd
                        next = State.ActionEnd;
                        // 返回 Task 用于等待管线范围完成
                        return task;
                    }

                    // 跳转到 State.ActionEnd
                    goto case State.ActionEnd;
                }
 
            case State.ActionEnd:
                {
                    // 如果当前处于 Scope.Action 管线范围
                    if (scope == Scope.Action)
                    {
                        if (_actionExecutedContext == null)
                        {
                            _actionExecutedContext = new ActionExecutedContextSealed(_controllerContext, _filters, _instance!)
                            {
                                // 将动作方法执行结果赋值给 ActionExecutedContext.Result 属性
                                Result = _result,
                            };
                        }
 
                        // 设置当前管线范围结束标志
                        isCompleted = true;
                        return Task.CompletedTask;
                    }
 
                    var actionExecutedContext = _actionExecutedContext;
                    Rethrow(actionExecutedContext);
 
                    if (actionExecutedContext != null)
                    {
                        _result = actionExecutedContext.Result;
                    }
 
                    isCompleted = true;
                    return Task.CompletedTask;
                }
 
            default:
                throw new InvalidOperationException();
        }
    }
    
    // 动作过滤器后续管线入口方法
    private Task InvokeNextActionFilterAsync()
    {
        try
        {
            // 设置管线范围下一个管线执行状态为 State.ActionNext
            var next = State.ActionNext;
            // 用于保存管线范围内的当前执行过滤器
            var state = (object?)null;
            // 创建 Scope.Action 管线范围
            var scope = Scope.Action;
            var isCompleted = false;
            while (!isCompleted)
            {
                // 执行管线的下一个状态
                var lastTask = Next(ref next, ref scope, ref state, ref isCompleted);
                // 没有同步完成
                if (!lastTask.IsCompletedSuccessfully)
                {
                    // 切换为异步等待模式
                    // 返回 Task 用于等待管线范围完成
                    return Awaited(this, lastTask, next, scope, state, isCompleted);
                }
            }
        }
        catch (Exception exception)
        {
            _actionExecutedContext = new ActionExecutedContextSealed(_controllerContext, _filters, _instance!)
            {
                ExceptionDispatchInfo = ExceptionDispatchInfo.Capture(exception),
            };
        }
 
        Debug.Assert(_actionExecutedContext != null);
        return Task.CompletedTask;
 
        static async Task Awaited(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, object? state, bool isCompleted)
        {
            try
            {
                await lastTask;
 
                while (!isCompleted)
                {
                    await invoker.Next(ref next, ref scope, ref state, ref isCompleted);
                }
            }
            catch (Exception exception)
            {
                invoker._actionExecutedContext = new ActionExecutedContextSealed(invoker._controllerContext, invoker._filters, invoker._instance!)
                {
                    ExceptionDispatchInfo = ExceptionDispatchInfo.Capture(exception),
                };
            }
 
            Debug.Assert(invoker._actionExecutedContext != null);
        }
    }
    
    // 执行动作方法
    private Task InvokeActionMethodAsync()
    {
        if (_diagnosticListener.IsEnabled() || _logger.IsEnabled(LogLevel.Trace))
        {
            return Logged(this);
        }
 
        var objectMethodExecutor = _cacheEntry.ObjectMethodExecutor;
        var actionMethodExecutor = _cacheEntry.ActionMethodExecutor;
        var orderedArguments = PrepareArguments(_arguments, objectMethodExecutor);

        // 无论动作方法的返回值类型是什么
        // 最终都会被封装为一个 ValueTask<IActionResult> 类型的对象
        var actionResultValueTask = actionMethodExecutor.Execute(ControllerContext, _mapper, objectMethodExecutor, _instance!, orderedArguments);
        if (actionResultValueTask.IsCompletedSuccessfully)
        {
            // 同步完成
            // 直接提取 IActionResult
            _result = actionResultValueTask.Result;
        }
        else
        {
            // 切换为异步等待模式
            // 返回 Task 用于等待管线范围完成
            return Awaited(this, actionResultValueTask);
        }
 
        return Task.CompletedTask;
 
        static async Task Awaited(ControllerActionInvoker invoker, ValueTask<IActionResult> actionResultValueTask)
        {
            invoker._result = await actionResultValueTask;
        }
    }
}
```

## 过滤器

- ProblemDetails

```C#
// 表示问题的详细信息
// 基于 RFC 7807 和 RFC 9110 规范
public class ProblemDetails
{
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    [JsonPropertyOrder(-5)]
    public string? Type { get; set; }
 
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    [JsonPropertyOrder(-4)]
    public string? Title { get; set; }
 
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    [JsonPropertyOrder(-3)]
    public int? Status { get; set; }
 
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    [JsonPropertyOrder(-2)]
    public string? Detail { get; set; }
 
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    [JsonPropertyOrder(-1)]
    public string? Instance { get; set; }
 
    [JsonExtensionData]
    public IDictionary<string, object?> Extensions { get; set; } = new Dictionary<string, object?>(StringComparer.Ordinal);
}
```

- ProblemDetailsClientErrorFactory

```C#
// 提供客户端错误信息的 IActionResult 工厂
internal sealed class ProblemDetailsClientErrorFactory : IClientErrorFactory
{
    private readonly ProblemDetailsFactory _problemDetailsFactory;
 
    public ProblemDetailsClientErrorFactory(ProblemDetailsFactory problemDetailsFactory)
    {
        _problemDetailsFactory = problemDetailsFactory ?? throw new ArgumentNullException(nameof(problemDetailsFactory));
    }
    
    // 返回实现 IActionResult 的 ObjectResult 类型对象
    // 用于提供基于 RFC 7807 格式的问题详细消息给客户端
    public IActionResult GetClientError(ActionContext actionContext, IClientErrorActionResult clientError)
    {
        // 利用 ProblemDetails 工厂创建 ProblemDetails 对象
        var problemDetails = _problemDetailsFactory.CreateProblemDetails(actionContext.HttpContext, clientError.StatusCode);
 
        return new ObjectResult(problemDetails)
        {
            StatusCode = problemDetails.Status,
            ContentTypes =
                {
                    "application/problem+json",
                    "application/problem+xml",
                },
        };
    }
}
```

- DefaultProblemDetailsFactory

```C#
// ProblemDetailsFactory 的子类
// 用于创建 ProblemDetails 和 ValidationProblemDetails 对象
internal sealed class DefaultProblemDetailsFactory : ProblemDetailsFactory
{
    private readonly ApiBehaviorOptions _options;
    private readonly Action<ProblemDetailsContext>? _configure;
 
    public DefaultProblemDetailsFactory(
        IOptions<ApiBehaviorOptions> options,
        IOptions<ProblemDetailsOptions>? problemDetailsOptions = null)
    {
        _options = options?.Value ?? throw new ArgumentNullException(nameof(options));
        _configure = problemDetailsOptions?.Value?.CustomizeProblemDetails;
    }
 
    // 提供 500 错误的问题详细消息
    public override ProblemDetails CreateProblemDetails(
        HttpContext httpContext,
        int? statusCode = null,
        string? title = null,
        string? type = null,
        string? detail = null,
        string? instance = null)
    {
        statusCode ??= 500;
 
        var problemDetails = new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Type = type,
            Detail = detail,
            Instance = instance,
        };
 
        ApplyProblemDetailsDefaults(httpContext, problemDetails, statusCode.Value);
 
        return problemDetails;
    }
    
    // 提供 400 错误的问题详细消息
    public override ValidationProblemDetails CreateValidationProblemDetails(
        HttpContext httpContext,
        ModelStateDictionary modelStateDictionary,
        int? statusCode = null,
        string? title = null,
        string? type = null,
        string? detail = null,
        string? instance = null)
    {
        ArgumentNullException.ThrowIfNull(modelStateDictionary);
 
        statusCode ??= 400;
 
        var problemDetails = new ValidationProblemDetails(modelStateDictionary)
        {
            Status = statusCode,
            Type = type,
            Detail = detail,
            Instance = instance,
        };
 
        if (title != null)
        {
            problemDetails.Title = title;
        }
 
        ApplyProblemDetailsDefaults(httpContext, problemDetails, statusCode.Value);
 
        return problemDetails;
    }
    
    // 根据状态码应用基于 RFC 9110 规范的默认值
    private void ApplyProblemDetailsDefaults(HttpContext httpContext, ProblemDetails problemDetails, int statusCode)
    {
        problemDetails.Status ??= statusCode;
 
        // 检查是否在 ApiBehaviorOptions 中配置了默认的基于状态码的客户端错误消息映射
        // 如果存在则应用默认的 Title 和 Type 属性值
        if (_options.ClientErrorMapping.TryGetValue(statusCode, out var clientErrorData))
        {
            problemDetails.Title ??= clientErrorData.Title;
            problemDetails.Type ??= clientErrorData.Link;
        }

        // 如果利用 Activity 开启了针对请求范围的活动跟踪，则使用 Activity.Id 填充 traceId 扩展属性，用于问题跟踪
        var traceId = Activity.Current?.Id ?? httpContext?.TraceIdentifier;
        if (traceId != null)
        {
            problemDetails.Extensions["traceId"] = traceId;
        }

        // 最后应用 ProblemDetailsOptions 选项中委托执行自定义配置
        _configure?.Invoke(new() { HttpContext = httpContext!, ProblemDetails = problemDetails });
    }
}
```

- IFilterMetadata

```C#
// 标记过滤器的空接口
public interface IFilterMetadata
{
}
```

- IOrderedFilter

```C#
// 排序过滤器接口
public interface IOrderedFilter : IFilterMetadata
{
    // 过滤器顺序
    int Order { get; }
}
```

- IFilterFactory

```C#
// 过滤器工厂接口
public interface IFilterFactory : IFilterMetadata
{
    // 指示创建的过滤器实例是否可以被缓存重用
    bool IsReusable { get; }
    
    // 创建过滤器实例
    IFilterMetadata CreateInstance(IServiceProvider serviceProvider);
}
```

- TypeFilterAttribute

```C#
// 基于类型的过滤器工厂
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = true, Inherited = true)]
public class TypeFilterAttribute : Attribute, IFilterFactory, IOrderedFilter
{
    private ObjectFactory? _factory;
 
    public TypeFilterAttribute(Type type)
    {
        ImplementationType = type ?? throw new ArgumentNullException(nameof(type));
    }
 
    // 支持外部传入过滤器构造函数的参数
    public object[]? Arguments { get; set; }
    
    // 实际过滤器的类型
    public Type ImplementationType { get; }
    
    // 默认创建的过滤器顺序为 0
    public int Order { get; set; }
 
    // 指示创建的过滤器实例是否可以被缓存重用
    public bool IsReusable { get; set; }
 
    // 反射创建过滤器实例，并可以利用 HttpContext.RequestServices 属性表示的范围容器提供过滤器构造函数的依赖服务
    public IFilterMetadata CreateInstance(IServiceProvider serviceProvider)
    {
        ArgumentNullException.ThrowIfNull(serviceProvider);
 
        if (_factory == null)
        {
            var argumentTypes = Arguments?.Select(a => a.GetType())?.ToArray();
            _factory = ActivatorUtilities.CreateFactory(ImplementationType, argumentTypes ?? Type.EmptyTypes);
        }
 
        var filter = (IFilterMetadata)_factory(serviceProvider, Arguments);
        if (filter is IFilterFactory filterFactory)
        {
            filter = filterFactory.CreateInstance(serviceProvider);
        }
 
        return filter;
    }
}
```

- ClientErrorResultFilterFactory

```C#
// 客户端错误结果消息过滤器工厂
internal sealed class ClientErrorResultFilterFactory : IFilterFactory, IOrderedFilter
{
    public int Order => ClientErrorResultFilter.FilterOrder;
 
    public bool IsReusable => true;
    
    // 反射创建过滤器实例
    // 并可以利用 HttpContext.RequestServices 属性表示的范围容器提供过滤器构造函数的参数
    public IFilterMetadata CreateInstance(IServiceProvider serviceProvider)
    {
        var resultFilter = ActivatorUtilities.CreateInstance<ClientErrorResultFilter>(serviceProvider);
        return resultFilter;
    }
}
```

- ClientErrorResultFilter

```C#
// 客户端错误结果消息过滤器
// 用于在流入管线短路时作为 IAlwaysRunResultFilter 结果过滤器执行
internal sealed partial class ClientErrorResultFilter : IAlwaysRunResultFilter, IOrderedFilter
{
    internal const int FilterOrder = -2000;
    private readonly IClientErrorFactory _clientErrorFactory;
    private readonly ILogger<ClientErrorResultFilter> _logger;
 
    public ClientErrorResultFilter(
        IClientErrorFactory clientErrorFactory,
        ILogger<ClientErrorResultFilter> logger)
    {
        _clientErrorFactory = clientErrorFactory ?? throw new ArgumentNullException(nameof(clientErrorFactory));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }
 
    public int Order => FilterOrder;
 
    public void OnResultExecuted(ResultExecutedContext context)
    {
    }
 
    public void OnResultExecuting(ResultExecutingContext context)
    {
        ArgumentNullException.ThrowIfNull(context);

        // 只负责处理 ResultExecutingContext.Result 是实现了 IClientErrorActionResult 的 IActionResult
        if (!(context.Result is IClientErrorActionResult clientError))
        {
            return;
        }
 
        // 只处理 400 和 500 的状态码
        if (clientError.StatusCode < 400)
        {
            return;
        }

        // 得到封装了问题详细消息的 ObjectResult
        var result = _clientErrorFactory.GetClientError(context, clientError);
        if (result == null)
        {
            return;
        }
 
        Log.TransformingClientError(_logger, context.Result.GetType(), result.GetType(), clientError.StatusCode);
        context.Result = result;
    }
 
    private static partial class Log
    {
        [LoggerMessage(49, LogLevel.Trace, "Replacing {InitialActionResultType} with status code {StatusCode} with {ReplacedActionResultType}.", EventName = "ClientErrorResultFilter")]
        public static partial void TransformingClientError(ILogger logger, Type initialActionResultType, Type replacedActionResultType, int? statusCode);
    }
}
```

- ModelStateInvalidFilterFactory

```C#
// 无效模型状态过滤器工厂
internal sealed class ModelStateInvalidFilterFactory : IFilterFactory, IOrderedFilter
{
    public int Order => ModelStateInvalidFilter.FilterOrder;
 
    public bool IsReusable => true;
 
    // 创建实际过滤器实例
    public IFilterMetadata CreateInstance(IServiceProvider serviceProvider)
    {
        var options = serviceProvider.GetRequiredService<IOptions<ApiBehaviorOptions>>();
        var loggerFactory = serviceProvider.GetRequiredService<ILoggerFactory>();
 
        return new ModelStateInvalidFilter(options.Value, loggerFactory.CreateLogger<ModelStateInvalidFilter>());
    }
}
```

- ModelStateInvalidFilter

```C#
// 无效模型状态过滤器
public partial class ModelStateInvalidFilter : IActionFilter, IOrderedFilter
{
    internal const int FilterOrder = -2000;
 
    private readonly ApiBehaviorOptions _apiBehaviorOptions;
    private readonly ILogger _logger;
 
    public ModelStateInvalidFilter(ApiBehaviorOptions apiBehaviorOptions, ILogger logger)
    {
        _apiBehaviorOptions = apiBehaviorOptions ?? throw new ArgumentNullException(nameof(apiBehaviorOptions));
        // 如果 ApiBehaviorOptions.SuppressModelStateInvalidFilter 为 false，则必须存在无效模型问题响应消息工厂
        if (!_apiBehaviorOptions.SuppressModelStateInvalidFilter && _apiBehaviorOptions.InvalidModelStateResponseFactory == null)
        {
            throw new ArgumentException(Resources.FormatPropertyOfTypeCannotBeNull(
                typeof(ApiBehaviorOptions),
                nameof(ApiBehaviorOptions.InvalidModelStateResponseFactory)));
        }
 
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }
 
    public int Order => FilterOrder;
 
    public bool IsReusable => true;
 
    public void OnActionExecuting(ActionExecutingContext context)
    {
        // 如果模型状态无效
        if (context.Result == null && !context.ModelState.IsValid)
        {
            Log.ModelStateInvalidFilterExecuting(_logger);
            // 利用无效模型问题响应消息工厂创建响应 BadRequestObjectResult 对象
            context.Result = _apiBehaviorOptions.InvalidModelStateResponseFactory(context);
        }
    }
 
    private static partial class Log
    {
        [LoggerMessage(1, LogLevel.Debug, "The request has model state errors, returning an error response.", EventName = "ModelStateInvalidFilterExecuting")]
        public static partial void ModelStateInvalidFilterExecuting(ILogger logger);
    }
}
```
