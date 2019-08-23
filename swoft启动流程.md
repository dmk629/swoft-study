## 前言

本文以文件名为一级标题，方法名为二级标题，记录文件目录与代码行数，方便日后追代码。如有错误，欢迎指正。

> *本文基于* [Swoft 2.0.5](http://https://github.com/swoft-cloud/swoft "Swoft")


### 零、 swoft

目录：`/bin/swoft`

入口文件，启动应用 - `14行`

	（new \App\Application())->run();

### 一、 Application.php

目录：`/app/Application.php`

应用类继承SwoftApplication类 - `14行`

	class Application extends SwoftApplication

### 二、 SwoftApplication.php 

目录：`/vendor/swoft/framework/src/SwoftApplication.php`

##### 2.1 __construct()

- 检查环境（php版本,swoole版本）
- 初始化命令行下日志功能 - `462行`
- 应用初始化（检测根目录，设置别名，加载处理器） - `465行`

##### 2.2 init()

注入处理器 - `179行`

	$processors = $this->processors();

注入处理器容器 - `182行`

	$this->processor = new ApplicationProcessor($this);

##### 2.2 processors()

返回处理器群 - `282行`

	protected function processors(): array
    {
        return [
            new EnvProcessor($this),
            new ConfigProcessor($this),
            new AnnotationProcessor($this),
            new BeanProcessor($this),
            new EventProcessor($this),
            new ConsoleProcessor($this),
        ];
    }

##### 2.3 run()

处理 - `219行`

	$this->processor->handle();

### 三、 ApplicationProcessor.php 

目录：`/vendor/swoft/framework/src/Processor/ApplicationProcessor.php`

##### 3.1 handle()

循环调用处理器，进行处理 - `22行`
	
	// 获取禁用插件
	$disabled = $this->application->getDisabledProcessors();

    foreach ($this->processors as $processor) {
        $class = get_class($processor);

        // 跳过被禁用插件
        if (isset($disabled[$class])) {
            continue;
        }

        $processor->handle();
    }

----
>环境参数加载开始

### 四、 EnvProcessor.php 

目录：`vendor/swoft/framework/src/Processor/EnvProcessor.php`

使用`phpdotenv`组件对`.env`进行管理。

##### 4.1 handle()

调用适配器，读取环境配置 - `45行`
	
	//配置适配器
	$factory = new DotenvFactory([
         new EnvConstAdapter,
         new PutenvAdapter,
         new ServerConstAdapter
    ]);
    Dotenv::create($path, $name, $factory)->load();

### 五、 DotenvFactory.php 

目录：`/vendor/vlucas/phpdotenv/src/Environment/DotenvFactory.php`

##### 5.1 __construct()

配置支持的适配器。 - `32行`

    public function __construct(array $adapters = null)
    {
		//配置支持的适配器
        $this->adapters = array_filter($adapters === null ? [new ApacheAdapter(), new EnvConstAdapter(), new ServerConstAdapter(), new PutenvAdapter()] : $adapters, function (AdapterInterface $adapter) {
            return $adapter->isSupported();
        });
    }

##### 5.2 create()

创建一个新的环境变量存放对象（基于上述适配器） - `44行`

    public function create()
    {
        return new DotenvVariables($this->adapters, false);
    }

>创建过程涉及多个文件多个类方法来回调用，且行为简单不予展示。

>结论：经过`create()`方法完成了`Dotenv`类的**环境准备**与**容器开拓**，后续操作即可开始加载。

### 六、 Dotenv.php 

目录：`/vendor/vlucas/phpdotenv/src/Dotenv.php`

##### 6.1 load()

加载环境文件 - `78行`

    public function load()
    {
        return $this->loadData();
    }

##### 6.2 loadData()

加载环境文件真实方法 - `78行`

    protected function loadData($overload = false)
    {
        return $this->loader->setImmutable(!$overload)->load();
    }

### 七、 Loader.php 

目录：`/vendor/vlucas/phpdotenv/src/Loader.php`

##### 7.1 __construct()

获取文件地址与准备好的**容器** - `58行`

    public function __construct(array $filePaths, FactoryInterface $envFactory, $immutable = false)
    {
        $this->filePaths = $filePaths;
        $this->envFactory = $envFactory;
        $this->setImmutable($immutable);
    }

##### 7.2 setImmutable()

获取准备好的**容器** - `72行`

    public function __construct(array $filePaths, FactoryInterface $envFactory, $immutable = false)
    {
        $this->filePaths = $filePaths;
        $this->envFactory = $envFactory;
        $this->setImmutable($immutable);
    }

##### 7.3 load()

加载 - `88行`

    public function load()
    {
        return $this->loadDirect(
            self::findAndRead($this->filePaths)
        );
    }

##### 7.4 loadDirect()

处理加载文件地址 - `104行`

    public function loadDirect($content)
    {
        return $this->processEntries(
            Lines::process(preg_split("/(\r\n|\n|\r)/", $content))
        );
    }

##### 7.5 processEntries()

