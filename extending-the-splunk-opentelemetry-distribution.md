# Extending the Splunk OpenTelemetry distribution

The [Splunk OpenTelemetry Collector](https://github.com/signalfx/splunk-otel-collector) is a distribution of the [OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector). In addition to supporting normal OpenTelemetry collector upstream use cases, the Splunk distribution exists to support Splunk-specific use cases such as HTTP Endpoint Collector (HEC) and Splunk Infrastructure Monitoring ingest APIs. In short, it provides a unified way for our customers to receive, process, and export metric, trace, and log data to Splunk Cloud and Splunk Observability products. 

Because Splunk provides support for our distribution, the receivers, processors, and exporters that come bundled and enabled are generally tailored around our own customer use cases and our ability to support them. While Splunk provides support for a wide variety of OpenTelemetry components, unusual use cases or newer and less stable components aren't enabled by default in our distribution.

If you'd like to experiment in a non-production environment, the opportunity for you to extend the Splunk OpenTelemetry distribution is straight forward and relatively non-intimidating even for someone without an extensive development background. In this blog, I'll walk you through how I "scratched my own" itch in my role as a Google Cloud partner engineer and extended the Splunk OpenTelemetry collector to include support for Google Cloud Pub/Sub. You'll learn:

- How to set up a minimal development environment
- How to customize the source code to include the Pub/Sub receiver
- How to compile and build a custom binary
- How to configure the agent to receive Google Cloud log messages via Pub/Sub and deliver them to Splunk as HEC messages

Please note that the use case I will describe and implement in this article isn't Splunk supported -- only the official Splunk OpenTelemetry distribution is supported. Also, be aware that custom distributions can also be built with the [OpenTelemetry Collector Builder](https://github.com/open-telemetry/opentelemetry-collector/tree/main/cmd/builder). Splunk has not yet migrated to using the builder and the process won't be covered here.

## Prerequisites

* You aren't afraid of the CLI
* You aren't afraid to compile your own code.
* A working [Go development environment](https://go.dev/doc/install).
* A Google Cloud log sink along with a corresponding Pub/Sub topic and subscription configured
* A Google Cloud service account exported JSON key with permission to subscribe to the aforementioned subscription
* [Optional] Docker or podman installed.

## Preparing the source

The basic steps for preparing the source code are as follows:

- Clone the Splunk OpenTelemetry Distribution from Github
- Add new components to the import in `internal/components/components.go`
- Add new components to `Get` function in `internal/components/components.go`
- Update tests by adding components to `TestDefaultComponents` function in `internal/components/components_test.go`


### Clone the Splunk OpenTelemetry Distribution

```
$ git clone https://github.com/signalfx/splunk-otel-collector
Cloning into 'splunk-otel-collector'...
remote: Enumerating objects: 11502, done.
remote: Counting objects: 100% (464/464), done.
remote: Compressing objects: 100% (290/290), done.
remote: Total 11502 (delta 216), reused 333 (delta 166), pack-reused 11038
Receiving objects: 100% (11502/11502), 5.81 MiB | 14.94 MiB/s, done.
Resolving deltas: 100% (6990/6990), done.
```

### Install tools

The `Makefile` included with Splunk's OpenTelemetry Distribution includes a target which installs prerequisite build tooling. Make sure you run this command to prepare your local development environment.

```
$ make install-tools
go install github.com/client9/misspell/cmd/misspell@v0.3.4
go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.45.0
go install github.com/google/addlicense@v0.0.0-20200906110928-a0294312aa76
go install github.com/jstemmer/go-junit-report@v0.9.1
go install github.com/ory/go-acc@v0.2.8
go: downloading github.com/ory/go-acc v0.2.8
go: downloading golang.org/x/sys v0.0.0-20220319134239-a9b59b0215f8
go install github.com/pavius/impi/cmd/impi@v0.0.3
go: finding module for package github.com/kisielk/gotool
go: found github.com/kisielk/gotool in github.com/kisielk/gotool v1.0.0
go install github.com/tcnksm/ghr@v0.14.0
go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest
```

### Edit necessary files

For the purposes of this exercise, I'm going to create a simple pipeline which leverages my new Pub/Sub receiver and transforms them into Splunk HEC log messages. Because the Pub/Sub receiver and the `logstransform` processor are not enabled by default with the Splunk distribution, I will need to enable the components in the source code and create a custom build based upon these changes. I've provided `diff` examples below to illustrate the changes made.

The following is a diff demonstrating the changes made to `internal/components/components.go`:

```
diff --git a/internal/components/components.go b/internal/components/components.go
index 398ba46..40ab407 100644
--- a/internal/components/components.go
+++ b/internal/components/components.go
@@ -34,6 +34,7 @@ import (
        "github.com/open-telemetry/opentelemetry-collector-contrib/processor/filterprocessor"
        "github.com/open-telemetry/opentelemetry-collector-contrib/processor/groupbyattrsprocessor"
        "github.com/open-telemetry/opentelemetry-collector-contrib/processor/k8sattributesprocessor"
+       "github.com/open-telemetry/opentelemetry-collector-contrib/processor/logstransformprocessor"
        "github.com/open-telemetry/opentelemetry-collector-contrib/processor/metricstransformprocessor"
        "github.com/open-telemetry/opentelemetry-collector-contrib/processor/probabilisticsamplerprocessor"
        "github.com/open-telemetry/opentelemetry-collector-contrib/processor/resourcedetectionprocessor"
@@ -46,6 +47,7 @@ import (
        "github.com/open-telemetry/opentelemetry-collector-contrib/receiver/collectdreceiver"
        "github.com/open-telemetry/opentelemetry-collector-contrib/receiver/filelogreceiver"
        "github.com/open-telemetry/opentelemetry-collector-contrib/receiver/fluentforwardreceiver"
+       "github.com/open-telemetry/opentelemetry-collector-contrib/receiver/googlecloudpubsubreceiver"
        "github.com/open-telemetry/opentelemetry-collector-contrib/receiver/hostmetricsreceiver"
        "github.com/open-telemetry/opentelemetry-collector-contrib/receiver/jaegerreceiver"
        "github.com/open-telemetry/opentelemetry-collector-contrib/receiver/journaldreceiver"
@@ -112,6 +114,7 @@ func Get() (component.Factories, error) {
                fluentforwardreceiver.NewFactory(),
                filelogreceiver.NewFactory(),
                hostmetricsreceiver.NewFactory(),
+               googlecloudpubsubreceiver.NewFactory(),
                jaegerreceiver.NewFactory(),
                journaldreceiver.NewFactory(),
                k8sclusterreceiver.NewFactory(),
@@ -160,6 +163,7 @@ func Get() (component.Factories, error) {
                filterprocessor.NewFactory(),
                groupbyattrsprocessor.NewFactory(),
                k8sattributesprocessor.NewFactory(),
+               logstransformprocessor.NewFactory(),
                memorylimiterprocessor.NewFactory(),
                metricstransformprocessor.NewFactory(),
                probabilisticsamplerprocessor.NewFactory(),
```


The following is a diff demonstrating the changes made to `internal/components/components_test.go`:

```
diff --git a/internal/components/components_test.go b/internal/components/components_test.go
index c7ee470..df10571 100644
--- a/internal/components/components_test.go
+++ b/internal/components/components_test.go
@@ -46,6 +46,7 @@ func TestDefaultComponents(t *testing.T) {
                "databricks",
                "filelog",
                "fluentforward",
+               "googlecloudpubsubreceiver",
                "hostmetrics",
                "jaeger",
                "journald",
@@ -75,6 +76,7 @@ func TestDefaultComponents(t *testing.T) {
                "filter",
                "groupbyattrs",
                "k8sattributes",
+               "logstransformprocessor",
                "memory_limiter",
                "metricstransform",
                "probabilistic_sampler",
```


### Get new dependencies

Before compiling, retrieve the new dependencies for the `googlecloudpubsubreceiver` and `logstransformprocessor` components.

```
$ go get github.com/open-telemetry/opentelemetry-collector-contrib/receiver/googlecloudpubsubreceiver
go: upgraded cloud.google.com/go v0.100.2 => v0.102.0
go: upgraded cloud.google.com/go/pubsub v1.3.1 => v1.22.2
go: added github.com/open-telemetry/opentelemetry-collector-contrib/receiver/googlecloudpubsubreceiver v0.53.0
```
```
$ go get github.com/open-telemetry/opentelemetry-collector-contrib/processor/logstransformprocessor
go: downloading github.com/open-telemetry/opentelemetry-collector-contrib v0.54.0
go: downloading github.com/open-telemetry/opentelemetry-collector-contrib/processor/logstransformprocessor v0.54.0
go: downloading github.com/open-telemetry/opentelemetry-collector-contrib/pkg/stanza v0.54.0
go: downloading github.com/open-telemetry/opentelemetry-collector-contrib/extension/storage v0.54.0
go: downloading github.com/open-telemetry/opentelemetry-collector-contrib/internal/coreinternal v0.54.0
go: upgraded github.com/knadh/koanf v1.4.1 => v1.4.2
go: upgraded github.com/open-telemetry/opentelemetry-collector-contrib/extension/storage v0.53.0 => v0.54.0
go: upgraded github.com/open-telemetry/opentelemetry-collector-contrib/internal/coreinternal v0.53.0 => v0.54.0
go: upgraded github.com/open-telemetry/opentelemetry-collector-contrib/pkg/stanza v0.53.0 => v0.54.0
go: added github.com/open-telemetry/opentelemetry-collector-contrib/processor/logstransformprocessor v0.54.0
go: upgraded go.opentelemetry.io/collector v0.53.1-0.20220615184617-4cefca87d2c6 => v0.54.0
go: upgraded go.opentelemetry.io/collector/pdata v0.53.1-0.20220615184617-4cefca87d2c6 => v0.54.0
go: upgraded go.opentelemetry.io/collector/semconv v0.53.1-0.20220615184617-4cefca87d2c6 => v0.54.0
go: upgraded k8s.io/api v0.24.1 => v0.24.2
go: upgraded k8s.io/apimachinery v0.24.1 => v0.24.2
go: upgraded k8s.io/client-go v0.24.1 => v0.24.2
```

### Make binary


Now that the necessary changes have been made to the source code and the dependencies retrieved, it's time to build new binaries. The `Makefile` included with the project makes this extremely easy. To build for the local operating system and platform, simply run `make otelcol` as shown below.

```
$ make otelcol
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/otelcol_darwin_amd64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/otelcol
ln -sf otelcol_darwin_amd64 ./bin/otelcol

```

And since Go also supports cross compilation, you can also generate binaries for other target platforms other than the one you are building on. For example, you can easily generate a Linux x86 binary from a Darwin x86 machine.

```
$ make binaries-all-sys                                                                                                                                                                                                  GOOS=darwin  GOARCH=amd64 /Library/Developer/CommandLineTools/usr/bin/make otelcol
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/otelcol_darwin_amd64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/otelcol
ln -sf otelcol_darwin_amd64 ./bin/otelcol
GOOS=darwin  GOARCH=amd64 /Library/Developer/CommandLineTools/usr/bin/make translatesfx
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/translatesfx_darwin_amd64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/translatesfx
ln -sf translatesfx_darwin_amd64 ./bin/translatesfx
GOOS=darwin  GOARCH=amd64 /Library/Developer/CommandLineTools/usr/bin/make migratecheckpoint
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/migratecheckpoint_darwin_amd64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/migratecheckpoint
ln -sf migratecheckpoint_darwin_amd64 ./bin/migratecheckpoint
GOOS=linux   GOARCH=amd64 /Library/Developer/CommandLineTools/usr/bin/make otelcol
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/otelcol_linux_amd64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/otelcol
ln -sf otelcol_linux_amd64 ./bin/otelcol
GOOS=linux   GOARCH=amd64 /Library/Developer/CommandLineTools/usr/bin/make translatesfx
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/translatesfx_linux_amd64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/translatesfx
ln -sf translatesfx_linux_amd64 ./bin/translatesfx
GOOS=linux   GOARCH=amd64 /Library/Developer/CommandLineTools/usr/bin/make migratecheckpoint
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/migratecheckpoint_linux_amd64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/migratecheckpoint
ln -sf migratecheckpoint_linux_amd64 ./bin/migratecheckpoint
GOOS=linux   GOARCH=arm64 /Library/Developer/CommandLineTools/usr/bin/make otelcol
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/otelcol_linux_arm64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/otelcol
ln -sf otelcol_linux_arm64 ./bin/otelcol
GOOS=linux   GOARCH=arm64 /Library/Developer/CommandLineTools/usr/bin/make translatesfx
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/translatesfx_linux_arm64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/translatesfx
ln -sf translatesfx_linux_arm64 ./bin/translatesfx
GOOS=linux   GOARCH=arm64 /Library/Developer/CommandLineTools/usr/bin/make migratecheckpoint
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/migratecheckpoint_linux_arm64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/migratecheckpoint
ln -sf migratecheckpoint_linux_arm64 ./bin/migratecheckpoint
GOOS=windows GOARCH=amd64 EXTENSION=.exe /Library/Developer/CommandLineTools/usr/bin/make otelcol
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/otelcol_windows_amd64.exe -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/otelcol
ln -sf otelcol_windows_amd64.exe ./bin/otelcol
GOOS=windows GOARCH=amd64 EXTENSION=.exe /Library/Developer/CommandLineTools/usr/bin/make translatesfx
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/translatesfx_windows_amd64.exe -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/translatesfx
ln -sf translatesfx_windows_amd64.exe ./bin/translatesfx
GOOS=windows GOARCH=amd64 EXTENSION=.exe /Library/Developer/CommandLineTools/usr/bin/make migratecheckpoint
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/migratecheckpoint_windows_amd64.exe -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/migratecheckpoint
ln -sf migratecheckpoint_windows_amd64.exe ./bin/migratecheckpoint
```

The `Makefile` also provides capabilities to generate rpm packages for installation on a Red Hat or CentOS system. Note for this to work you must also have Docker installed.

```
$ make rpm-package
/Library/Developer/CommandLineTools/usr/bin/make binaries-linux_amd64
GOOS=linux   GOARCH=amd64 /Library/Developer/CommandLineTools/usr/bin/make otelcol
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/otelcol_linux_amd64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/otelcol
ln -sf otelcol_linux_amd64 ./bin/otelcol
GOOS=linux   GOARCH=amd64 /Library/Developer/CommandLineTools/usr/bin/make translatesfx
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/translatesfx_linux_amd64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/translatesfx
ln -sf translatesfx_linux_amd64 ./bin/translatesfx
GOOS=linux   GOARCH=amd64 /Library/Developer/CommandLineTools/usr/bin/make migratecheckpoint
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/migratecheckpoint_linux_amd64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/migratecheckpoint
ln -sf migratecheckpoint_linux_amd64 ./bin/migratecheckpoint
docker build -t otelcol-fpm internal/buildscripts/packaging/fpm
STEP 1/10: FROM debian:10
STEP 2/10: VOLUME /repo
...
+ rpm -qpli '/repo/dist//splunk-otel-collector-0.53.1~4-g5729a1e*.x86_64.rpm'
```

Similarly, the same can be done for a .deb package for use on an Ubuntu or other Debian-based distribution.

```
$ make deb-package
/Library/Developer/CommandLineTools/usr/bin/make binaries-linux_amd64
GOOS=linux   GOARCH=amd64 /Library/Developer/CommandLineTools/usr/bin/make otelcol
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/otelcol_linux_amd64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/otelcol
ln -sf otelcol_linux_amd64 ./bin/otelcol
GOOS=linux   GOARCH=amd64 /Library/Developer/CommandLineTools/usr/bin/make translatesfx
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/translatesfx_linux_amd64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/translatesfx
ln -sf translatesfx_linux_amd64 ./bin/translatesfx
GOOS=linux   GOARCH=amd64 /Library/Developer/CommandLineTools/usr/bin/make migratecheckpoint
go generate ./...
GO111MODULE=on CGO_ENABLED=0 go build -o ./bin/migratecheckpoint_linux_amd64 -ldflags "-X github.com/signalfx/splunk-otel-collector/internal/version.Version=v0.53.1-4-g5729a1e -X go.opentelemetry.io/collector/internal/version.Version=v0.53.1-4-g5729a1e" ./cmd/migratecheckpoint
ln -sf migratecheckpoint_linux_amd64 ./bin/migratecheckpoint
docker build -t otelcol-fpm internal/buildscripts/packaging/fpm
STEP 1/10: FROM debian:10
STEP 2/10: VOLUME /repo
--> Using cache 6358c550dcce0e555be8196cb66b12a7e252ca9e101a1f797324e429dad7bc07
--> 6358c550dcc
...

```

After the build process for the deb or rpm build is complete, you will find the package artifact in the `dist/` directory.

```
$ ls -al dist                                                                                                                                                                                                           total 918320
drwx------@  4 mhite  staff        128 Jun 22 14:13 .
drwxr-xr-x  26 mhite  staff        832 Jun 22 14:12 ..
-rw-------@  1 mhite  staff  226006452 Jun 22 14:09 splunk-otel-collector-0.53.1~4_g5729a1e-1.x86_64.rpm
-rw-------@  1 mhite  staff  226641502 Jun 22 14:13 splunk-otel-collector_0.53.1-4-g5729a1e_amd64.deb
```

Similarly, you will find the binary builds in the `bin/` directory.

```
ls -alh bin                                                                                                                                                                                                           total 1397312
drwxr-xr-x  17 mhite  staff   544B Jun 22 14:12 .
drwxr-xr-x  26 mhite  staff   832B Jun 22 14:12 ..
lrwxr-xr-x   1 mhite  staff    29B Jun 22 14:12 migratecheckpoint -> migratecheckpoint_linux_amd64
-rwxr-xr-x   1 mhite  staff   2.9M Jun 22 13:12 migratecheckpoint_darwin_amd64
-rwxr-xr-x   1 mhite  staff   3.0M Jun 22 14:12 migratecheckpoint_linux_amd64
-rwxr-xr-x   1 mhite  staff   3.0M Jun 22 13:17 migratecheckpoint_linux_arm64
-rwxr-xr-x   1 mhite  staff   3.0M Jun 22 13:19 migratecheckpoint_windows_amd64.exe
lrwxr-xr-x   1 mhite  staff    19B Jun 22 14:12 otelcol -> otelcol_linux_amd64
-rwxr-xr-x   1 mhite  staff   164M Jun 22 13:12 otelcol_darwin_amd64
-rwxr-xr-x   1 mhite  staff   166M Jun 22 14:12 otelcol_linux_amd64
-rwxr-xr-x   1 mhite  staff   162M Jun 22 13:17 otelcol_linux_arm64
-rwxr-xr-x   1 mhite  staff   166M Jun 22 13:19 otelcol_windows_amd64.exe
lrwxr-xr-x   1 mhite  staff    24B Jun 22 14:12 translatesfx -> translatesfx_linux_amd64
-rwxr-xr-x   1 mhite  staff   3.0M Jun 22 13:12 translatesfx_darwin_amd64
-rwxr-xr-x   1 mhite  staff   3.1M Jun 22 14:12 translatesfx_linux_amd64
-rwxr-xr-x   1 mhite  staff   3.1M Jun 22 13:17 translatesfx_linux_arm64
-rwxr-xr-x   1 mhite  staff   3.2M Jun 22 13:19 translatesfx_windows_amd64.exe
```

### Create a configuration file

Next, let's create a configuration file which exercises the new components. Our configuration will handle the following tasks:

- Using the `batch` processor, log messages are batched into 10 second windows.
- Using the `resourcedetection` processor, the agent hostname is detected. Messages delivered to HEC via this agent will have their `host` metadata attribute set to the local hostname of the agent machine.
- Using the `logtransformer` processor, the Pub/Sub transported Google Cloud log message is parsed as JSON and the timestamp is extracted.
- The `googlecloudpubsub` receiver is configured to pull messages from a subscription that receives Google Cloud log messages.
- The `splunk_hec` exported is configure to deliver messages to a Splunk HEC endpoint.
- Finally, a log pipeline is constructed using the aforementioned components.


```
processors:
  batch:
    # gather messages into 10s batches
    timeout: 10s
  resourcedetection:
    # this will ensure the "host" field of our hec message is populated
    detectors: ["system"]
    system:
      hostname_sources: ["os"]
  logstransform:
    operators:
      # this parses the payload as JSON and puts it in internal key "attributes"
      # everything in attributes ends up as a HEC "fields" key/value
      - type: json_parser
        parse_from: body
        # grab timestamp
        timestamp:
          parse_from: attributes.timestamp
          layout: '%Y-%m-%dT%H:%M:%S.%LZ'
      # now that we've parsed the payload as JSON, blow away current internal key "body"
      - type: remove
        field: body
      # hacky way to encapsulate parsed JSON inside "data" so it works with the GCP-TA
      - type: add
        field: body.data
        value: EXPR(attributes)
      # let's blow away attributes so we don't send any "fields" to HEC
      - type: remove
        field: attributes

extensions:
  health_check:
      endpoint: 0.0.0.0:13133
  pprof:
      endpoint: :1888
  zpages:

receivers:
  googlecloudpubsub:
    project: "${GOOGLE_PROJECT}"
    subscription: "${GOOGLE_SUBSCRIPTION}"
    encoding: raw_text

exporters:
  splunk_hec:
    token: "${SPLUNK_HEC_TOKEN}"
    endpoint: "${SPLUNK_HEC_URL}"
    source: "otel"
    index: "${SPLUNK_INDEX}"
    sourcetype: "google:gcp:pubsub:message"
    max_connections: 20
    disable_compression: false
    timeout: 10s
    tls:
      insecure_skip_verify: true

service:
  extensions: [ pprof, zpages, health_check ]
  pipelines:
    logs:
      receivers: [ googlecloudpubsub ]
      processors: [ batch, resourcedetection, logstransform ]
      exporters: [ splunk_hec ]

```

### Export environment variables

Since our configuration file references multiple environment variables, we'll need to declare them in our shell environment for the purposes of our prototype.

These environment variables are described below:

- `SPLUNK_HEC_TOKEN` - Splunk HEC token, ie. `97c79685-c5f2-402a-be39-7a13460fbced`
- `SPLUNK_HEC_URL` - URL path to the Splunk HEC endpoint, ie. https://mysplunkhec.com:8088/
- `SPLUNK_INDEX` - Splunk index name to store log messages
- `GOOGLE_APPLICATION_CREDENTIALS` - Path to GCP service account JSON credential
- `GOOGLE_PROJECT` - GCP project name
- `GOOGLE_SUBSCRIPTION` - GCP Pub/Sub subscription path, ie. `projects/[PROJECT NAME]/subscriptions/[SUBCRIPTION-NAME]`

Export each of them before launching the OpenTelemetry agent.

```
$ export SPLUNK_HEC_URL="https://mysplunkhec.com:8088"
$ export SPLUNK_HEC_TOKEN="97c79685-c5f2-402a-be39-7a13460fbced"      
$ export SPLUNK_INDEX="oteltest"                                                                                                                               
$ export GOOGLE_APPLICATION_CREDENTIALS="/Users/mhite/repo/blog/custom-agent/splunk-otel-collector/gcp.json" 
$ export GOOGLE_PROJECT="your-project-name"
$ export GOOGLE_SUBSCRIPTION="projects/your-project-name/subscriptions/your-log-subscription-name"
```

### Test it

All that's left is for you to fire up new agent and point it at your configuration file. With a little luck, you'll see see your log sink exported Google Cloud logs in Splunk!

## Resources

- [Splunk HTTP Event Collector (HEC) Exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/exporter/splunkhecexporter/README.md)
- [Google Pub/Sub Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/googlecloudpubsubreceiver/README.md)
- [Batch Processor](https://github.com/open-telemetry/opentelemetry-collector/blob/main/processor/batchprocessor/README.md)
- [Log Processing Operators](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/pkg/stanza/docs/operators)
- [Resource Detection Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/resourcedetectionprocessor)
- [OpenTelemetry local development](https://github.com/signalfx/splunk-otel-collector/blob/main/CONTRIBUTING.md#local-development)

