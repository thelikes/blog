# sliver-ext

A look into the workings of External Builders in Sliver C2.

## situational awareness
- `server/` package is for server things, including `rpc`
- `client/` package is for command execution
- `implant/` package is the implant

### (Rough) Flow of Operations
1. `client/` package receives command
2. `client/` package performs pre-process execution tasks
3. `client/` package calls RPC
4. `server/` package receives RPC
5. `server/` package sends command via RPC to `implant/` package
6. `implant/` package executes commands and sends data back to `server/` package via RPC

### Command Walkthrough: generate
The `client/` package's `generate` code starts here: [client/command/generate](https://github.com/BishopFox/sliver/blob/master/client/command/generate/commands.go):

```go
func Commands(con *console.SliverClient) []*cobra.Command {
	// [ Generate ] --------------------------------------------------------------
	generateCmd := &cobra.Command{
		Use:   consts.GenerateStr,
		Short: "Generate an implant binary",
		Long:  help.GetHelpFor([]string{consts.GenerateStr}),
		Run: func(cmd *cobra.Command, args []string) {
			GenerateCmd(cmd, con, args)
		},
		GroupID: consts.PayloadsHelpGroup,
	}
	flags.Bind("generate", true, generateCmd, func(f *pflag.FlagSet) {
		f.IntP("timeout", "t", flags.DefaultTimeout, "grpc timeout in seconds")
	})
    ...
}
```

#### "Delegation is when your boss shows confidence in your abilities—and a complete lack of interest in doing it themselves."

