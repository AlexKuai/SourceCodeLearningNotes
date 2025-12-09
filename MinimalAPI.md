# MinimalAPI

## 源码涉及的核心类型

- WebApplication  
- WebApplicationOptions  
- HostApplicationBuilderSettings  
- IHostApplicationBuilder  
- HostApplicationBuilder  
- WebApplicationBuilder  
- BootstrapHostBuilder  
- ConfigureHostBuilder  
- ConfigureWebHostBuilder  

## Minimal API 编程模型

- WebApplication

```C#
// 表示 Web 应用程序
// 实现 IHost 并封装实际的 IHost 对象，实现的所有方法最终都会转移到实际的 IHost 上
// 实现 IApplicationBuilder 并封装实际的 IApplicationBuilder 对象，实现的所有方法最终都会转移到实际的 IApplicationBuilder 上
// 实现 IEndpointRouteBuilder 用来注册 EndpointDataSource
public sealed class WebApplication : IHost, IApplicationBuilder, IEndpointRouteBuilder, IAsyncDisposable
{
    internal const string GlobalEndpointRouteBuilderKey = "__GlobalEndpointRouteBuilder";

    private readonly IHost _host;
    private readonly List<EndpointDataSource> _dataSources = new();

    internal WebApplication(IHost host)
    {
        // 封装实际的 IHost 对象
        _host = host;
        // 创建实际的 IApplicationBuilder 对象
        ApplicationBuilder = new ApplicationBuilder(host.Services, ServerFeatures);
        // 创建日志类别为 {IWebHostEnvironment.ApplicationName} 或 "WebApplication" 的 ILogger
        Logger = host.Services.GetRequiredService<ILoggerFactory>().CreateLogger(Environment.ApplicationName ?? nameof(WebApplication));
 
        // 将自身以 "__GlobalEndpointRouteBuilder" 为 Key 添加到共享字典中
        Properties[GlobalEndpointRouteBuilderKey] = this;
    }

    // 从实际的 IHost 中返回表示根容器的 IServiceProvider
    public IServiceProvider Services => _host.Services;

    // 返回应用配置
    // 由 HostApplicationBuilder.Configuration 提供的 ConfigurationManager
    public IConfiguration Configuration => _host.Services.GetRequiredService<IConfiguration>();

    // 利用根容器得到 IWebHostEnvironment
    public IWebHostEnvironment Environment => _host.Services.GetRequiredService<IWebHostEnvironment>();

    // 利用根容器得到 IHostApplicationLifetime
    public IHostApplicationLifetime Lifetime => _host.Services.GetRequiredService<IHostApplicationLifetime>();

    // 返回日志类别为 {IWebHostEnvironment.ApplicationName} 或 "WebApplication" 的 ILogger
    public ILogger Logger { get; }

    // 返回服务器提供的特性集合中 IServerAddressesFeature.Addresses 表示的地址集合
    // 可以直接设置多个监听地址
    public ICollection<string> Urls => ServerFeatures.GetRequiredFeature<IServerAddressesFeature>().Addresses;

    // 返回实际的 IApplicationBuilder
    internal ApplicationBuilder ApplicationBuilder { get; }

    // 实现 IHost
    // 启动宿主，使用实际的 IHost 调用 StartAsync 方法
    public Task StartAsync(CancellationToken cancellationToken = default) =>
        _host.StartAsync(cancellationToken);

    // 实现 IHost
    // 停止宿主，使用实际的 IHost 调用 StopAsync 方法
    public Task StopAsync(CancellationToken cancellationToken = default) =>
        _host.StopAsync(cancellationToken);

    // 以异步方式启动宿主
    // 用于需要长时间运行的服务
    // 支持直接设置监听地址
    public Task RunAsync(string? url = null)
    {
        Listen(url);
        return HostingAbstractionsHostExtensions.RunAsync(this);
    }
    
    // 以同步方式启动宿主
    // 用于需要长时间运行的服务
    // 支持直接设置监听地址
    public void Run(string? url = null)
    {
        Listen(url);
        HostingAbstractionsHostExtensions.Run(this);
    }

    // 设置监听地址
    // 注意：
    // 每次只能设置一个地址，不能使用 ";" 分割设置多个地址
    // 如果需要设置多个地址，可以直接通过 Urls 属性设置
    private void Listen(string? url)
    {
        if (url is null)
        {
            return;
        }
 
        var addresses = ServerFeatures.Get<IServerAddressesFeature>()?.Addresses;
        if (addresses is null)
        {
            throw new InvalidOperationException($"Changing the URL is not supported because no valid {nameof(IServerAddressesFeature)} was found.");
        }
        if (addresses.IsReadOnly)
        {
            throw new InvalidOperationException($"Changing the URL is not supported because {nameof(IServerAddressesFeature.Addresses)} {nameof(ICollection<string>.IsReadOnly)}.");
        }

        // 每次设置都会清空地址集合并添加新的地址
        addresses.Clear();
        addresses.Add(url);
    }

    // 注意：
    // 由于 WebApplication 显式实现了除 Use 方法以外的所有 IApplicationBuilder 接口成员
    // 所以不能直接通过 WebApplication 调用除 Use 方法外的成员

    // 返回 ApplicationBuilder.Properties 表示的共享字典
    internal IDictionary<string, object?> Properties => ApplicationBuilder.Properties;
    // 显式实现 IApplicationBuilder.Properties
    IDictionary<string, object?> IApplicationBuilder.Properties => Properties;

    // 返回由服务器提供的特性集合
    internal IFeatureCollection ServerFeatures => _host.Services.GetRequiredService<IServer>().Features;
    // 显式实现 IApplicationBuilder.ServerFeatures
    IFeatureCollection IApplicationBuilder.ServerFeatures => ServerFeatures;

    // 返回 ApplicationBuilder.ApplicationServices 表示的根容器
    // 显式实现 IApplicationBuilder.ApplicationServices
    IServiceProvider IApplicationBuilder.ApplicationServices
    {
        get => ApplicationBuilder.ApplicationServices;
        set => ApplicationBuilder.ApplicationServices = value;
    }

    // 调用 ApplicationBuilder.Build 方法构建请求处理委托链，并返回位于头部的请求处理器
    internal RequestDelegate BuildRequestDelegate() => ApplicationBuilder.Build();
    // 显式实现 IApplicationBuilder.Build
    RequestDelegate IApplicationBuilder.Build() => BuildRequestDelegate();
 
    // 调用 ApplicationBuilder.New 方法创建新的 IApplicationBuilder
    // 显式实现 IApplicationBuilder.New
    IApplicationBuilder IApplicationBuilder.New()
    {
        var newBuilder = ApplicationBuilder.New();
        // 利用 ApplicationBuilder.New 创建的新的 IApplicationBuilder 时会复制内部的共享字典
        // 为了保证只存在一个代表全局 IEndpointRouteBuilder 的项，需要从新建的 IApplicationBuilder 的共享字典中删除以 "__GlobalEndpointRouteBuilder" 为 Key 的项
        newBuilder.Properties.Remove(GlobalEndpointRouteBuilderKey);
        return newBuilder;
    }
 
    // 实现 IApplicationBuilder.Use
    // 可以通过 WebApplication 直接调用 Use 方法注册中间件的原始形式
    public IApplicationBuilder Use(Func<RequestDelegate, RequestDelegate> middleware)
    {
        ApplicationBuilder.Use(middleware);
        return this;
    }

    // 注意：
    // 由于 WebApplication 显示实现了所有 IEndpointRouteBuilder 接口成员
    // 所以在程序集外部不能直接通过 WebApplication 调用对应的实现方法
    
    // 显式实现 IEndpointRouteBuilder.CreateApplicationBuilder
    IApplicationBuilder IEndpointRouteBuilder.CreateApplicationBuilder() => ((IApplicationBuilder)this).New();

    internal ICollection<EndpointDataSource> DataSources => _dataSources;
    // 显式实现 IEndpointRouteBuilder.DataSources
    ICollection<EndpointDataSource> IEndpointRouteBuilder.DataSources => DataSources;

    // 显式实现 IEndpointRouteBuilder.ServiceProvider
    IServiceProvider IEndpointRouteBuilder.ServiceProvider => Services;

    // 静态方法
    // 创建 WebApplicationBuilder 并调用 WebApplicationBuilder.Build 方法直接构建 WebApplication
    // 注意：
    // 调用此方法创建的 WebApplication 所有的配置和服务注册都使用默认设置
    public static WebApplication Create(string[]? args = null) =>
        new WebApplicationBuilder(new() { Args = args }).Build();
    
    // 静态方法
    // 创建 WebApplicationBuilder，并使用命令行参数
    public static WebApplicationBuilder CreateBuilder(string[] args) =>
        new(new() { Args = args });

    // 静态方法
    // 创建 WebApplicationBuilder，并使用 WebApplicationOptions 选项
    public static WebApplicationBuilder CreateBuilder(WebApplicationOptions options) =>
        new(options);

    // 静态方法
    // 创建 WebApplicationBuilder
    public static WebApplicationBuilder CreateBuilder() =>
        new(new());
}
```

