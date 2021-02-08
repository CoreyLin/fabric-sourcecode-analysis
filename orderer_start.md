# 排序节点Orderer的启动流程源码分析

排序节点Orderer的启动入口在`orderer/common/server/main.go`中的`func Main()`

话不多说，直接进入代码分析启动流程

```go
// Main is the entry point of orderer process
func Main() {
	fullCmd := kingpin.MustParse(app.Parse(os.Args[1:]))

	// "version" command
	if fullCmd == version.FullCommand() {
		fmt.Println(metadata.GetVersionInfo())
		return
	}
	// Load解析orderer YAML文件和环境，生成一个适合配置使用的struct，失败时返回error。
	conf, err := localconfig.Load()
	if err != nil {
		logger.Error("failed to parse config: ", err)
		os.Exit(1)
	}
	initializeLogging()

	prettyPrintStruct(conf)

	// 获取一个加密服务提供者，默认是BCCSP，BCCSP是区块链加密服务提供者，它提供加密标准和算法的实现。
	cryptoProvider := factory.GetDefault()

	// 根据配置加载本地MSP，然后获取默认签名身份
	signer, signErr := loadLocalMSP(conf).GetDefaultSigningIdentity()
	if signErr != nil {
		logger.Panicf("Failed to get local MSP identity: %s", signErr)
	}

	// 启动运维监控子系统，即operation系统。这是从Fabric 1.4开始新增的一套RESTFUL运维服务，提供了部分运维管理功能：
	// 1.日志级别管理
	// 2.健康检查
	// 3.可用Prometheus消费的系统运行指标
	opsSystem := newOperationsSystem(conf.Operations, conf.Metrics)
	if err = opsSystem.Start(); err != nil {
		logger.Panicf("failed to start operations subsystem: %s", err)
	}
	defer opsSystem.Stop()
	metricsProvider := opsSystem.Provider
	logObserver := floggingmetrics.NewObserver(metricsProvider)
	flogging.SetObserver(logObserver)

	// 利用前面生成的配置初始化grpc server的ServerConfig和GRPCServer。这当中会利用SecureOptions、KeepaliveOptions来保存TLS的公私钥，
	// C/S两端的证书以及呼应时间、等待时间等。
	serverConfig := initializeServerConfig(conf, metricsProvider)
	grpcServer := initializeGrpcServer(conf, serverConfig)
	// 初始化caManager。其中clientRootCAs是一组PEM编码的X509证书颁发机构即CA，用于服务器验证客户端的证书
	caMgr := &caManager{
		appRootCAsByChain:     make(map[string][][]byte),
		ordererRootCAsByChain: make(map[string][][]byte),
		clientRootCAs:         serverConfig.SecOpts.ClientRootCAs,
	}

	lf, err := createLedgerFactory(conf, metricsProvider)
	if err != nil {
		logger.Panicf("Failed to create ledger factory: %v", err)
	}

	var bootstrapBlock *cb.Block
	switch conf.General.BootstrapMethod {
	// 以文件方式引导（bootstrap）orderer节点
	case "file":
		if len(lf.ChannelIDs()) > 0 {
			logger.Info("Not bootstrapping the system channel because of existing channels")
			break
		}

		// 根据配置中的BootstrapFile，生成创世区块，创世区块用于引导生成账本。
		bootstrapBlock = file.New(conf.General.BootstrapFile).GenesisBlock()
		// 对创世区块进行校验，判断这个区块是否可以用作引导区块。一个引导区块是系统通道（system channel）的区块，需要包含一个ConsortiumsConfig，即联盟配置；另外，区块号必须是0.
		if err := onboarding.ValidateBootstrapBlock(bootstrapBlock, cryptoProvider); err != nil {
			logger.Panicf("Failed validating bootstrap block: %v", err)
		}

		if bootstrapBlock.Header.Number > 0 {
			logger.Infof("Not bootstrapping the system channel because the bootstrap block number is %d (>0), replication is needed", bootstrapBlock.Header.Number)
			break
		}

		// bootstrapping with a genesis block (i.e. bootstrap block number = 0)
		// generate the system channel with a genesis block.
		logger.Info("Bootstrapping the system channel")
		// 使用创世区块生成系统通道
		initializeBootstrapChannel(bootstrapBlock, lf)
	case "none":
		bootstrapBlock = initSystemChannelWithJoinBlock(conf, cryptoProvider, lf)
	default:
		logger.Panicf("Unknown bootstrap method: %s", conf.General.BootstrapMethod)
	}

	// 选择系统通道中bootstrap区块和最新的一次配置区块（config block）中区块号更大的那个区块，来作为集群启动区块（cluster boot block）
	// select the highest numbered block among the bootstrap block and the last config block if the system channel.
	sysChanConfigBlock := extractSystemChannel(lf, cryptoProvider)
	clusterBootBlock := selectClusterBootBlock(bootstrapBlock, sysChanConfigBlock)

	// determine whether the orderer is of cluster type
	var isClusterType bool
	if clusterBootBlock == nil {
		logger.Infof("Starting without a system channel")
		isClusterType = true
	} else {
		// 从集群启动区块中获取系统通道的通道ID，以及共识类型
		sysChanID, err := protoutil.GetChannelIDFromBlock(clusterBootBlock)
		if err != nil {
			logger.Panicf("Failed getting channel ID from clusterBootBlock: %s", err)
		}

		consensusTypeName := consensusType(clusterBootBlock, cryptoProvider)
		logger.Infof("Starting with system channel: %s, consensus type: %s", sysChanID, consensusTypeName)
		_, isClusterType = clusterTypes[consensusTypeName]
	}

	// configure following artifacts properly if orderer is of cluster type
	var repInitiator *onboarding.ReplicationInitiator
	clusterServerConfig := serverConfig
	clusterGRPCServer := grpcServer // by default, cluster shares the same grpc server
	var clusterClientConfig comm.ClientConfig
	var clusterDialer *cluster.PredicateDialer

	var reuseGrpcListener bool
	var serversToUpdate []*comm.GRPCServer

	if isClusterType {
		logger.Infof("Setting up cluster")
		// 初始化集群客户端配置，签名、私钥、证书和TLS等
		clusterClientConfig = initializeClusterClientConfig(conf)
		clusterDialer = &cluster.PredicateDialer{
			Config: clusterClientConfig,
		}

		if reuseGrpcListener = reuseListener(conf); !reuseGrpcListener {
			clusterServerConfig, clusterGRPCServer = configureClusterListener(conf, serverConfig, ioutil.ReadFile)
		}

		// If we have a separate gRPC server for the cluster,
		// we need to update its TLS CA certificate pool.
		serversToUpdate = append(serversToUpdate, clusterGRPCServer)

		// If the orderer has a system channel and is of cluster type, it may have
		// to replicate first.
		if clusterBootBlock != nil {
			// When we are bootstrapping with a clusterBootBlock with number >0,
			// replication will be performed. Only clusters that are equipped with
			// a recent config block (number i.e. >0) can replicate. This will
			// replicate all channels if the clusterBootBlock number > system-channel
			// height (i.e. there is a gap in the ledger).
			// 执行复制：当用区块号大于0的clusterBootBlock引导时，将执行复制，此时当前节点的状态落后于其他节点。
			// 只有配置了一个最近配置区块(区块号大于0)的集群才能执行复制。
			// 如果clusterBootBlock区块号大于系统通道区块高度(即在账本中有gap)，则将复制所有通道。
			// 此处的复制指的是从其他节点拉取所有通道到当前节点。
			repInitiator = onboarding.NewReplicationInitiator(lf, clusterBootBlock, conf, clusterClientConfig.SecOpts, signer, cryptoProvider)
			repInitiator.ReplicateIfNeeded(clusterBootBlock)
			// With BootstrapMethod == "none", the bootstrapBlock comes from a
			// join-block. If it exists, we need to remove the system channel
			// join-block from the filerepo.
			if conf.General.BootstrapMethod == "none" && bootstrapBlock != nil {
				discardSystemChannelJoinBlock(conf, bootstrapBlock)
			}
		}
	}

	identityBytes, err := signer.Serialize()
	if err != nil {
		logger.Panicf("Failed serializing signing identity: %v", err)
	}

	expirationLogger := flogging.MustGetLogger("certmonitor")
	// 检查节点的证书是否要过期了，会提前7天告警
	crypto.TrackExpiration(
		serverConfig.SecOpts.UseTLS,
		serverConfig.SecOpts.Certificate,
		[][]byte{clusterClientConfig.SecOpts.Certificate},
		identityBytes,
		expirationLogger.Infof,
		expirationLogger.Warnf, // This can be used to piggyback a metric event in the future
		time.Now(),
		time.AfterFunc)

	// if cluster is reusing client-facing server, then it is already
	// appended to serversToUpdate at this point.
	if grpcServer.MutualTLSRequired() && !reuseGrpcListener {
		serversToUpdate = append(serversToUpdate, grpcServer)
	}

	// TLS连接认证的回调函数，更新每个通道的TLS客户端root CA证书
	tlsCallback := func(bundle *channelconfig.Bundle) {
		logger.Debug("Executing callback to update root CAs")
		caMgr.updateTrustedRoots(bundle, serversToUpdate...)
		if isClusterType {
			caMgr.updateClusterDialer(
				clusterDialer,
				clusterClientConfig.SecOpts.ServerRootCAs,
			)
		}
	}

	// 多通道注册初始化--创建数据存储路径，包括索引数据库和通道区块数据
	manager := initializeMultichannelRegistrar(
		clusterBootBlock,
		repInitiator,
		clusterDialer,
		clusterServerConfig,
		clusterGRPCServer,
		conf,
		signer,
		metricsProvider,
		opsSystem,
		lf,
		cryptoProvider,
		tlsCallback,
	)

	adminServer := newAdminServer(conf.Admin)
	adminServer.RegisterHandler(
		channelparticipation.URLBaseV1,
		channelparticipation.NewHTTPHandler(conf.ChannelParticipation, manager),
		conf.Admin.TLS.Enabled,
	)
	if err = adminServer.Start(); err != nil {
		logger.Panicf("failed to start admin server: %s", err)
	}
	defer adminServer.Stop()

	mutualTLS := serverConfig.SecOpts.UseTLS && serverConfig.SecOpts.RequireClientCert
	// 基于广播目标和账本Reader创建一个ab.AtomicBroadcastServer，即排序服务server
	server := NewServer(
		manager,
		metricsProvider,
		&conf.Debug,
		conf.General.Authentication.TimeWindow,
		mutualTLS,
		conf.General.Authentication.NoExpirationChecks,
	)

	logger.Infof("Starting %s", metadata.GetVersionInfo())
	handleSignals(addPlatformSignals(map[os.Signal]func(){
		syscall.SIGTERM: func() {
			grpcServer.Stop()
			if clusterGRPCServer != grpcServer {
				clusterGRPCServer.Stop()
			}
		},
	}))

	if !reuseGrpcListener && isClusterType {
		logger.Info("Starting cluster listener on", clusterGRPCServer.Address())
		go clusterGRPCServer.Start()
	}

	// 初始化Profile服务，用于启动监听
	if conf.General.Profile.Enabled {
		go initializeProfilingService(conf)
	}
	// 在grpc server中注册AtomicBroadcastServer即排序服务，并启动grpc server以处理从peer接收到的请求
	ab.RegisterAtomicBroadcastServer(grpcServer.Server(), server)
	logger.Info("Beginning to serve requests")
	if err := grpcServer.Start(); err != nil {
		logger.Fatalf("Atomic Broadcast gRPC server has terminated while serving requests due to: %v", err)
	}
}
```