转存环境参数，方便用户获取 - `164行`

    private function processEntries(array $entries)
    {
        $vars = [];

		//循环转存至预设容器内，供用户读取
        foreach ($entries as $entry) {
            list($name, $value) = Parser::parse($entry);
            $vars[$name] = $this->resolveNestedVariables($value);
            $this->setEnvironmentVariable($name, $vars[$name]);
        }

        return $vars;
    }

##### 7.5 resolveNestedVariables()

解析参数 - `187行`

	private function resolveNestedVariables($value = null)
    {
        return Option::fromValue($value)
            ->filter(function ($str) {
                return strpos($str, '$') !== false;
            })
            ->flatMap(function ($str) {
                return Regex::replaceCallback(
                    '/\${([a-zA-Z0-9_.]+)}/',
                    function (array $matches) {
                        return Option::fromValue($this->getEnvironmentVariable($matches[1]))
                            ->getOrElse($matches[0]);
                    },
                    $str
                )->success();
            })
            ->getOrElse($value);
    }

>环境参数加载完成。

-----

### 八、 ConfigProcessor.php 

目录：`/vendor/swoft/framework/src/Processor/ConfigProcessor.php`

##### 8.1 handle()

处理 - `18行`

    public function handle(): bool
    {
        $this->defineConstant();
        if (!$this->application->beforeConfig()) {
            return false;
        }
        return $this->application->afterConfig();
    }

##### 8.2 defineConstant()

获取环境变量DEBUG等级 - `33行`

    protected function defineConstant(): void
    {
        define('APP_DEBUG', (int)env('APP_DEBUG', 0));
        define('SWOFT_DEBUG', (int)env('SWOFT_DEBUG', 0));
    }

### 九、 AnnotationProcessor.php 

目录：`/vendor/swoft/framework/src/Processor/AnnotationProcessor.php`

##### 9.1 handle()

注册注解 - `31行`

	AnnotationRegister::load([
        'inPhar'               => \IN_PHAR,
        'basePath'             => $app->getBasePath(),
        'notifyHandler'        => [$this, 'notifyHandler'],
        'disabledAutoLoaders'  => $app->getDisabledAutoLoaders(),
        'disabledPsr4Prefixes' => $app->getDisabledPsr4Prefixes(),
    ]);

    $stats = AnnotationRegister::getClassStats();

### 十、 AnnotationRegister.php 

目录：`/vendor/swoft/annotation/src/AnnotationRegister.php`

##### 10.1 load()

注册加载器 - `130行`

	public static function load(array $config = []): void
    {
        $resource = new AnnotationResource($config);
        $resource->load();
    }

### 十一、 AnnotationResource.php 

目录：`/vendor/swoft/annotation/src/Resource/AnnotationResource.php`

##### 11.1 __construct()

注册加载器 - `127行`

	$this->registerLoader();
    $this->classLoader = ComposerHelper::getClassLoader();//自动加载
    $this->includedFiles = get_included_files();

##### 11.2 registerLoader()

注册自动加载 - `420行`

	private function registerLoader(): void
    {
        AnnotationRegistry::registerLoader(function (string $class) {
            if (class_exists($class)) {
                return true;
            }

            return false;
        });
    }

>AnnotationRegistry类请自行阅读。

##### 11.2 registerLoader()

加载 - `139行`

	public function load(): void
    {
		//psr4加载类
        $prefixDirsPsr4 = $this->classLoader->getPrefixesPsr4();

        foreach ($prefixDirsPsr4 as $ns => $paths) {
            // Only scan namespaces
            if ($this->onlyNamespaces && !in_array($ns, $this->onlyNamespaces, true)) {
                $this->notify('excludeNs', $ns);
                continue;
            }

            // It is excluded psr4 prefix
            if ($this->isExcludedPsr4Prefix($ns)) {
                AnnotationRegister::registerExcludeNs($ns);
                $this->notify('excludeNs', $ns);
                continue;
            }

            // Find package/component loader class
            foreach ($paths as $path) {
                $loaderFile = $this->getAnnotationClassLoaderFile($path);
                if (!file_exists($loaderFile)) {
                    $this->notify('noLoaderFile', $this->clearBasePath($path), $loaderFile);
                    continue;
                }

                $loaderClass = $this->getAnnotationLoaderClassName($ns);
                if (!class_exists($loaderClass)) {
                    $this->notify('noLoaderClass', $loaderClass);
                    continue;
                }

                $loaderObject = new $loaderClass();
                if (!$loaderObject instanceof LoaderInterface) {
                    $this->notify('invalidLoader', $loaderFile);
                    continue;
                }

                $this->notify('findLoaderClass', $this->clearBasePath($loaderFile));

                // If is disable, will skip scan annotation classes
                if (!isset($this->disabledAutoLoaders[$loaderClass])) {
                    AnnotationRegister::registerAutoLoaderFile($loaderFile);
                    $this->notify('addLoaderClass', $loaderClass);
                    $this->loadAnnotation($loaderObject);
                }

                // Storage auto loader to register
                AnnotationRegister::addAutoLoader($ns, $loaderObject);
            }
        }
    }