- WebApplicationOptions  

```C#
// Web 应用程序选项
// 提供用于直接设置应用程序环境相关的属性
public class WebApplicationOptions
{
    public string[]? Args { get; init; }
 
    public string? EnvironmentName { get; init; }
 
    public string? ApplicationName { get; init; }
 
    public string? ContentRootPath { get; init; }

    public string? WebRootPath { get; init; }
}
```

- HostApplicationBuilderSettings  
  
```C#
// 宿主应用建造者选项
// 大部分属性值来源于 WebApplicationOptions
public sealed class HostApplicationBuilderSettings
{
    public HostApplicationBuilderSettings()
    {
    }
 
    public bool DisableDefaults { get; set; }

    public string[]? Args { get; set; }
 
    public ConfigurationManager? Configuration { get; set; }
 
    public string? EnvironmentName { get; set; }
 
    public string? ApplicationName { get; set; }
 
    public string? ContentRootPath { get; set; }
}
```

- IHostApplicationBuilder  

```C#
// 宿主应用建造者接口
public interface IHostApplicationBuilder
{
    // 构建过程中用于共享的数据的字典
    IDictionary<object, object> Properties { get; }
    
    // 返回 IConfigurationManager
    IConfigurationManager Configuration { get; }
    
    // 返回 IHostEnvironment
    IHostEnvironment Environment { get; }
 
    // 返回 ILoggingBuilder
    ILoggingBuilder Logging { get; }
 
    // 返回 IMetricsBuilder
    IMetricsBuilder Metrics { get; }
 
    // 返回 IServiceCollection
    IServiceCollection Services { get; }
    
    // 主要用于第三方 IServiceProvider 工厂创建容器建造者，并使用对应的容器建造者配置
    void ConfigureContainer<TContainerBuilder>(IServiceProviderFactory<TContainerBuilder> factory, Action<TContainerBuilder>? configure = null) where TContainerBuilder : notnull;
}
```

- HostApplicationBuilder  

