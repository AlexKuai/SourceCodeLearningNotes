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
// 实现 IHost 并对实际的 IHost 封装，实现的所有方法最终都会转移到实际的 IHost 上
// 实现 IApplicationBuilder 并对实际的 IApplicationBuilder 封装，实现的所有方法最终都会转移到实际的 IApplicationBuilder 上
// 实现 IEndpointRouteBuilder 用来注册 EndpointDataSource
public sealed class WebApplication : IHost, IApplicationBuilder, IEndpointRouteBuilder, IAsyncDisposable
{
    internal const string GlobalEndpointRouteBuilderKey = "__GlobalEndpointRouteBuilder";

    private readonly IHost _host;
    private readonly List<EndpointDataSource> _dataSources = new();

    internal WebApplication(IHost host)
    {
        _host = host;
        // 创建实际的 IApplicationBuilder
        ApplicationBuilder = new ApplicationBuilder(host.Services, ServerFeatures);
        // 创建名称为 {IWebHostEnvironment.ApplicationName} 或 "WebApplication" 日志类别的 ILogger
        Logger = host.Services.GetRequiredService<ILoggerFactory>().CreateLogger(Environment.ApplicationName ?? nameof(WebApplication));
 
        // 由实际的 IApplicationBuilder 提供的共享字典
        // 将自身以 "__GlobalEndpointRouteBuilder" 为 Key 添加到共享字典中
        Properties[GlobalEndpointRouteBuilderKey] = this;
    }

    // 从实际的 IHost 中返回表示根容器的 IServiceProvider
    public IServiceProvider Services => _host.Services;

    // 返回应用配置
    // 本质是 HostApplicationBuilder.Configuration 保存的 ConfigurationManager
    public IConfiguration Configuration => _host.Services.GetRequiredService<IConfiguration>();

    // 返回 IWebHostEnvironment
    public IWebHostEnvironment Environment => _host.Services.GetRequiredService<IWebHostEnvironment>();

    // 返回 IHostApplicationLifetime
    public IHostApplicationLifetime Lifetime => _host.Services.GetRequiredService<IHostApplicationLifetime>();

    // 返回日志类别为 {IWebHostEnvironment.ApplicationName} 或 "WebApplication" 的 ILogger
    public ILogger Logger { get; }

    // 返回服务器提供的特性集合中 IServerAddressesFeature.Addresses 表示的地址集合
    // 可以利用这个集合直接添加监听地址
    public ICollection<string> Urls => ServerFeatures.GetRequiredFeature<IServerAddressesFeature>().Addresses;

    // 返回 ApplicationBuilder
    internal ApplicationBuilder ApplicationBuilder { get; }

    // 实现 IHost
    // 启动宿主
    public Task StartAsync(CancellationToken cancellationToken = default) =>
        _host.StartAsync(cancellationToken);

    // 实现 IHost
    // 停止宿主
    public Task StopAsync(CancellationToken cancellationToken = default) =>
        _host.StopAsync(cancellationToken);

    // 启动宿主（异步方式）
    // 支持长时间运行的服务
    // 可以直接设置监听地址
    public Task RunAsync(string? url = null)
    {
        Listen(url);
        return HostingAbstractionsHostExtensions.RunAsync(this);
    }
    
    // 启动宿主（同步方式）
    // 支持长时间运行的服务
    // 可以直接设置监听地址
    public void Run(string? url = null)
    {
        Listen(url);
        HostingAbstractionsHostExtensions.Run(this);
    }

    // 设置监听地址
    // 直接针对服务器提供的特性集合中 IServerAddressesFeature.Addresses 进行设置
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

