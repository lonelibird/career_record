**Apache Doris的Fe是如何启动的，它启动的过程中做了些什么**<br>
`FE是什么`<br>
FE是FrontEnd的缩写，是Apache Doris面向应用的服务端。<br>
`FE的语言`<br>
JAVA<br>
`启动类`<br>
PaloFe<br>
<br>
    //读取环境变量DORIS_HOME
    public static final String DORIS_HOME_DIR = System.getenv("DORIS_HOME");
    //读取进程目录环境变量，用于防止同一台服务器同时启动两个实例
    public static final String PID_DIR = System.getenv("PID_DIR");

    public static void main(String[] args) {
        start(DORIS_HOME_DIR, PID_DIR, args);
    }

    // entrance for doris frontend
    public static void start(String dorisHomeDir, String pidDir, String[] args) {
        if (Strings.isNullOrEmpty(dorisHomeDir)) {
            System.err.println("env DORIS_HOME is not set.");
            return;
        }

        if (Strings.isNullOrEmpty(pidDir)) {
            System.err.println("env PID_DIR is not set.");
            return;
        }

        //解析命令行的中参数
        支持的参数有 -v(-version):打印版本 -h(-helper)：打印帮助 -b(-bdb):打开debug -l(-listdb):列出所有的db 
         -d(-db):指定数据库 -s(-stat):打印数据库的统计信息，-f(-from):指定扫描的键 -t(-to):与from配对使用,-m(-metaversion):元数据
        CommandLineOptions cmdLineOpts = parseArgs(args);

        try {
            // pid file 通过获取文件锁防止启动多个实例
            if (!createAndLockPidFile(pidDir + "/fe.pid")) {
                throw new IOException("pid file is already locked.");
            }
            //读取配置文件
            // init config
            Config config = new Config();
            config.init(dorisHomeDir + "/conf/fe.conf");
            // Must init custom config after init config, separately.
            // Because the path of custom config file is defined in fe.conf
            config.initCustom(Config.custom_config_dir + "/fe_custom.conf");
            //读取LDAP配置文件
            LdapConfig ldapConfig = new LdapConfig();
            if (new File(dorisHomeDir + "/conf/ldap.conf").exists()) {
                ldapConfig.init(dorisHomeDir + "/conf/ldap.conf");
            }

            // check it after Config is initialized, otherwise the config 'check_java_version' won't work.
            if (!JdkUtils.checkJavaVersion()) {
                throw new IllegalArgumentException("Java version doesn't match");
            }
            //读取日志配置
            Log4jConfig.initLogging(dorisHomeDir + "/conf/");

            // set dns cache ttl
            java.security.Security.setProperty("networkaddress.cache.ttl", "60");

            // check command line options
            checkCommandLineOptions(cmdLineOpts);

            LOG.info("Palo FE starting...");
            
            FrontendOptions.init();

            // check all port
            checkAllPorts();

            if (Config.enable_bdbje_debug_mode) {
                // Start in BDB Debug mode
                BDBDebugger.get().startDebugMode(dorisHomeDir);
                return;
            }
        
            // init catalog and wait it be ready
            Catalog.getCurrentCatalog().initialize(args);
            Catalog.getCurrentCatalog().waitForReady();

            // init and start:
            // 1. QeService for MySQL Server
            // 2. FeServer for Thrift Server
            // 3. HttpServer for HTTP Server
            QeService qeService = new QeService(Config.query_port, Config.mysql_service_nio_enabled, ExecuteEnv.getInstance().getScheduler());
            FeServer feServer = new FeServer(Config.rpc_port);

            feServer.start();
            
            if (!Config.enable_http_server_v2) {
                HttpServer httpServer = new HttpServer(
                        Config.http_port,
                        Config.http_max_line_length,
                        Config.http_max_header_size,
                        Config.http_max_chunk_size
                );
                httpServer.setup();
                httpServer.start();
            } else {
                org.apache.doris.httpv2.HttpServer httpServer2 = new org.apache.doris.httpv2.HttpServer();
                httpServer2.setPort(Config.http_port);
                httpServer2.setMaxHttpPostSize(Config.jetty_server_max_http_post_size);
                httpServer2.setAcceptors(Config.jetty_server_acceptors);
                httpServer2.setSelectors(Config.jetty_server_selectors);
                httpServer2.setWorkers(Config.jetty_server_workers);
                httpServer2.start(dorisHomeDir);
            }

            qeService.start();

            ThreadPoolManager.registerAllThreadPoolMetric();

            while (true) {
                Thread.sleep(2000);
            }
        } catch (Throwable e) {
            e.printStackTrace();
        }
    }