```C#
// IHostApplicationBuilder 的默认实现
// 主要作用：
// 1. 构建默认的宿主和应用配置
// 2. 注册默认的服务
// 3. 使用第三方 IServiceProvider 工厂创建容器建造者，并使用对应的容器建造者配置
// 4. 构建实际的 IHost
// 5. 对外提供核心属性用来收集配置和注册服务
public sealed class HostApplicationBuilder : IHostApplicationBuilder
{
    private readonly HostBuilderContext _hostBuilderContext;
    private readonly ServiceCollection _serviceCollection = new();
    private readonly IHostEnvironment _environment;
    private readonly LoggingBuilder _logging;
    private readonly MetricsBuilder _metrics;
 
    // 创建 IServiceProvider 的工厂委托
    private Func<IServiceProvider> _createServiceProvider;
    // 用于容器建造者的配置委托，默认是一个空配置委托
    private Action<object> _configureContainer = _ => { };
    private HostBuilderAdapter? _hostBuilderAdapter;
 
    // 表示根容器的 IServiceProvider
    private IServiceProvider? _appServices;
    // 用于标记构建状态，防止重复调用 Build 方法
    private bool _hostBuilt;
 
    public HostApplicationBuilder()
        : this(args: null)
    {
    }
 
    public HostApplicationBuilder(string[]? args)
        : this(new HostApplicationBuilderSettings { Args = args })
    {
    }
 
    public HostApplicationBuilder(HostApplicationBuilderSettings? settings)
    {
        settings ??= new HostApplicationBuilderSettings();
        // 如果 HostApplicationBuilderSettings.Configuration 属性为 null，则创建 ConfigurationManager 用来收集配置源
        // 注意：
        // ConfigurationManager 使用的是一种边收集边构建配置的方式
        // 即每次添加新的 IConfigurationSource 都会立即触发构建对应的 IConfigurationProvider 用来提供配置
        Configuration = settings.Configuration ?? new ConfigurationManager();
 
        if (!settings.DisableDefaults)
        {
            // 确定配置中是否已经设置了工作目录路径
            if (settings.ContentRootPath is null && Configuration[HostDefaults.ContentRootKey] is null)
            {
                // 添加内存配置源用于设置工作目录路径（一般为应用程序根目录）
                HostingHostBuilderExtensions.SetDefaultContentRoot(Configuration);
            }

            // 添加 "DOTNET_" 前缀的环境变量配置源 
            Configuration.AddEnvironmentVariables(prefix: "DOTNET_");
        }

        // 执行初始化
        Initialize(settings, out _hostBuilderContext, out _environment, out _logging, out _metrics);
 
        ServiceProviderOptions? serviceProviderOptions = null;
        if (!settings.DisableDefaults)
        {
            // 添加默认应用配置源
            HostingHostBuilderExtensions.ApplyDefaultAppConfiguration(_hostBuilderContext, Configuration, settings.Args);
            // 注册默认服务
            HostingHostBuilderExtensions.AddDefaultServices(_hostBuilderContext, Services);
            // 根据环境创建 ServiceProviderOptions 选项
            serviceProviderOptions = HostingHostBuilderExtensions.CreateDefaultServiceProviderOptions(_hostBuilderContext);
        }
 
        // 创建默认的 IServiceProvider
        _createServiceProvider = () =>
        {
            // 默认情况下是一个空配置委托
            _configureContainer(Services);
            return serviceProviderOptions is null ? Services.BuildServiceProvider() : Services.BuildServiceProvide(serviceProviderOptions);
        };
    }
    
    // 初始化
    private void Initialize(HostApplicationBuilderSettings settings, out HostBuilderContext hostBuilderContext, out IHostEnvironment environment, out LoggingBuilder logging, out MetricsBuilder metrics)
    {
        // 添加命令行配置源
        HostingHostBuilderExtensions.AddCommandLineConfig(Configuration, settings.Args);
 
        // 添加内存配置源，利用 HostApplicationBuilderSettings 中的属性作为配置值
        // 添加 "applicationName" 配置节表示的应用程序名称
        // 添加 "environment" 配置节表示的环境名称
        // 添加 "contentRoot" 配置节表示的工作目录路径
        List<KeyValuePair<string, string?>>? optionList = null;
        if (settings.ApplicationName is not null)
        {
            optionList ??= new List<KeyValuePair<string, string?>>();
            optionList.Add(new KeyValuePair<string, string?>(HostDefaults.ApplicationKey, settings.ApplicationName));
        }
        if (settings.EnvironmentName is not null)
        {
            optionList ??= new List<KeyValuePair<string, string?>>();
            optionList.Add(new KeyValuePair<string, string?>(HostDefaults.EnvironmentKey, settings.EnvironmentName));
        }
        if (settings.ContentRootPath is not null)
        {
            optionList ??= new List<KeyValuePair<string, string?>>();
            optionList.Add(new KeyValuePair<string, string?>(HostDefaults.ContentRootKey, settings.ContentRootPath));
        }
        if (optionList is not null)
        {
            Configuration.AddInMemoryCollection(optionList);
        }

        // 利用配置创建 HostingEnvironment 和 PhysicalFileProvider
        (HostingEnvironment hostingEnvironment, PhysicalFileProvider physicalFileProvider) = HostBuilder.CreateHostingEnvironment(Configuration);

        // 向 IConfigurationBuilder.Properties 共享字典添加默认的 IFileProvider
        // 目的是为后续添加基于文件的配置源提供默认的 IFileProvider
        Configuration.SetFileProvider(physicalFileProvider);

        // 创建 HostBuilderContext
        hostBuilderContext = new HostBuilderContext(new Dictionary<object, object>())
        {
            HostingEnvironment = hostingEnvironment,
            Configuration = Configuration
        };
 
        environment = hostingEnvironment;

        // 注册默认服务
        HostBuilder.PopulateServiceCollection(
            Services,
            hostBuilderContext,
            hostingEnvironment,
            physicalFileProvider,
            Configuration,
            () => _appServices!);
 
        logging = new LoggingBuilder(Services);
        metrics = new MetricsBuilder(Services);
    }
 
    IDictionary<object, object> IHostApplicationBuilder.Properties => _hostBuilderContext.Properties;
 
    public IHostEnvironment Environment => _environment;
 
    public ConfigurationManager Configuration { get; }
 
    IConfigurationManager IHostApplicationBuilder.Configuration => Configuration;
 
    public IServiceCollection Services => _serviceCollection;
 
    public ILoggingBuilder Logging => _logging;
 
    public IMetricsBuilder Metrics => _metrics;
    
    // 使用第三方 IServiceProvider 工厂和对应的容器建造者配置
    public void ConfigureContainer<TContainerBuilder>(IServiceProviderFactory<TContainerBuilder> factory, Action<TContainerBuilder>? configure = null) where TContainerBuilder : notnull
    {
        // 定义 Func<IServiceProvider> 委托的原型方法
        _createServiceProvider = () =>
        {
            // 利用第三方 IServiceProvider 工厂创建容器建造者
            TContainerBuilder containerBuilder = factory.CreateBuilder(Services);
            // 配置容器建造者
            _configureContainer(containerBuilder);
            // 创建 IServiceProvider 并返回
            return factory.CreateServiceProvider(containerBuilder);
        };

        // 模拟适配器模式
        _configureContainer = containerBuilder => configure?.Invoke((TContainerBuilder)containerBuilder);
    }
 
    // 构建得到实际的 IHost
    public IHost Build()
    {
        if (_hostBuilt)
        {
            throw new InvalidOperationException(SR.BuildCalled);
        }
        _hostBuilt = true;
 
        using DiagnosticListener diagnosticListener = HostBuilder.LogHostBuilding(this);
        _hostBuilderAdapter?.ApplyChanges();

        // 利用 Func<IServiceProvider> 工厂委托创建表示根容器的 IServiceProvider
        _appServices = _createServiceProvider();
 
        // 将 ServiceCollection 标记为只读，不允许再添加服务注册
        _serviceCollection.MakeReadOnly();

        // 利用根容器创建实际的 IHost
        return HostBuilder.ResolveHost(_appServices, diagnosticListener);
    }
 
    private sealed class LoggingBuilder : ILoggingBuilder
    {
        public LoggingBuilder(IServiceCollection services)
        {
            Services = services;
        }
 
        public IServiceCollection Services { get; }
    }
 
    private sealed class MetricsBuilder(IServiceCollection services) : IMetricsBuilder
    {
        public IServiceCollection Services { get; } = services;
    }
}
```

- WebApplicationBuilder

