# A [Helm3](https://github.com/helm/helm) HTTP Wrapper With Go SDK

Helm3 abandons Helm2's Tiller architecture and uses a pure command line to perform related operations. If you want to implement related functions through Helm API, unfortunately the official does not provide similar services. However, because the official provides a relatively friendly [Helm Go SDK] (https://helm.sh/docs/topics/advanced/), we only need to do packaging on this basis. [helm-wrapper](https://github.com/opskumu/helm-wrapper) is such a Go [Gin](https://github.com/gin-gonic/gin) Web framework, combined with Helm Go SDK The encapsulated HTTP Server allows Helm-related daily command operations to achieve the same operations through the Restful API.

## Support API

Correspondence between helm's native command line and related APIs:

+ helm install
    - `POST`
    - `/api/namespaces/:namespace/releases/:release`

POST Body:

``` json
{
    "values": "",           // `--values`
    "set": [],              // `--set`
    "set_string": []        // `--set-string`
}
```

> The content of values ​​here is the same as helm install `--values` option

+ uninstall helmet
    - `DELETE`
    - `/api/namespaces/:namespace/releases/:release`
+ helm upgrade
    - `PUT`
    - `/api/namespaces/:namespace/releases/:release`

PUT Body:

``` json
{
    "values": "",           // `--values`
    "set": [],              // `--set`
    "set_string": []        // `--set-string`
}
```

> The content of values ​​here is the same as helm upgrade `--values` option

+ helm rollback
    - `PUT`
    - `/api/namespaces/:namespace/releases/:release/versions/:reversion`
+ helm list
    - `GET`
    - `/api/namespaces/:namespace/releases`
+ helm get
    - `GET`
    - `/api/namespaces/:namespace/releases/:release`

| Params | Name                                                  |
| :----- | :---------------------------------------------------- |
| info   | Support all/hooks/manifest/notes/values ​​information |

+ helm release history
    - `GET`
    - `/api/namespaces/:namespace/releases/:release/histories`

+ helm show
    - `GET`
    - `/api/charts`

| Params  | Name                                                |
| :------ | :-------------------------------------------------- |
| chart   | Specify chart name, required                        |
| info    | Support readme/values/chart information             |
| version | Support version specification, same as command line |

+ helm search repo
    - `GET`
    - `/api/repositories/charts`

| Params   | Name                     |
| :------- | :----------------------- |
| keyword  | Search keyword, required |
| version  | 指定 chart version       |
| versions | if "true", all versions  |

+ helm repo update
    - `PUT`
    - `/api/repositories`

+ helm env
    - `GET`
    - `/ api / envs`

> At present, this version is in Alpha state, and has not undergone a lot of testing, but just tested the relevant functions again. You can also customize your own version based on this.

### Response

In order to simplify, all requests return a 200 status code uniformly, and determine whether the response is normal by returning the Code value in the Body:

``` go
type respBody struct {
    Code  int         `json:"code"` // 0 or 1, 0 is ok, 1 is error
    Data  interface{} `json:"data,omitempty"`
    Error string      `json:"error,omitempty"`
}
```


## Build & Run

### Build

The source code provides a simple Makefile file. If you want to build a binary, you only need to build it in the following way.

```
make build // Build a binary version of the current host architecture
make build-linux // build the Linux version of the binary
make build-docker // Build Docker image
```

Direct build will generate a binary program named `helm-wrapper`, you can get help by:

```
$ helm-wrapper -h
Usage of helm-wrapper:
      --addr string                      server listen addr (default "0.0.0.0")
      --alsologtostderr                  log to standard error as well as files
      --config string                    helm wrapper config (default "config.yaml")
      --debug                            enable verbose output
      --kube-context string              name of the kubeconfig context to use
      --kubeconfig string                path to the kubeconfig file
      --log_backtrace_at traceLocation   when logging hits line file:N, emit a stack trace (default :0)
      --log_dir string                   If non-empty, write log files in this directory
      --logtostderr                      log to standard error instead of files (default true)
  -n, --namespace string                 namespace scope for this request
      --port string                      server listen port (default "8080")
      --registry-config string           path to the registry config file (default "/root/.config/helm/registry.json")
      --repository-cache string          path to the file containing cached repository indexes (default "/root/.cache/helm/repository")
      --repository-config string         path to the file containing repository names and URLs (default "/root/.config/helm/repositories.yaml")
      --stderrthreshold severity         logs at or above this threshold go to stderr (default 2)
  -v, --v Level                          log level for V logs
      --vmodule moduleSpec               comma-separated list of pattern=N settings for file-filtered logging
pflag: help requested
```

The key options are explained:

+ `--config` helm-wrapper configuration item, the content is as follows, mainly to specify the Helm Repo name and URL, used for Repo initialization.

```
$ cat config-example.yaml
helmRepos:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
```
+ `--kubeconfig` default If you don’t specify, use the default path, usually `~/.kube/config`. This configuration is required, which indicates the Kubernetes cluster address and access method you want to operate. How to generate the `kubeconfig` file, but there is not much introduction here. For details, see [Configure Access to Multiple Clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/ )

### Run

The operation is relatively simple. If you already have the default `kubeconfig` file locally, you only need to configure the repo configuration file required by helm-wrapper, and then execute the following command to run it. The example is as follows:

```
$ ./helm-wrapper --config config-example.yaml
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /                         --> main.main.func1 (3 handlers)
[GIN-debug] GET    /api/envs                 --> main.getHelmEnvs (3 handlers)
[GIN-debug] GET    /api/repositories/charts  --> main.listRepoCharts (3 handlers)
[GIN-debug] PUT    /api/repositories         --> main.updateRepositories (3 handlers)
[GIN-debug] GET    /api/charts               --> main.showChartInfo (3 handlers)
[GIN-debug] GET    /api/namespaces/:namespace/releases --> main.listReleases (3 handlers)
[GIN-debug] GET    /api/namespaces/:namespace/releases/:release --> main.showReleaseInfo (3 handlers)
[GIN-debug] POST   /api/namespaces/:namespace/releases/:release --> main.installRelease (3 handlers)
[GIN-debug] PUT    /api/namespaces/:namespace/releases/:release --> main.upgradeRelease (3 handlers)
[GIN-debug] DELETE /api/namespaces/:namespace/releases/:release --> main.uninstallRelease (3 handlers)
[GIN-debug] PUT    /api/namespaces/:namespace/releases/:release/versions/:reversion --> main.rollbackRelease (3 handlers)
[GIN-debug] GET    /api/namespaces/:namespace/releases/:release/status --> main.getReleaseStatus (3 handlers)
[GIN-debug] GET    /api/namespaces/:namespace/releases/:release/histories --> main.listReleaseHistories (3 handlers)
```

> The repo will be initialized at startup, so it will take some time depending on the size of the repo itself or network factors