The above code is [cobra](https://github.com/spf13/cobra) code to direct code-execution to the `Generate()` function located in [client/command/generate/generate.go](https://github.com/BishopFox/sliver/blob/master/client/command/generate/generate.go). The function performs preprocessing, and then, based on the booolean flag `--external-builder`, redirects the flow to the `externalBuild()` function:

```go
// GenerateCmd - The main command used to generate implant binaries
func GenerateCmd(cmd *cobra.Command, con *console.SliverClient, args []string) {
    ...
	if external, _ := cmd.Flags().GetBool("external-builder"); !external {
		compile(config, save, con)
	} else {
		_, err := externalBuild(name, config, save, con)
		if err != nil {
			if err == ErrNoExternalBuilder {
				con.PrintErrorf("There are no external builders currently connected to the server\n")
				con.PrintErrorf("See 'builders' command for more information\n")
			} else if err == ErrNoValidBuilders {
				con.PrintErrorf("There are external builders connected to the server, but none can build the target you specified\n")
				con.PrintErrorf("Invalid target %s\n", fmt.Sprintf("%s:%s/%s", config.Format, config.GOOS, config.GOARCH))
				con.PrintErrorf("See 'builders' command for more information\n")
			} else {
				con.PrintErrorf("%s\n", err)
			}
			return
		}
	}
    ...
}
```

The [externalBuild()](https://github.com/BishopFox/sliver/blob/master/client/command/generate/generate.go#L785) function, also located in `generate.go` of the `generate` package, is a bigger function that's primarily concerned with setting up the build configuration and the communication channel to the "client". Client in this case is the sliver process running on the host of the external builder, started as `./sliver-server builder -c operator-multiplayer.cfg`.

The function will eventually delegate the build to the external builder via the `GenerateExternal()` function, in the *RPC* package:

```go
func externalBuild(name string, config *clientpb.ImplantConfig, save string, con *console.SliverClient) (*commonpb.File, error) {
    ...
	con.PrintInfof("Creating external build ... ")
	externalImplantConfig, err := con.Rpc.GenerateExternal(context.Background(), &clientpb.ExternalGenerateReq{
		Name:        name,
		Config:      config,
		BuilderName: externalBuilder.Name,
	})
    ...
}
```

Using the IDE to view the declaration of this function will result in viewing the following line of code located in [protobuf/rpcpb/services_grpc.pb.go](https://github.com/BishopFox/sliver/blob/master/protobuf/rpcpb/services_grpc.pb.go#L88):

```go
GenerateExternal(ctx context.Context, in *clientpb.ExternalGenerateReq, opts ...grpc.CallOption) (*clientpb.ExternalImplantConfig, error)
```

To view the actual implementation, we need to find the function in the `rpc/` package in the `server/` package. Within the [server/rpc](https://github.com/BishopFox/sliver/tree/master/server/rpc) directory, there exists numerous files named `rpc-*.go`, including the file, [rpc-generate.go](https://github.com/BishopFox/sliver/blob/master/server/rpc/rpc-generate.go). Within this file we can find the `GenerateExternal()` function of the `rpc` package.

The function performs some initial checks, such as the naming of the implant, and eventually calls `SliverExternal` of the `generate` package of the `server/` package:

```go
// Generate - Generate a new implant
func (rpc *Server) GenerateExternal(ctx context.Context, req *clientpb.ExternalGenerateReq) (*clientpb.ExternalImplantConfig, error) {
    ...
	externalConfig, err := generate.SliverExternal(name, config)
	if err != nil {
		return nil, err
	}

    core.EventBroker.Publish(core.Event{
		EventType: consts.ExternalBuildEvent,
		Data:      []byte(fmt.Sprintf("%s:%s", req.BuilderName, externalConfig.Build.ID)),
	})
	return externalConfig, err
}
```

The `generate.SliverExternal` function first generates the implant configuration object, then makes two calls to functions in the `db` package:

```go
// SliverExternal - Generates the cryptographic keys for the implant but compiles no code
func SliverExternal(name string, config *clientpb.ImplantConfig) (*clientpb.ExternalImplantConfig, error) {
	...

	build, err := GenerateConfig(name, config)
	if err != nil {
		return nil, err
	}
	config, err = db.SaveImplantConfig(config)
	if err != nil {
		return nil, err
	}

	build.ImplantConfigID = config.ID
	implantBuild, err := db.SaveImplantBuild(build)
	if err != nil {
		return nil, err
	}
    ...
}
```

As we can see, this order of operation ends with storing the implant configuration in the database.

So the question becomes, how and when does the external builder _compile_ code?

#### Call me... maybe?
If we take a look back at `rpc.GenerateExternal()` we note that the last instruction is a call to `Publish()`:

```go
// Generate - Generate a new implant
func (rpc *Server) GenerateExternal(ctx context.Context, req *clientpb.ExternalGenerateReq) (*clientpb.ExternalImplantConfig, error) {
    ...

    core.EventBroker.Publish(core.Event{
		EventType: consts.ExternalBuildEvent,
		Data:      []byte(fmt.Sprintf("%s:%s", req.BuilderName, externalConfig.Build.ID)),
	})
```

So, the answer is, an external builder compiles code via a publisher-subscriber (pub/sub) pattern, where:
1. Publisher (main sliver client/server) creates a build event
2. Subscriber (external) "builder" process listens for and handles those events

The channel is the `gRPC` stream.

Let's take a look at the code where the subscriber is setup and the implementation for their activity.

As the builder process is started with `./sliver-server builder...` we can look for the cobra code that implements this sub-scommand. We can find that code in [server/cli/builder.go](https://github.com/BishopFox/sliver/blob/master/server/cli/builder.go):

```go
func initBuilderCmd() *cobra.Command {
	builderCmd.Flags().StringP(nameFlagStr, "n", "", "Name of the builder (blank = hostname)")
	builderCmd.Flags().IntP(logLevelFlagStr, "L", 4, "Logging level: 1/fatal, 2/error, 3/warn, 4/info, 5/debug, 6/trace")
	builderCmd.Flags().StringP(operatorConfigFlagStr, "c", "", "operator config file path")
	builderCmd.Flags().StringP(operatorConfigDirFlagStr, "d", "", "operator config directory path")
	builderCmd.Flags().BoolP(quietFlagStr, "q", false, "do not write any content to stdout")

	// Artifact configuration options
	builderCmd.Flags().StringSlice(enableTargetFlagStr, []string{}, "force enable a target: format:goos/goarch")
	builderCmd.Flags().StringSlice(disableTargetFlagStr, []string{}, "force disable target arch: format:goos/goarch")

	return builderCmd
}
```

Where the declaration for the `builderCmd` object is:

```go
var builders []*builder.Builder

var builderCmd = &cobra.Command{
	Use:   "builder",
	Short: "Start the process as an external builder",
	Long:  ``,
	Run: func(cmd *cobra.Command, args []string) {
		configPath, err := cmd.Flags().GetString(operatorConfigFlagStr)
        ...
        startBuilders(cmd, config, multipleBuilders)
        ...
}
```

Continuing this flow of execution, we see that `startBuilders()` will eventually call the `Start()` function defined for the `builder` struct (Go's version of a class):

```go
// Start all builders if multpile is true or a single builder otherwise.
func startBuilders(cmd *cobra.Command, config string, multpile bool) {
    ...
	// Single builder
	if !multpile {
		...
		singleBuilder.Start()
	} else {
		// Multiple builders
        ...
				builderInst.Start()
                ...
}
```

The implementation of the `Start()` function of the `builder` struct (class) is straight-forward:

```go
// StartBuilder - main entry point for the builder
func (b *Builder) Start() {
	builderLog.Infof("Attempting to register builder: %s", b.externalBuilder.Name)
	events, err := b.buildEvents()
	if err != nil {
		os.Exit(1)
	}

	for event := range events {
		go b.handleBuildEvent(event)
	}
}
```

Or, 1) start listening for build events and 2) on event, pass execution to `handleBuildEvent()`.

The function, `func (b *Builder) handleBuildEvent()` is a rather large function. However, looking through it, we see that we have finally made it to the compilation code. After a large amount of preprocessing, we see the following switch statement in [server/builder/builder.go](https://github.com/BishopFox/sliver/blob/master/server/builder/builder.go#L201):

```go
func (b *Builder) handleBuildEvent(event *clientpb.Event) {
    ...
    switch extConfig.Config.Format {
	case clientpb.OutputFormat_SERVICE:
		fallthrough
	case clientpb.OutputFormat_EXECUTABLE:
		b.mutex.Lock()
		fPath, err = generate.SliverExecutable(extConfig.Build.Name, extConfig.Build, extConfig.Config, httpC2Config.ImplantConfig)
		b.mutex.Unlock()
	case clientpb.OutputFormat_SHARED_LIB:
		b.mutex.Lock()
		fPath, err = generate.SliverSharedLibrary(extConfig.Build.Name, extConfig.Build, extConfig.Config, httpC2Config.ImplantConfig)
		b.mutex.Unlock()
	case clientpb.OutputFormat_SHELLCODE:
		b.mutex.Lock()
		fPath, err = generate.SliverShellcode(extConfig.Build.Name, extConfig.Build, extConfig.Config, httpC2Config.ImplantConfig)
		b.mutex.Unlock()
	default:
    ...
    builderLog.Infof("Uploading '%s' to server ...", extConfig.Build.Name)
    ...
}
```

So, the flow of data is:

1. Initial Config & Build Generation:
    - ImplantConfig: Stores target specs (OS, arch), behavior settings (beacon intervals, evasion), and enabled C2 methods
    - ImplantBuild: Stores crypto materials (keys, certs) and build artifacts (checksums)
2. Database Storage:
    - One-to-many relationship between ImplantConfig and ImplantBuild
    - Each build linked via ImplantConfigID
3. Build Process:
    - `handleBuildEvent()` retrieves these stored configs
    - Uses them to compile implants with embedded keys and settings

## Detours

So what if we want to modify the compilation? The `generate.SliverExecutable` function, responsible for building the implant in an executable format and located in 
the file, [server/generate/binaries.go](https://github.com/BishopFox/sliver/blob/master/server/generate/binaries.go#L269), is a good place to start.

We can see the first operation is to create an object of the `GoConfig` struct:

```go
// SliverExecutable - Generates a sliver executable binary
func SliverExecutable(name string, build *clientpb.ImplantBuild, config *clientpb.ImplantConfig, pbC2Implant *clientpb.HTTPC2ImplantConfig) (string, error) {
	appDir := assets.GetRootAppDir()

	...
	goConfig := &gogo.GoConfig{
		CGO:        "0",
		GOOS:       config.GOOS,
		GOARCH:     config.GOARCH,
		GOROOT:     gogo.GetGoRootDir(appDir),
		GOCACHE:    gogo.GetGoCache(appDir),
		GOMODCACHE: gogo.GetGoModCache(appDir),
		GOPROXY:    getGoProxy(),
		HTTPPROXY:  getGoHttpProxy(),
		HTTPSPROXY: getGoHttpsProxy(),

		Obfuscation: config.ObfuscateSymbols,
		GOGARBLE:    goGarble(config),
	}
    ...
```

Next, we have the setup of the newly created implant directory with `renderSliverGoCode()` and a check for `goConfig.GOOS == WINDOWS`:

```go
// SliverExecutable - Generates a sliver executable binary
func SliverExecutable(name string, build *clientpb.ImplantBuild, config *clientpb.ImplantConfig, pbC2Implant *clientpb.HTTPC2ImplantConfig) (string, error) {
    ...
	pkgPath, err := renderSliverGoCode(name, build, config, goConfig, pbC2Implant)
	if err != nil {
		return "", err
	}

	dest := filepath.Join(goConfig.ProjectDir, "bin", filepath.Base(name))
	if goConfig.GOOS == WINDOWS {
		dest += ".exe"
	}
    ...
}
```

And finally, compilation flag processing and a call to `gogo.GoBuild()`:

```go
// SliverExecutable - Generates a sliver executable binary
func SliverExecutable(name string, build *clientpb.ImplantBuild, config *clientpb.ImplantConfig, pbC2Implant *clientpb.HTTPC2ImplantConfig) (string, error) {
    ...
	gcFlags := ""
	asmFlags := ""
	if config.Debug {
		gcFlags = "all=-N -l"
		ldflags = []string{}
	}
	_, err = gogo.GoBuild(*goConfig, pkgPath, dest, "", tags, ldflags, gcFlags, asmFlags)
	if err != nil {
		return "", err
	}
    ...
}
```

The `renderSliverGoCode()` is a rather convoluated function, but can be referred to for the setup of the implant build directory- `~/.sliver/slivers/<os>/<arch>/<name>/bin`. This code is important for the compilation of the implant, as it is essentially its own program that in order to build will require typical resources, depending on the language (eg, Go, Rust, C++, etc).

Taking a look at the `GoGo` package's `GoBuild()` function, we can see that the operations are used to set build parameters and lastly call `GoCmd()` or `GarbleCmd()` depending on `config.Obfuscation`.

Let's take a look at the Garble compilation function in [server/gogo/go.go](https://github.com/BishopFox/sliver/blob/master/server/gogo/go.go#L90):

```go
// GarbleCmd - Execute a go command
func GarbleCmd(config GoConfig, cwd string, command []string) ([]byte, error) {
	target := fmt.Sprintf("%s/%s", config.GOOS, config.GOARCH)
	if _, ok := ValidCompilerTargets(config)[target]; !ok {
		return nil, fmt.Errorf(fmt.Sprintf("Invalid compiler target: %s", target))
	}
	garbleBinPath := filepath.Join(config.GOROOT, "bin", "garble")
	garbleFlags := []string{"-seed=random", "-literals", "-tiny"}
	command = append(garbleFlags, command...)
	cmd := exec.Command(garbleBinPath, command...)
	cmd.Dir = cwd
	cmd.Env = []string{
		fmt.Sprintf("CC=%s", config.CC),
		fmt.Sprintf("CGO_ENABLED=%s", config.CGO),
		fmt.Sprintf("GOOS=%s", config.GOOS),
		fmt.Sprintf("GOARCH=%s", config.GOARCH),
		fmt.Sprintf("GOPATH=%s", config.ProjectDir),
		fmt.Sprintf("GOCACHE=%s", config.GOCACHE),
		fmt.Sprintf("GOMODCACHE=%s", config.GOMODCACHE),
		fmt.Sprintf("GOPROXY=%s", config.GOPROXY),
		fmt.Sprintf("HTTP_PROXY=%s", config.HTTPPROXY),
		fmt.Sprintf("HTTPS_PROXY=%s", config.HTTPSPROXY),
		fmt.Sprintf("PATH=%s:%s:%s", filepath.Join(config.GOROOT, "bin"), assets.GetZigDir(), os.Getenv("PATH")),
		fmt.Sprintf("GOGARBLE=%s", config.GOGARBLE),
		fmt.Sprintf("HOME=%s", getHomeDir()),
	}
	var stdout bytes.Buffer
	var stderr bytes.Buffer
	cmd.Stdout = &stdout
	cmd.Stderr = &stderr
	gogoLog.Debugf("--- env ---\n")
	for _, envVar := range cmd.Env {
		gogoLog.Debugf("%s\n", envVar)
	}
	gogoLog.Infof("garble cmd: '%v'", cmd)
	err := cmd.Run()
	if err != nil {
		gogoLog.Debugf("--- env ---\n")
		for _, envVar := range cmd.Env {
			gogoLog.Debugf("%s\n", envVar)
		}
		gogoLog.Errorf("--- stdout ---\n%s\n", stdout.String())
		gogoLog.Errorf("--- stderr ---\n%s\n", stderr.String())
		gogoLog.Error(err)
	}

	return stdout.Bytes(), err
}
```

As we can see, fairly straight-forward function that relies on `exec.Cmd` to execute system commands via the `cmd.Run() function.

## Conclusion
We have learned that an external builder, registered via `gRPC`, leverages a publish/subscriber pattern to listen for build events and take action on them when one is observed. We have also identified the information that is required to be passed to the external builder and baked into the implant, in order to communicate with the Sliver framework. Lastly, we have also identified the functions responsible for picking up the build event, configuring the build parameters, and executing the system command to perform the implant's compilation. Armed with this information, we can now modify Sliver to perform a different compilation and build an implant we wrote ourselves.