```C#
// 表示 Web 应用程序建造者
// 实现 IHostApplicationBuilder 并封装实际的 HostApplicationBuilder 
public sealed class WebApplicationBuilder : IHostApplicationBuilder
{
    private const string EndpointRouteBuilderKey = "__EndpointRouteBuilder";
    private const string AuthenticationMiddlewareSetKey = "__AuthenticationMiddlewareSet";
    private const string AuthorizationMiddlewareSetKey = "__AuthorizationMiddlewareSet";
    private const string UseRoutingKey = "__UseRouting";

    // 实际的 IHostApplicationBuilder
    private readonly HostApplicationBuilder _hostApplicationBuilder;
    // 表示针对 GenericWebHostService 服务注册的 ServiceDescriptor
    private readonly ServiceDescriptor _genericWebHostServiceDescriptor;

    // 保存构建得到的 WebApplication
    private WebApplication? _builtApplication;

    internal WebApplicationBuilder(WebApplicationOptions options, Action<IHostBuilder>? configureDefaults = null)
    {
        // 创建 ConfigurationManager 用于收集和提供配置
        var configuration = new ConfigurationManager();
        
        // 在 WebApplication 编程模型下，最早添加的是以 "ASPNETCORE_" 为前缀的环境变量配置源
        configuration.AddEnvironmentVariables(prefix: "ASPNETCORE_");
 
        // 使用 HostApplicationBuilderSettings 创建 HostApplicationBuilder
        _hostApplicationBuilder = new HostApplicationBuilder(new HostApplicationBuilderSettings
        {
            Args = options.Args,
            ApplicationName = options.ApplicationName,
            EnvironmentName = options.EnvironmentName,
            ContentRootPath = options.ContentRootPath,
            Configuration = configuration
        });
 
        // 添加内存配置源
        // 添加 "webroot" 配置节表示的 Web 目录路径
        if (options.WebRootPath is not null)
        {
            Configuration.AddInMemoryCollection(new[]
            {
                new KeyValuePair<string, string?>(WebHostDefaults.WebRootKey, options.WebRootPath),
            });
        }
 
        // 创建 BootstrapHostBuilder
        var bootstrapHostBuilder = new BootstrapHostBuilder(_hostApplicationBuilder);

        // 测试方法
        // 使用 BootstrapHostBuilder 收集配置和注册服务
        // 注意：
        // 虽然 BootstrapHostBuilder 实现了 IHostBuilder，但内部很多功能已经不支持
        configureDefaults?.Invoke(bootstrapHostBuilder);
 
        // 调用 IHostBuilder.ConfigureWebHostDefaults 扩展方法添加默认配置源和注册默认服务
        bootstrapHostBuilder.ConfigureWebHostDefaults(
            webHostBuilder =>
            {
                // 添加针对中间件建造者的配置
                webHostBuilder.Configure(ConfigureApplication);

                // 为 IWebHostBuilder 的备用配置添加环境配置信息
                InitializeWebHostSettings(webHostBuilder);
            },
            options =>
            {
                // 禁用环境变量配置源
                // 因为以 "ASPNETCORE_" 为前缀的环境变量配置源已经添加，无需重复添加
                options.SuppressEnvironmentConfiguration = true;
            }
        );

        // 得到表示 GenericWebHostService 服务注册的 ServiceDescriptor
        _genericWebHostServiceDescriptor = InitializeHosting(bootstrapHostBuilder);
    }

    // 通过 IWebHostBuilder.UseSetting 方法为 IWebHostBuilder 的备用配置添加环境配置信息
    private void InitializeWebHostSettings(IWebHostBuilder webHostBuilder)
    {
        webHostBuilder.UseSetting(WebHostDefaults.ApplicationKey, _hostApplicationBuilder.Environment.ApplicationName ?? "");
        webHostBuilder.UseSetting(WebHostDefaults.PreventHostingStartupKey, Configuration[WebHostDefaults.PreventHostingStartupKey]);
        webHostBuilder.UseSetting(WebHostDefaults.HostingStartupAssembliesKey, Configuration[WebHostDefaults.HostingStartupAssembliesKey]);
        webHostBuilder.UseSetting(WebHostDefaults.HostingStartupExcludeAssembliesKey, Configuration[WebHostDefaults.HostingStartupExcludeAssembliesKey]);
    }
    
    // 应用 BootstrapHostBuilder 收集配置和服务注册
    private ServiceDescriptor InitializeHosting(BootstrapHostBuilder bootstrapHostBuilder)
    {
        // 收集配置和注册服务
        var genericWebHostServiceDescriptor = bootstrapHostBuilder.RunDefaultCallbacks();
 
        // 从共享字典中得到 WebHostBuilderContext
        var webHostContext = (WebHostBuilderContext)bootstrapHostBuilder.Properties[typeof(WebHostBuilderContext)];
        Environment = webHostContext.HostingEnvironment;
 
        // 创建 ConfigureHostBuilder
        // 对外提供 IHostBuilder 用来收集配置和注册服务
        Host = new ConfigureHostBuilder(bootstrapHostBuilder.Context, Configuration, Services);
        // 创建 ConfigureWebHostBuilder
        // 对外提供 IWebHostBuilder 用来收集配置和注册服务
        WebHost = new ConfigureWebHostBuilder(webHostContext, Configuration, Services);
 
        return genericWebHostServiceDescriptor;
    }

    public IWebHostEnvironment Environment { get; private set; }
 
    public IServiceCollection Services => _hostApplicationBuilder.Services;
 
    public ConfigurationManager Configuration => _hostApplicationBuilder.Configuration;
 
    public ILoggingBuilder Logging => _hostApplicationBuilder.Logging;
 
    public IMetricsBuilder Metrics => _hostApplicationBuilder.Metrics;
 
    public ConfigureWebHostBuilder WebHost { get; private set; }
 
    public ConfigureHostBuilder Host { get; private set; }
 
    IDictionary<object, object> IHostApplicationBuilder.Properties => ((IHostApplicationBuilder)_hostApplicationBuilder).Properties;
 
    IConfigurationManager IHostApplicationBuilder.Configuration => Configuration;
 
    IHostEnvironment IHostApplicationBuilder.Environment => Environment;

    // 构建 WebApplication
    public WebApplication Build()
    {
        // 重新添加表示 GenericWebHostService 的服务注册
        _hostApplicationBuilder.Services.Add(_genericWebHostServiceDescriptor);
        // 应用第三方 IServiceProvider 工厂和对应的容器建造者配置
        Host.ApplyServiceProviderFactory(_hostApplicationBuilder);
        // 构建实际的 IHost 并创建 WebApplication
        _builtApplication = new WebApplication(_hostApplicationBuilder.Build());
        return _builtApplication;
    }

    // 中间件建造者配置方法
    private void ConfigureApplication(WebHostBuilderContext context, IApplicationBuilder app) =>
        ConfigureApplication(context, app, allowDeveloperExceptionPage: true);
    
    // 中间件建造者配置方法
    // 注意：
    // 方法调用时候的 IApplicationBuilder 实参是由 GenericWebHostService 通过 IApplicationBuilderFactory 创建的
    private void ConfigureApplication(WebHostBuilderContext context, IApplicationBuilder app, bool allowDeveloperExceptionPage)
    {
        Debug.Assert(_builtApplication is not null);
 
        // 如果存在以 "__EndpointRouteBuilder" 为 Key 的项，则将其移除
        // 因为在 Minimal API 编程模型中，WebApplication 扮演着全局 IEndpointRouteBuilder 的角色
        if (app.Properties.TryGetValue(EndpointRouteBuilderKey, out var priorRouteBuilder))
        {
            app.Properties.Remove(EndpointRouteBuilderKey);
        }
 
        // 开发环境添加开发者异常页中间件
        if (allowDeveloperExceptionPage && context.HostingEnvironment.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        // 将 WebApplication 以 "__GlobalEndpointRouteBuilder" 为 Key 添加到当前 IApplicationBuilder 的共享字典中作为全局 IEndpointRouteBuilder
        app.Properties.Add(WebApplication.GlobalEndpointRouteBuilderKey, _builtApplication);

        // 如果利用 WebApplication 作为 IEndpointRouteBuilder 已经注册了 EndpointDataSource
        if (_builtApplication.DataSources.Count > 0)
        {
            // 检查是否已经利用 WebApplication 作为 IApplicationBuilder 调用了 IApplicationBuilder.UseRouting 扩展方法注册 EndpointRoutingMiddleware 中间件
            if (!_builtApplication.Properties.TryGetValue(EndpointRouteBuilderKey, out var localRouteBuilder))
            {
                // 没有则使用当前 IApplicationBuilder 调用 IApplicationBuilder.UseRouting 扩展方法注册 EndpointRoutingMiddleware 中间件
                app.UseRouting();
                _builtApplication.Properties[UseRoutingKey] = app.Properties[UseRoutingKey];
            }
            else
            {
                // 有则只需将对应的 IEndpointRouteBuilder 添加到当前 IApplicationBuilder 的共享字典中
                app.Properties[EndpointRouteBuilderKey] = localRouteBuilder;
            }
        }
 
        // 得到 IServiceProviderIsService 服务
        // 注意：
        // 如果使用默认的依赖注入框架，则实际返回 CallSiteFactory
        var serviceProviderIsService = _builtApplication.Services.GetService<IServiceProviderIsService>();
        // 检查是否注册了 IAuthenticationSchemeProvider 服务
        if (serviceProviderIsService?.IsService(typeof(IAuthenticationSchemeProvider)) is true)
        {
            // 检查是否使用 WebApplication 作为 IApplicationBuilder 时调用 IApplicationBuilder.UseAuthentication 扩展方法注册 AuthenticationMiddleware 中间件
            if (!_builtApplication.Properties.ContainsKey(AuthenticationMiddlewareSetKey))
            {
                // 没有则需要使用当前 IApplicationBuilder 调用 IApplicationBuilder.UseAuthentication 扩展方法注册 AuthenticationMiddleware 中间件
                _builtApplication.Properties[AuthenticationMiddlewareSetKey] = true;
                app.UseAuthentication();
            }
        }

        // 检查是否注册了 IAuthorizationHandlerProvider 服务
        if (serviceProviderIsService?.IsService(typeof(IAuthorizationHandlerProvider)) is true)
        {
            // 检查是否使用 WebApplication 作为 IApplicationBuilder 时调用 IApplicationBuilder.UseAuthorization 扩展方法注册 AuthorizationMiddleware 中间件
            if (!_builtApplication.Properties.ContainsKey(AuthorizationMiddlewareSetKey))
            {
                // 没有则需要使用当前 IApplicationBuilder 调用 IApplicationBuilder.UseAuthorization 扩展方法注册 AuthorizationMiddleware 中间件
                _builtApplication.Properties[AuthorizationMiddlewareSetKey] = true;
                app.UseAuthorization();
            }
        }
 
        // 注册导线中间件
        var wireSourcePipeline = new WireSourcePipeline(_builtApplication);
        app.Use(wireSourcePipeline.CreateMiddleware);

        // 如果利用 WebApplication 作为 IEndpointRouteBuilder 注册了 EndpointDataSource
        if (_builtApplication.DataSources.Count > 0)
        {
            // 使用当前 IApplicationBuilder 调用 IApplicationBuilder.UseEndpoints 扩展方法注册 EndpointMiddleware 中间件
            // 注意：
            // 此处的目的是确保注册 EndpointMiddleware 中间件，并且传入一个空的配置委托，表示不再需要通过 IEndpointRouteBuilder 注册 EndpointDataSource
            app.UseEndpoints(_ => { });
        }
 
        MergeMiddlewareDescriptions(app);

        // 将 WebApplication 在扮演 IApplicationBuilder 时利用共享字典收集的数据转移到当前 IApplicationBuilder 的共享字典中
        foreach (var item in _builtApplication.Properties)
        {
            app.Properties[item.Key] = item.Value;
        }
 
        // 移除以 "__GlobalEndpointRouteBuilder" 为 Key 的项
        app.Properties.Remove(WebApplication.GlobalEndpointRouteBuilderKey);
 
        if (priorRouteBuilder is not null)
        {
            app.Properties[EndpointRouteBuilderKey] = priorRouteBuilder;
        }
    }

    // 使用第三方 IServiceProvider 工厂和对应的容器建造者配置
    void IHostApplicationBuilder.ConfigureContainer<TContainerBuilder>(IServiceProviderFactory<TContainerBuilder> factory, Action<TContainerBuilder>? configure) =>
        // 直接调用 HostApplicationBuilder.ConfigureContainer 方法
        _hostApplicationBuilder.ConfigureContainer(factory, configure);

    // 管道导线
    // 作用是构建请求处理委托链用于连接到其他管道上
    private sealed class WireSourcePipeline(IApplicationBuilder builtApplication)
    {
        private readonly IApplicationBuilder _builtApplication = builtApplication;
 
        public RequestDelegate CreateMiddleware(RequestDelegate next)
        {
            // 短路操作，目的是在构建请求处理委托链时丢弃自己的末端处理器（一般为 404 处理器），使请求能够直接传递到外部传入的请求处理委托链
            _builtApplication.Run(next);
            return _builtApplication.Build();
        }
    }
}
```