        // 只支持设置一个地址，并且不支持 ";" 分割的多个地址
        addresses.Clear();
        addresses.Add(url);
    }

    // 返回 ApplicationBuilder.Properties 表示的共享字典
    internal IDictionary<string, object?> Properties => ApplicationBuilder.Properties;
    // 显式实现 IApplicationBuilder
    IDictionary<string, object?> IApplicationBuilder.Properties => Properties;

    // 返回由服务器提供的特性集合
    internal IFeatureCollection ServerFeatures => _host.Services.GetRequiredService<IServer>().Features;
    // 显式实现 IApplicationBuilder
    IFeatureCollection IApplicationBuilder.ServerFeatures => ServerFeatures;

    // 显式实现 IApplicationBuilder
    // 返回 ApplicationBuilder.ApplicationServices 表示的根容器
    IServiceProvider IApplicationBuilder.ApplicationServices
    {
        get => ApplicationBuilder.ApplicationServices;
        set => ApplicationBuilder.ApplicationServices = value;
    }

    // 调用 ApplicationBuilder.Build 方法构建请求处理委托链，并返回位于头部的请求处理器
    internal RequestDelegate BuildRequestDelegate() => ApplicationBuilder.Build();
    // 显式实现 IApplicationBuilder
    RequestDelegate IApplicationBuilder.Build() => BuildRequestDelegate();
 
    // 显式实现 IApplicationBuilder
    // 调用 ApplicationBuilder.New 方法创建新的 IApplicationBuilder
    IApplicationBuilder IApplicationBuilder.New()
    {
        var newBuilder = ApplicationBuilder.New();
        // 利用 ApplicationBuilder.New 创建的 IApplicationBuilder 会复制共享字典
        // 而新建的 IApplicationBuilder 还是 ApplicationBuilder，不能代表全局 IEndpointRouteBuilder
        // 所以需要从新建的 IApplicationBuilder 的共享字典中删除 Key 为 "__GlobalEndpointRouteBuilder" 的项
        newBuilder.Properties.Remove(GlobalEndpointRouteBuilderKey);
        return newBuilder;
    }
 
    // 实现 IApplicationBuilder
    // 本质是由 ApplicationBuilder 完成中间件注册
    public IApplicationBuilder Use(Func<RequestDelegate, RequestDelegate> middleware)
    {
        ApplicationBuilder.Use(middleware);
        return this;
    }
    
    // 显式实现 IEndpointRouteBuilder
    IApplicationBuilder IEndpointRouteBuilder.CreateApplicationBuilder() => ((IApplicationBuilder)this).New();

    internal ICollection<EndpointDataSource> DataSources => _dataSources;
    // 显式实现 IEndpointRouteBuilder
    ICollection<EndpointDataSource> IEndpointRouteBuilder.DataSources => DataSources;

    // 显式实现 IEndpointRouteBuilder
    IServiceProvider IEndpointRouteBuilder.ServiceProvider => Services;

    // 静态方法
    // 创建 WebApplicationBuilder 并调用 WebApplicationBuilder.Build 方法构建 WebApplication
    public static WebApplication Create(string[]? args = null) =>
        new WebApplicationBuilder(new() { Args = args }).Build();
    
    // 静态方法
    // 创建 WebApplicationBuilder
    public static WebApplicationBuilder CreateBuilder(string[] args) =>
        new(new() { Args = args });

    // 静态方法
    // 创建 WebApplicationBuilder
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
// WebApplication 选项
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
// HostApplicationBuilder 选项
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
// HostApplication 建造者接口
public interface IHostApplicationBuilder
{
    // 构建过程中用于共享的数据字典
    IDictionary<object, object> Properties { get; }
    
    // 返回 IConfigurationManager
    IConfigurationManager Configuration { get; }
    
    // 返回 IHostEnvironment
    IHostEnvironment Environment { get; }
 
    // 返回 ILoggingBuilder
    ILoggingBuilder Logging { get; }
 
    IMetricsBuilder Metrics { get; }
 
    // 返回 IServiceCollection
    IServiceCollection Services { get; }
    