- BootstrapHostBuilder

```C#
// 实现 IHostBuilder
// 在 Minimal API 编程模型中 BootstrapHostBuilder 承担 IHostBuilder 的作用，但只提供部分功能
// 保留功能：
// 1. 用来收集针对宿主的配置
// 2. 用来收集针对应用的配置
// 3. 用来收集针对服务注册的配置
internal sealed class BootstrapHostBuilder : IHostBuilder
{
    private readonly HostApplicationBuilder _builder;
 
    private readonly List<Action<IConfigurationBuilder>> _configureHostActions = new();
    private readonly List<Action<HostBuilderContext, IConfigurationBuilder>> _configureAppActions = new();
    private readonly List<Action<HostBuilderContext, IServiceCollection>> _configureServicesActions = new();
 
    public BootstrapHostBuilder(HostApplicationBuilder builder)
    {
        _builder = builder;

        // 从服务注册描述符中确保取出 HostBuilderContext
        foreach (var descriptor in _builder.Services)
        {
            if (descriptor.ServiceType == typeof(HostBuilderContext))
            {
                Context = (HostBuilderContext)descriptor.ImplementationInstance!;
                break;
            }
        }

        if (Context is null)
        {
            throw new InvalidOperationException($"{nameof(HostBuilderContext)} must exist in the {nameof(IServiceCollection)}");
        }
    }
 
    public IDictionary<object, object> Properties => Context.Properties;
 
    public HostBuilderContext Context { get; }
    
    // 收集针对宿主的配置
    public IHostBuilder ConfigureHostConfiguration(Action<IConfigurationBuilder> configureDelegate)
    {
        _configureHostActions.Add(configureDelegate ?? throw new ArgumentNullException(nameof(configureDelegate)));
        return this;
    }
    
    // 收集针对应用的配置
    public IHostBuilder ConfigureAppConfiguration(Action<HostBuilderContext, IConfigurationBuilder> configureDelegate)
    {
        _configureAppActions.Add(configureDelegate ?? throw new ArgumentNullException(nameof(configureDelegate)));
        return this;
    }
    
    // 收集针对服务注册的配置
    public IHostBuilder ConfigureServices(Action<HostBuilderContext, IServiceCollection> configureDelegate)
    {
        _configureServicesActions.Add(configureDelegate ?? throw new ArgumentNullException(nameof(configureDelegate)));
        return this;
    }
    
    // 不支持
    public IHost Build()
    {
        throw new InvalidOperationException();
    }
 
    // 不支持
    public IHostBuilder ConfigureContainer<TContainerBuilder>(Action<HostBuilderContext, TContainerBuilder> configureDelegate)
    {
        throw new InvalidOperationException();
    }
 
    // 不支持
    public IHostBuilder UseServiceProviderFactory<TContainerBuilder>(IServiceProviderFactory<TContainerBuilder> factory) where TContainerBuilder : notnull
    {
        throw new InvalidOperationException();
    }
    
    // 不支持
    public IHostBuilder UseServiceProviderFactory<TContainerBuilder>(Func<HostBuilderContext, IServiceProviderFactory<TContainerBuilder>> factory) where TContainerBuilder : notnull
    {
        throw new InvalidOperationException();
    }
    
    // 执行默认回调
    // 收集配置和注册服务并返回表示 GenericWebHostService 的服务注册
    public ServiceDescriptor RunDefaultCallbacks()
    {
        // 注意：
        // 这里收集的宿主配置是通过调用 IHostBuilder.ConfigureWebHostDefaults 扩展方法时添加的
        // 在 WebApplication 编程模型中属于优先级比较高的配置
        foreach (var configureHostAction in _configureHostActions)
        {
            configureHostAction(_builder.Configuration);
        }
 
        foreach (var configureAppAction in _configureAppActions)
        {
            configureAppAction(Context, _builder.Configuration);
        }
 
        foreach (var configureServicesAction in _configureServicesActions)
        {
            configureServicesAction(Context, _builder.Services);
        }
 
        ServiceDescriptor? genericWebHostServiceDescriptor = null;
 
        // 确保 GenericWebHostService 的服务注册存在
        for (int i = _builder.Services.Count - 1; i >= 0; i--)
        {
            var descriptor = _builder.Services[i];
            if (descriptor.ServiceType == typeof(IHostedService))
            {
                Debug.Assert(descriptor.ImplementationType?.Name == "GenericWebHostService");
 
                genericWebHostServiceDescriptor = descriptor;
                // 找到后先暂时移除该服务注册，后续会在 WebApplicationBuild.Build 方法中重新添加对应的服务注册
                _builder.Services.RemoveAt(i);
                break;
            }
        }
 
        return genericWebHostServiceDescriptor ?? throw new InvalidOperationException($"GenericWebHostedService must exist in the {nameof(IServiceCollection)}");
    }
}
```

- ConfigureHostBuilder

```C#
// 用于提供 IHostBuilder 编程模型
public sealed class ConfigureHostBuilder : IHostBuilder, ISupportsConfigureWebHost
{
    private readonly ConfigurationManager _configuration;
    private readonly IServiceCollection _services;
    private readonly HostBuilderContext _context;
 
    private readonly List<Action<HostBuilderContext, object>> _configureContainerActions = new();
    private IServiceProviderFactory<object>? _serviceProviderFactory;
 
    internal ConfigureHostBuilder(
        HostBuilderContext context,
        ConfigurationManager configuration,
        IServiceCollection services)
    {
        _configuration = configuration;
        _services = services;
        _context = context;
    }
 
    public IDictionary<object, object> Properties => _context.Properties;
 
    // 不支持
    IHost IHostBuilder.Build()
    {
        throw new NotSupportedException($"Call {nameof(WebApplicationBuilder)}.{nameof(WebApplicationBuilder.Build)}() instead.");
    }

    // 添加针对宿主的配置
    public IHostBuilder ConfigureHostConfiguration(Action<IConfigurationBuilder> configureDelegate)
    {
        var previousApplicationName = _configuration[HostDefaults.ApplicationKey];
        var previousContentRoot = HostingPathResolver.ResolvePath(_context.HostingEnvironment.ContentRootPath);
        var previousContentRootConfig = _configuration[HostDefaults.ContentRootKey];
        var previousEnvironment = _configuration[HostDefaults.EnvironmentKey];
 
        // 执行配置收集
        configureDelegate(_configuration);
 
        // 注意：
        // 确保收集的配置没有更改应用程序名称、工作目录路径和环境名称

        if (!string.Equals(previousApplicationName, _configuration[HostDefaults.ApplicationKey], StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The application name changed from \"{previousApplicationName}\" to \"{_configuration[HostDefaults.ApplicationKey]}\". Changing the host configuration using WebApplicationBuilder.Host is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
 
        if (!string.Equals(previousContentRootConfig, _configuration[HostDefaults.ContentRootKey], StringComparison.OrdinalIgnoreCase)
            && !string.Equals(previousContentRoot, HostingPathResolver.ResolvePath(_configuration[HostDefaults.ContentRootKey]), StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The content root changed from \"{previousContentRoot}\" to \"{HostingPathResolver.ResolvePath(_configuration[HostDefaults.ContentRootKey])}\". Changing the host configuration using WebApplicationBuilder.Host is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
 
        if (!string.Equals(previousEnvironment, _configuration[HostDefaults.EnvironmentKey], StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The environment changed from \"{previousEnvironment}\" to \"{_configuration[HostDefaults.EnvironmentKey]}\". Changing the host configuration using WebApplicationBuilder.Host is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
 
        return this;
    }

    // 添加针对应用的配置
    public IHostBuilder ConfigureAppConfiguration(Action<HostBuilderContext, IConfigurationBuilder> configureDelegate)
    {
        // 执行配置收集
        configureDelegate(_context, _configuration);
        return this;
    }

    // 添加注册服务
    public IHostBuilder ConfigureServices(Action<HostBuilderContext, IServiceCollection> configureDelegate)
    {
        // 执行服务注册
        configureDelegate(_context, _services);
        return this;
    }
 
    // 添加针对容器建造者的配置
    public IHostBuilder ConfigureContainer<TContainerBuilder>(Action<HostBuilderContext, TContainerBuilder> configureDelegate)
    {
        ArgumentNullException.ThrowIfNull(configureDelegate);
 
        _configureContainerActions.Add((context, containerBuilder) => configureDelegate(context, (TContainerBuilder)containerBuilder));
 
        return this;
    }
 
    // 使用第三方 IServiceProvider 工厂
    public IHostBuilder UseServiceProviderFactory<TContainerBuilder>(IServiceProviderFactory<TContainerBuilder> factory) where TContainerBuilder : notnull
    {
        ArgumentNullException.ThrowIfNull(factory);
 
        _serviceProviderFactory = new ServiceProviderFactoryAdapter<TContainerBuilder>(factory);
        return this;
    }
 
    // 使用第三方 IServiceProvider 工厂
    public IHostBuilder UseServiceProviderFactory<TContainerBuilder>(Func<HostBuilderContext, IServiceProviderFactory<TContainerBuilder>> factory) where TContainerBuilder : notnull
    {
        // 利用工厂委托得到第三方 IServiceProvider 工厂
        return UseServiceProviderFactory(factory(_context));
    }
    
    // 不支持
    IHostBuilder ISupportsConfigureWebHost.ConfigureWebHost(Action<IWebHostBuilder> configure, Action<WebHostBuilderOptions> configureOptions)
    {
        throw new NotSupportedException("ConfigureWebHost() is not supported by WebApplicationBuilder.Host. Use the WebApplication returned by WebApplicationBuilder.Build() instead.");
    }
 
    // 应用第三方 IServiceProvider 工厂和对应的容器建造者配置
    internal void ApplyServiceProviderFactory(HostApplicationBuilder hostApplicationBuilder)
    {
        // 如果没有使用第三方 IServiceProvider 工厂，则直接应用容器建造者配置并返回
        // 注意：
        // 默认 IServiceProvider 工厂的容器建造者类型就是 IServiceCollection，所以本质就是针对 IServiceCollection 进行配置
        if (_serviceProviderFactory is null)
        {
            foreach (var action in _configureContainerActions)
            {
                action(_context, _services);
            }
 
            return;
        }
 
        void ConfigureContainerBuilderAdapter(object containerBuilder)
        {
            foreach (var action in _configureContainerActions)
            {
                action(_context, containerBuilder);
            }
        }
 
        hostApplicationBuilder.ConfigureContainer(_serviceProviderFactory, ConfigureContainerBuilderAdapter);
    }
    
    // IServiceProviderFactory<object> 默认实现
    // 用于以适配器模式封装第三方 IServiceProvider 工厂
    private sealed class ServiceProviderFactoryAdapter<TContainerBuilder> : IServiceProviderFactory<object> where TContainerBuilder : notnull
    {
        private readonly IServiceProviderFactory<TContainerBuilder> _serviceProviderFactory;
 
        public ServiceProviderFactoryAdapter(IServiceProviderFactory<TContainerBuilder> serviceProviderFactory)
        {
            _serviceProviderFactory = serviceProviderFactory;
        }
 
        public object CreateBuilder(IServiceCollection services) => _serviceProviderFactory.CreateBuilder(services);
        public IServiceProvider CreateServiceProvider(object containerBuilder) => _serviceProviderFactory.CreateServiceProvider((TContainerBuilder)containerBuilder);
    }
}
```