    // 使用第三方 IServiceProvider 工厂和对应的容器建造者配置
    void ConfigureContainer<TContainerBuilder>(IServiceProviderFactory<TContainerBuilder> factory, Action<TContainerBuilder>? configure = null) where TContainerBuilder : notnull;
}
```

- HostApplicationBuilder  

```C#
// IHostApplicationBuilder 的默认实现
// 主要作用：
// 1. 收集并构建默认的宿主和应用配置
// 2. 收集并构建默认的服务注册
// 3. 使用第三方 IServiceProvider 工厂和对应的容器建造者配置创建 IServiceProvider
// 4. 构建实际的 IHost
// 5. 对外提供核心属性用来收集各种配置
public sealed class HostApplicationBuilder : IHostApplicationBuilder
{
    private readonly HostBuilderContext _hostBuilderContext;
    // 创建 ServiceCollection 用于服务注册
    private readonly ServiceCollection _serviceCollection = new();
    private readonly IHostEnvironment _environment;
    private readonly LoggingBuilder _logging;
    private readonly MetricsBuilder _metrics;
 
    // 创建 IServiceProvider 的工厂方法
    private Func<IServiceProvider> _createServiceProvider;
    // 配置容器建造者的工厂方法
    private Action<object> _configureContainer = _ => { };
    private HostBuilderAdapter? _hostBuilderAdapter;
 
    // 表示根容器的 IServiceProvider
    private IServiceProvider? _appServices;
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
        Configuration = settings.Configuration ?? new ConfigurationManager();
 
        if (!settings.DisableDefaults)
        {
            if (settings.ContentRootPath is null && Configuration[HostDefaults.ContentRootKey] is null)
            {
                // 添加内存配置源
                // 添加 "contentRoot" 配置节表示的工作目录路径
                // 默认使用当前目录（一般为应用程序根目录）
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
            // 应用默认配置源
            HostingHostBuilderExtensions.ApplyDefaultAppConfiguration(_hostBuilderContext, Configuration, settings.Args);
            // 注册默认服务
            HostingHostBuilderExtensions.AddDefaultServices(_hostBuilderContext, Services);
            // 根据环境创建 ServiceProviderOptions 选项
            serviceProviderOptions = HostingHostBuilderExtensions.CreateDefaultServiceProviderOptions(_hostBuilderContext);
        }
 
        // 用来创建默认的 IServiceProvider
        _createServiceProvider = () =>
        {
            // 默认情况下 _configureContainer 保存的 Action<object> 委托是一个空方法
            _configureContainer(Services);
            return serviceProviderOptions is null ? Services.BuildServiceProvider() : Services.BuildServiceProvide(serviceProviderOptions);
        };
    }
    