- ConfigureWebHostBuilder

```C#
// 用于提供 IWebHostBuilder 编程模型
public sealed class ConfigureWebHostBuilder : IWebHostBuilder, ISupportsStartup
{
    private readonly IWebHostEnvironment _environment;
    private readonly ConfigurationManager _configuration;
    private readonly IServiceCollection _services;
    private readonly WebHostBuilderContext _context;
 
    internal ConfigureWebHostBuilder(WebHostBuilderContext webHostBuilderContext, ConfigurationManager configuration, IServiceCollection services)
    {
        _configuration = configuration;
        _environment = webHostBuilderContext.HostingEnvironment;
        _services = services;
        _context = webHostBuilderContext;
    }
    
    // 不支持
    IWebHost IWebHostBuilder.Build()
    {
        throw new NotSupportedException($"Call {nameof(WebApplicationBuilder)}.{nameof(WebApplicationBuilder.Build)}() instead.");
    }
 
    // 添加针对应用的配置
    public IWebHostBuilder ConfigureAppConfiguration(Action<WebHostBuilderContext, IConfigurationBuilder> configureDelegate)
    {
        var previousContentRoot = HostingPathResolver.ResolvePath(_context.HostingEnvironment.ContentRootPath);
        var previousContentRootConfig = _configuration[WebHostDefaults.ContentRootKey];
        var previousWebRoot = HostingPathResolver.ResolvePath(_context.HostingEnvironment.WebRootPath, previousContentRoot);
        var previousWebRootConfig = _configuration[WebHostDefaults.WebRootKey];
        var previousApplication = _configuration[WebHostDefaults.ApplicationKey];
        var previousEnvironment = _configuration[WebHostDefaults.EnvironmentKey];
        var previousHostingStartupAssemblies = _configuration[WebHostDefaults.HostingStartupAssembliesKey];
        var previousHostingStartupAssembliesExclude = _configuration[WebHostDefaults.HostingStartupExcludeAssembliesKey];
 
        // 执行配置收集
        configureDelegate(_context, _configuration);
 
        // 注意：
        // 确保收集的配置没有更改 Web 目录路径、应用程序名称、工作目录路径、环境名称、托管启动程序集列表和托管启动程序集排除列表
        
        if (!string.Equals(previousWebRootConfig, _configuration[WebHostDefaults.WebRootKey], StringComparison.OrdinalIgnoreCase)
            && !string.Equals(HostingPathResolver.ResolvePath(previousWebRoot, previousContentRoot), HostingPathResolver.ResolvePath(_configuration[WebHostDefaults.WebRootKey], previousContentRoot), StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The web root changed from \"{HostingPathResolver.ResolvePath(previousWebRoot, previousContentRoot)}\" to \"{HostingPathResolver.ResolvePath(_configuration[WebHostDefaults.WebRootKey], previousContentRoot)}\". Changing the host configuration using WebApplicationBuilder.WebHost is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
        else if (!string.Equals(previousApplication, _configuration[WebHostDefaults.ApplicationKey], StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The application name changed from \"{previousApplication}\" to \"{_configuration[WebHostDefaults.ApplicationKey]}\". Changing the host configuration using WebApplicationBuilder.WebHost is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
        else if (!string.Equals(previousContentRootConfig, _configuration[WebHostDefaults.ContentRootKey], StringComparison.OrdinalIgnoreCase)
            && !string.Equals(previousContentRoot, HostingPathResolver.ResolvePath(_configuration[WebHostDefaults.ContentRootKey]), StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The content root changed from \"{previousContentRoot}\" to \"{HostingPathResolver.ResolvePath(_configuration[WebHostDefaults.ContentRootKey])}\". Changing the host configuration using WebApplicationBuilder.WebHost is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
        else if (!string.Equals(previousEnvironment, _configuration[WebHostDefaults.EnvironmentKey], StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The environment changed from \"{previousEnvironment}\" to \"{_configuration[WebHostDefaults.EnvironmentKey]}\". Changing the host configuration using WebApplicationBuilder.WebHost is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
        else if (!string.Equals(previousHostingStartupAssemblies, _configuration[WebHostDefaults.HostingStartupAssembliesKey], StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The hosting startup assemblies changed from \"{previousHostingStartupAssemblies}\" to \"{_configuration[WebHostDefaults.HostingStartupAssembliesKey]}\". Changing the host configuration using WebApplicationBuilder.WebHost is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
        else if (!string.Equals(previousHostingStartupAssembliesExclude, _configuration[WebHostDefaults.HostingStartupExcludeAssembliesKey], StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The hosting startup assemblies exclude list changed from \"{previousHostingStartupAssembliesExclude}\" to \"{_configuration[WebHostDefaults.HostingStartupExcludeAssembliesKey]}\". Changing the host configuration using WebApplicationBuilder.WebHost is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
 
        return this;
    }
 
    // 添加注册服务
    public IWebHostBuilder ConfigureServices(Action<WebHostBuilderContext, IServiceCollection> configureServices)
    {
        // 执行服务注册
        configureServices(_context, _services);
        return this;
    }
 
    // 添加注册服务
    public IWebHostBuilder ConfigureServices(Action<IServiceCollection> configureServices)
    {
        // 本质是调用重载的 ConfigureServices 方法
        return ConfigureServices((WebHostBuilderContext context, IServiceCollection services) => configureServices(services));
    }
 
    // 直接从 ConfigurationManager 获取对应配置节的值
    public string? GetSetting(string key)
    {
        return _configuration[key];
    }
 
    // 直接向 ConfigurationManager 添加或更新对应配置节的值
    public IWebHostBuilder UseSetting(string key, string? value)
    {
        if (value is null)
        {
            return this;
        }
 
        var previousContentRoot = HostingPathResolver.ResolvePath(_context.HostingEnvironment.ContentRootPath);
        var previousWebRoot = HostingPathResolver.ResolvePath(_context.HostingEnvironment.WebRootPath);
        var previousApplication = _configuration[WebHostDefaults.ApplicationKey];
        var previousEnvironment = _configuration[WebHostDefaults.EnvironmentKey];
        var previousHostingStartupAssemblies = _configuration[WebHostDefaults.HostingStartupAssembliesKey];
        var previousHostingStartupAssembliesExclude = _configuration[WebHostDefaults.HostingStartupExcludeAssembliesKey];

        // 注意：
        // 不支持更改 Web 目录路径、应用程序名称、工作目录路径、环境名称、托管启动程序集列表和托管启动程序集排除列表

        if (string.Equals(key, WebHostDefaults.WebRootKey, StringComparison.OrdinalIgnoreCase) &&
                !string.Equals(HostingPathResolver.ResolvePath(previousWebRoot, previousContentRoot), HostingPathResolver.ResolvePath(value, previousContentRoot), StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The web root changed from \"{HostingPathResolver.ResolvePath(previousWebRoot, previousContentRoot)}\" to \"{HostingPathResolver.ResolvePath(value, previousContentRoot)}\". Changing the host configuration using WebApplicationBuilder.WebHost is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
        else if (string.Equals(key, WebHostDefaults.ApplicationKey, StringComparison.OrdinalIgnoreCase) &&
                !string.Equals(previousApplication, value, StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The application name changed from \"{previousApplication}\" to \"{value}\". Changing the host configuration using WebApplicationBuilder.WebHost is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
        else if (string.Equals(key, WebHostDefaults.ContentRootKey, StringComparison.OrdinalIgnoreCase) &&
                !string.Equals(previousContentRoot, HostingPathResolver.ResolvePath(value), StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The content root changed from \"{previousContentRoot}\" to \"{HostingPathResolver.ResolvePath(value)}\". Changing the host configuration using WebApplicationBuilder.WebHost is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
        else if (string.Equals(key, WebHostDefaults.EnvironmentKey, StringComparison.OrdinalIgnoreCase) &&
                !string.Equals(previousEnvironment, value, StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The environment changed from \"{previousEnvironment}\" to \"{value}\". Changing the host configuration using WebApplicationBuilder.WebHost is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
        else if (string.Equals(key, WebHostDefaults.HostingStartupAssembliesKey, StringComparison.OrdinalIgnoreCase) &&
                !string.Equals(previousHostingStartupAssemblies, value, StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The hosting startup assemblies changed from \"{previousHostingStartupAssemblies}\" to \"{value}\". Changing the host configuration using WebApplicationBuilder.WebHost is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
        else if (string.Equals(key, WebHostDefaults.HostingStartupExcludeAssembliesKey, StringComparison.OrdinalIgnoreCase) &&
                !string.Equals(previousHostingStartupAssembliesExclude, value, StringComparison.OrdinalIgnoreCase))
        {
            throw new NotSupportedException($"The hosting startup assemblies exclude list changed from \"{previousHostingStartupAssembliesExclude}\" to \"{value}\". Changing the host configuration using WebApplicationBuilder.WebHost is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.");
        }
 
        _configuration[key] = value;
 
        return this;
    }
    
    // 已不再支持实现 ISupportsStartup 接口的所有方法
    IWebHostBuilder ISupportsStartup.Configure(Action<IApplicationBuilder> configure)
    {
        throw new NotSupportedException("Configure() is not supported by WebApplicationBuilder.WebHost. Use the WebApplication returned by WebApplicationBuilder.Build() instead.");
    }
 
    // 已不再支持实现 ISupportsStartup 接口的所有方法
    IWebHostBuilder ISupportsStartup.Configure(Action<WebHostBuilderContext, IApplicationBuilder> configure)
    {
        throw new NotSupportedException("Configure() is not supported by WebApplicationBuilder.WebHost. Use the WebApplication returned by WebApplicationBuilder.Build() instead.");
    }
 
    // 已不再支持实现 ISupportsStartup 接口的所有方法
    IWebHostBuilder ISupportsStartup.UseStartup(Type startupType)
    {
        throw new NotSupportedException("UseStartup() is not supported by WebApplicationBuilder.WebHost. Use the WebApplication returned by WebApplicationBuilder.Build() instead.");
    }
 
    // 已不再支持实现 ISupportsStartup 接口的所有方法
    IWebHostBuilder ISupportsStartup.UseStartup<TStartup>(Func<WebHostBuilderContext, TStartup> startupFactory)
    {
        throw new NotSupportedException("UseStartup() is not supported by WebApplicationBuilder.WebHost. Use the WebApplication returned by WebApplicationBuilder.Build() instead.");
    }
}
```