    // 初始化
    private void Initialize(HostApplicationBuilderSettings settings, out HostBuilderContext hostBuilderContext, out IHostEnvironment environment, out LoggingBuilder logging, out MetricsBuilder metrics)
    {
        // 添加命令行配置源
        HostingHostBuilderExtensions.AddCommandLineConfig(Configuration, settings.Args);
 
        // 添加内存配置源
        // 利用 HostApplicationBuilderSettings 中的属性添加配置
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

        // 为后续使用 ConfigurationManager 添加基于文件的配置源时提供默认的 IFileProvider
        Configuration.SetFileProvider(physicalFileProvider);

        // 创建 HostBuilderContext
        hostBuilderContext = new HostBuilderContext(new Dictionary<object, object>())
        {
            HostingEnvironment = hostingEnvironment,
            Configuration = Configuration,
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
            // 得到 IServiceProvider 并返回
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

        // 应用 IServiceProvider 工厂和配置创建表示根容器的 IServiceProvider
        _appServices = _createServiceProvider();
 
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
// WebApplication 建造者
// 实现 IHostApplicationBuilder 并对实际的 IHostApplicationBuilder 封装
public sealed class WebApplicationBuilder : IHostApplicationBuilder
{
    private const string EndpointRouteBuilderKey = "__EndpointRouteBuilder";
    private const string AuthenticationMiddlewareSetKey = "__AuthenticationMiddlewareSet";
    private const string AuthorizationMiddlewareSetKey = "__AuthorizationMiddlewareSet";
    private const string UseRoutingKey = "__UseRouting";

    // 实际的 IHostApplicationBuilder
    private readonly HostApplicationBuilder _hostApplicationBuilder;
    // 表示 GenericWebHostService 服务注册的 ServiceDescriptor
    private readonly ServiceDescriptor _genericWebHostServiceDescriptor;

    // 构建得到的 WebApplication
    private WebApplication? _builtApplication;

    internal WebApplicationBuilder(WebApplicationOptions options, Action<IHostBuilder>? configureDefaults = null)
    {
        // ConfigurationManager 同时实现了 IConfigurationBuilder 和 IConfiguration
        // 所以既可以用来收集配置源，也可以用来提供配置
        // 注意：
        // 使用 ConfigurationManager 收集配置源时会立即构建对应的 IConfigurationProvider 用于提供配置
        // 而不需要像传统的配置模型那样先调用 IConfigurationBuilder.Build 方法构建 IConfigurationRoot 后才能使用配置
        var configuration = new ConfigurationManager();
        
        // 添加 "ASPNETCORE_" 前缀的环境变量配置源
        configuration.AddEnvironmentVariables(prefix: "ASPNETCORE_");
 
        // 创建 HostApplicationBuilder
        _hostApplicationBuilder = new HostApplicationBuilder(new HostApplicationBuilderSettings
        {
            Args = options.Args,
            ApplicationName = options.ApplicationName,
            EnvironmentName = options.EnvironmentName,
            ContentRootPath = options.ContentRootPath,
            Configuration = configuration,
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

        // 应用委托
        // 使用 BootstrapHostBuilder 收集配置
        // 虽然 BootstrapHostBuilder 实现了 IHostBuilder
        // 但内部很多功能已经不支持，所以此处只作为测试目的
        configureDefaults?.Invoke(bootstrapHostBuilder);
 
        // 调用 IWebHostBuilder.ConfigureWebHostDefaults 扩展方法
        // 最终会创建 GenericWebHostBuilder，并将收集到的配置转移到 BootstrapHostBuilder 上
        // 并注册 IHostedService，使用 GenericWebHostService
        bootstrapHostBuilder.ConfigureWebHostDefaults(
            webHostBuilder =>
            {
                // 添加针对中间件建造者的配置
                webHostBuilder.Configure(ConfigureApplication);
    
                InitializeWebHostSettings(webHostBuilder);
            },
            options =>
            {
                // 禁用环境变量配置源
                // 因为 "ASPNETCORE_" 前缀的环境变量配置已经添加到配置中，无需重复添加
                options.SuppressEnvironmentConfiguration = true;
            }
        );

        // 得到表示 GenericWebHostService 服务注册的 ServiceDescriptor
        // 此时 IServiceCollection 中表示 GenericWebHostService 服务注册的 ServiceDescriptor 已被移除
        _genericWebHostServiceDescriptor = InitializeHosting(bootstrapHostBuilder);
    }

    // 由于在构建 GenericWebHostBuilder 时阻止添加 "ASPNETCORE_" 前缀的环境变量配置
    // 所以通过 UseSetting 方法为备用配置添加强类型 Startup 所需的配置
    private void InitializeWebHostSettings(IWebHostBuilder webHostBuilder)
    {
        webHostBuilder.UseSetting(WebHostDefaults.ApplicationKey, _hostApplicationBuilder.Environment.ApplicationName ?? "");
        webHostBuilder.UseSetting(WebHostDefaults.PreventHostingStartupKey, Configuration[WebHostDefaults.PreventHostingStartupKey]);
        webHostBuilder.UseSetting(WebHostDefaults.HostingStartupAssembliesKey, Configuration[WebHostDefaults.HostingStartupAssembliesKey]);
        webHostBuilder.UseSetting(WebHostDefaults.HostingStartupExcludeAssembliesKey, Configuration[WebHostDefaults.HostingStartupExcludeAssembliesKey]);
    }
    
    // 初始化
    private ServiceDescriptor InitializeHosting(BootstrapHostBuilder bootstrapHostBuilder)
    {
        // 应用 BootstrapHostBuilder 收集的各种配置
        // 并返回表示 GenericWebHostService 服务注册的 ServiceDescriptor
        var genericWebHostServiceDescriptor = bootstrapHostBuilder.RunDefaultCallbacks();
 
        var webHostContext = (WebHostBuilderContext)bootstrapHostBuilder.Properties[typeof(WebHostBuilderContext)];
        Environment = webHostContext.HostingEnvironment;
 
        // 创建 ConfigureHostBuilder
        // 用来提供 IHostBuilder 编程模型
        Host = new ConfigureHostBuilder(bootstrapHostBuilder.Context, Configuration, Services);
        // 创建 ConfigureWebHostBuilder
        // 用来提供 IWebHostBuilder 编程模型
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
        // 重新添加表示 GenericWebHostService 服务注册的 ServiceDescriptor
        _hostApplicationBuilder.Services.Add(_genericWebHostServiceDescriptor);
        // 可以通过 IHostBuilder 编程模型提供第三方的 IServiceProvider 工厂和对应的容器建造者配置
        Host.ApplyServiceProviderFactory(_hostApplicationBuilder);
        // 构建实际的 IHost 并用来创建 WebApplication
        _builtApplication = new WebApplication(_hostApplicationBuilder.Build());
        return _builtApplication;
    }

    // 配置中间件建造者
    // 参数 app 表示的 IApplicationBuilder 是通过 GenericWebHostService 构造函数注入的 IApplicationBuilderFactory 创建的
    private void ConfigureApplication(WebHostBuilderContext context, IApplicationBuilder app) =>
        ConfigureApplication(context, app, allowDeveloperExceptionPage: true);
    
    // 配置中间件建造者
    // 参数 app 表示的 IApplicationBuilder 是通过 GenericWebHostService 构造函数注入的 IApplicationBuilderFactory 创建的
    private void ConfigureApplication(WebHostBuilderContext context, IApplicationBuilder app, bool allowDeveloperExceptionPage)
    {
        Debug.Assert(_builtApplication is not null);
 
        // 在 Minimal API 编程模型中，WebApplication 扮演着全局 IEndpointRouteBuilder 的角色
        // 并且 WebApplication 还扮演着 IApplicationBuilder 的角色，提供 IApplicationBuilder.Properties 表示的共享字典
        // 创建 WebApplication 时会将自身以 "__GlobalEndpointRouteBuilder" 为 Key 添加到共享字典中作为全局 IEndpointRouteBuilder
        // 所以需要从当前 IApplicationBuilder 的共享字典中删除 Key 为 "__EndpointRouteBuilder" 的项
        if (app.Properties.TryGetValue(EndpointRouteBuilderKey, out var priorRouteBuilder))
        {
            app.Properties.Remove(EndpointRouteBuilderKey);
        }
 
        // 开发环境添加开发者异常页中间件
        if (allowDeveloperExceptionPage && context.HostingEnvironment.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }
 
        // Wrap the entire destination pipeline in UseRouting() and UseEndpoints(), essentially:
        // destination.UseRouting()
        // destination.Run(source)
        // destination.UseEndpoints()

        // 将 WebApplication 以 "__GlobalEndpointRouteBuilder" 为 Key 添加到当前 IApplicationBuilder 的共享字典中作为全局 IEndpointRouteBuilder
        app.Properties.Add(WebApplication.GlobalEndpointRouteBuilderKey, _builtApplication);

        // 如果利用 WebApplication 作为 IEndpointRouteBuilder 注册了 EndpointDataSource
        if (_builtApplication.DataSources.Count > 0)
        {
            // 在 Minimal API 编程模型中，由于 WebApplication 扮演着全局 IEndpointRouteBuilder 的角色
            // 所以可以直接利用 WebApplication 注册 EndpointDataSource
            // 在 IHost/IHostBuilder 编程模型中，需要先调用 IApplicationBuilder.UseRouting 扩展方法
            // 创建 DefaultEndpointRouteBuilder 作为 IEndpointRouteBuilder
            // 再调用 IApplicationBuilder.UseEndpoints 扩展方法
            // 添加针对 IEndpointRouteBuilder 的配置
            // 最后利用 DefaultEndpointRouteBuilder 注册 EndpointDataSource

            // 如果利用 WebApplication 作为 IApplicationBuilder 没有调用 IApplicationBuilder.UseRouting 扩展方法
            // 则需要利用当前 IApplicationBuilder 调用 IApplicationBuilder.UseRouting 扩展方法
            // 主要完成两个功能：
            // 1. 确保注册 EndpointRoutingMiddleware 中间件
            // 2. 利用 WebApplication 作为 IEndpointRouteBuilder 传递给 EndpointRoutingMiddleware 的构造函数
            if (!_builtApplication.Properties.TryGetValue(EndpointRouteBuilderKey, out var localRouteBuilder))
            {
                app.UseRouting();
                _builtApplication.Properties[UseRoutingKey] = app.Properties[UseRoutingKey];
            }
            else
            {
                app.Properties[EndpointRouteBuilderKey] = localRouteBuilder;
            }
        }
 
        var serviceProviderIsService = _builtApplication.Services.GetService<IServiceProviderIsService>();
        if (serviceProviderIsService?.IsService(typeof(IAuthenticationSchemeProvider)) is true)
        {
            if (!_builtApplication.Properties.ContainsKey(AuthenticationMiddlewareSetKey))
            {
                _builtApplication.Properties[AuthenticationMiddlewareSetKey] = true;
                app.UseAuthentication();
            }
        }
 
        if (serviceProviderIsService?.IsService(typeof(IAuthorizationHandlerProvider)) is true)
        {
            if (!_builtApplication.Properties.ContainsKey(AuthorizationMiddlewareSetKey))
            {
                _builtApplication.Properties[AuthorizationMiddlewareSetKey] = true;
                app.UseAuthorization();
            }
        }
 
        // 注册导线中间件
        var wireSourcePipeline = new WireSourcePipeline(_builtApplication);
        app.Use(wireSourcePipeline.CreateMiddleware);
 
        if (_builtApplication.DataSources.Count > 0)
        {
            app.UseEndpoints(_ => { });
        }
 
        MergeMiddlewareDescriptions(app);

        // 将 WebApplication 在扮演 IApplicationBuilder 时利用共享字典收集的数据转移到当前 IApplicationBuilder 的共享字典上
        foreach (var item in _builtApplication.Properties)
        {
            app.Properties[item.Key] = item.Value;
        }
 
        app.Properties.Remove(WebApplication.GlobalEndpointRouteBuilderKey);
 
        if (priorRouteBuilder is not null)
        {
            app.Properties[EndpointRouteBuilderKey] = priorRouteBuilder;
        }
    }

    // 显式实现 IHostApplicationBuilder
    // 转移到 HostApplicationBuilder 上
    void IHostApplicationBuilder.ConfigureContainer<TContainerBuilder>(IServiceProviderFactory<TContainerBuilder> factory, Action<TContainerBuilder>? configure) =>
        _hostApplicationBuilder.ConfigureContainer(factory, configure);

    // 源管道导线
    // 作用是将源管道连接到目标管道上
    private sealed class WireSourcePipeline(IApplicationBuilder builtApplication)
    {
        private readonly IApplicationBuilder _builtApplication = builtApplication;
 
        // 这个 Func<RequestDelegate, RequestDelegate> 委托原型方法可以理解为导线中间件
        // 通过注册这个中间件可以实现将 WebApplication 模拟 IApplicationBuilder 注册的中间件连接到目标管道上
        public RequestDelegate CreateMiddleware(RequestDelegate next)
        {
            _builtApplication.Run(next);
            // 由于 ApplicationBuilder.Builder 方法实现中会将一个返回 404 状态码的请求处理器作为末端处理器
            // 所以通过注册一个短路中间件的方式，即可以连接后续请求处理器，又可以丢弃这个末端请求处理器
            return _builtApplication.Build();
        }
    }
}
```

- BootstrapHostBuilder

```C#
// 实现 IHostBuilder，作用类似于 HostBuilder
// 可以用来配置 IWebHostBuilder
// 在 Minimal API 编程模型中 BootstrapHostBuilder 也是 GenericWebHostBuilder 持有的 IHostBuilder
// 主要作用：
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

        // 确保 HostBuilderContext 已经创建
        // 否则抛出 InvalidOperationException
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
    
    // 应用配置并返回表示 GenericWebHostService 服务注册的 ServiceDescriptor
    // 本质是通过 HostApplicationBuilder 的两个核心属性来完成工作
    // 利用 HostApplicationBuilder.Configuration 构建宿主、应用配置
    // 利用 HostApplicationBuilder.Services 构建服务注册
    // 本质上所有配置都是通过 IWebHostBuilder 收集的
    public ServiceDescriptor RunDefaultCallbacks()
    {
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
 
        // 确保 IHostedService 服务注册存在，并且实现类型是 GenericWebHostService
        // 否则抛出 InvalidOperationException
        for (int i = _builder.Services.Count - 1; i >= 0; i--)
        {
            var descriptor = _builder.Services[i];
            if (descriptor.ServiceType == typeof(IHostedService))
            {
                Debug.Assert(descriptor.ImplementationType?.Name == "GenericWebHostService");
 
                genericWebHostServiceDescriptor = descriptor;
                // 注意：
                // 此处只是为了保证 GenericWebHostService 服务注册存在
                // 删除的目的是最终对应的 ServiceDescriptor 会在 WebApplicationBuild.Build 方法中重新添加
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
 
    // 添加针对应用的配置
    // 本质是利用 HostApplicationBuilder.Configuration 添加配置并立即构建应用配置
    public IHostBuilder ConfigureAppConfiguration(Action<HostBuilderContext, IConfigurationBuilder> configureDelegate)
    {
        configureDelegate(_context, _configuration);
        return this;
    }
 
    // 添加针对容器建造者的配置
    // 使用适配器模式
    public IHostBuilder ConfigureContainer<TContainerBuilder>(Action<HostBuilderContext, TContainerBuilder> configureDelegate)
    {
        ArgumentNullException.ThrowIfNull(configureDelegate);
 
        _configureContainerActions.Add((context, containerBuilder) => configureDelegate(context, (TContainerBuilder)containerBuilder));
 
        return this;
    }
 
    // 添加针对宿主的配置
    // 本质是利用 HostApplicationBuilder.Configuration 添加配置并立即构建宿主配置
    public IHostBuilder ConfigureHostConfiguration(Action<IConfigurationBuilder> configureDelegate)
    {
        var previousApplicationName = _configuration[HostDefaults.ApplicationKey];
        var previousContentRoot = HostingPathResolver.ResolvePath(_context.HostingEnvironment.ContentRootPath);
        var previousContentRootConfig = _configuration[HostDefaults.ContentRootKey];
        var previousEnvironment = _configuration[HostDefaults.EnvironmentKey];
 
        configureDelegate(_configuration);
 
        // 通过 IHostBuilder 编程模型添加的宿主配置不能覆盖以上已经存在的配置节
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
 
    // 添加针对服务注册的配置
    // 本质是利用 HostApplicationBuilder.Services 添加配置并立即构建服务注册
    public IHostBuilder ConfigureServices(Action<HostBuilderContext, IServiceCollection> configureDelegate)
    {
        configureDelegate(_context, _services);
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
        // 利用 HostBuilderContext 通过工厂方法创建 IServiceProvider 工厂
        return UseServiceProviderFactory(factory(_context));
    }
    
    // IHostBuilder 编程模型已不再支持通过扩展方法配置 IWebHostBuilder
    // 防止开发人员以为可以通过调用 IHostBuilder.ConfigureWebHost 扩展方法配置 IWebHostBuilder
    IHostBuilder ISupportsConfigureWebHost.ConfigureWebHost(Action<IWebHostBuilder> configure, Action<WebHostBuilderOptions> configureOptions)
    {
        throw new NotSupportedException("ConfigureWebHost() is not supported by WebApplicationBuilder.Host. Use the WebApplication returned by WebApplicationBuilder.Build() instead.");
    }
 
    // 本质是利用 HostApplicationBuilder.ConfigureContainer 添加第三方 IServiceProvider 工厂和配置
    // 并在后续的 HostApplicationBuilder.Build 方法中创建 IServiceProvider
    internal void ApplyServiceProviderFactory(HostApplicationBuilder hostApplicationBuilder)
    {
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
    
    // IServiceProviderFactory 的适配器
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
    // 本质是利用 HostApplicationBuilder.Configuration 添加并立即构建应用配置
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
 
        configureDelegate(_context, _configuration);
 
        // 通过 IWebHostBuilder 编程模型添加的应用配置不能覆盖以上已经存在的配置节
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
 
    // 添加针对服务注册的配置
    // 本质是利用 HostApplicationBuilder.Services 添加并立即构建服务注册
    public IWebHostBuilder ConfigureServices(Action<WebHostBuilderContext, IServiceCollection> configureServices)
    {
        configureServices(_context, _services);
        return this;
    }
 
    // 添加针对服务注册的配置
    // 本质是利用 HostApplicationBuilder.Services 添加并立即构建服务注册
    public IWebHostBuilder ConfigureServices(Action<IServiceCollection> configureServices)
    {
        return ConfigureServices((WebHostBuilderContext context, IServiceCollection services) => configureServices(services));
    }
 
    // 本质是利用 HostApplicationBuilder.Configuration 得到对应配置节的配置 
    public string? GetSetting(string key)
    {
        return _configuration[key];
    }
 
    // 本质是利用 HostApplicationBuilder.Configuration 设置对应配置节的配置
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

        // 通过 IWebHostBuilder 编程模型添加的配置不能覆盖以上已经存在的配置节
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
    
    // 通过 IWebHostBuilder 编程模型已不再支持实现 ISupportsStartup 接口的所有方法
    IWebHostBuilder ISupportsStartup.Configure(Action<IApplicationBuilder> configure)
    {
        throw new NotSupportedException("Configure() is not supported by WebApplicationBuilder.WebHost. Use the WebApplication returned by WebApplicationBuilder.Build() instead.");
    }
 
    // 通过 IWebHostBuilder 编程模型已不再支持实现 ISupportsStartup 接口的所有方法
    IWebHostBuilder ISupportsStartup.Configure(Action<WebHostBuilderContext, IApplicationBuilder> configure)
    {
        throw new NotSupportedException("Configure() is not supported by WebApplicationBuilder.WebHost. Use the WebApplication returned by WebApplicationBuilder.Build() instead.");
    }
 
    // 通过 IWebHostBuilder 编程模型已不再支持实现 ISupportsStartup 接口的所有方法
    IWebHostBuilder ISupportsStartup.UseStartup(Type startupType)
    {
        throw new NotSupportedException("UseStartup() is not supported by WebApplicationBuilder.WebHost. Use the WebApplication returned by WebApplicationBuilder.Build() instead.");
    }
 
    // 通过 IWebHostBuilder 编程模型已不再支持实现 ISupportsStartup 接口的所有方法
    IWebHostBuilder ISupportsStartup.UseStartup<TStartup>(Func<WebHostBuilderContext, TStartup> startupFactory)
    {
        throw new NotSupportedException("UseStartup() is not supported by WebApplicationBuilder.WebHost. Use the WebApplication returned by WebApplicationBuilder.Build() instead.");
    }
}
```
