# kpx - Kerberos proxy with dynamic proxy selection

**kpx** is a configurable authentication proxy, that can be used to:

- route traffic to **different remote proxies** based on url
- **stop traffic** for specific url
- **inject credential** for remote proxies, allowing to use proxy without setting credentials
- support **kerberos** and **basic** credentials
- support **kerberos native** authentication on Windows and Linux (and MacOS?)
- support remote **http** and **socks** proxies
- **internally developed**, allowing to add features when necessary, like **proxy failover**, **regex rules**, **password caching**, **PAC support**, ...
- **multi-platform binaries**, for Windows, Linux and MacOS
- support automatic **update** and **restart** when configured
- experimental features like `connection-pools` (reuse http connections when possible) and `hosts-cache` (cache proxy lookup result by host:port, incompatible with url matching)

Alternatives tools that can be used:

- **Fiddler**, a debugging proxy that inspect/modify flows: https://www.telerik.com/fiddler
- **Charles**, a debugging proxy that can inspect flows: https://www.charlesproxy.com
- **Px**, a ntlm/kerberos proxy written in python: https://github.com/genotrance/px
- **Alpaca**, a ntlm proxy written in go: https://github.com/samuong/alpaca

Also check the [Notes](#notes) below for specific configuration tips.

## Installation

Download the latest release from the releases page.

## TL;DR

Start a krb-proxy on `127.0.0.1:8888`:
```shell
$ krb-proxy -u user_login@example.com -l 8888 proxy:8080
```

Start a krb-proxy on `0.0.0.0:8888`:
```shell
$ krb-proxy -u user_login@example.com -l 0.0.0.0:8888 proxy:8080
```

Start a krb-proxy with the default `krb-proxy.yaml` config file:
```shell
$ krb-proxy
```

Start a krb-proxy with a specific my-config.yaml file:
```shell
$ krb-proxy -c my-config.yaml
```

## Usage

The kpx binary comes with default kerberos settings, however you might want to add specific configurations related to you kerberos environment, like default domain names or aliases.

An example of how to do this can be found [here](./build).

### Temporary proxy without configuration file

> When used like this, the program will exit automatically after **1 hour**, to prevent kpx from running forever.

```shell
# Start a proxy on port 8888
$ kpx -u user_login@example.com -l 8888 proxy:8080
Credential [user] - Enter password for user 'user_login': *********
2022/08/24 09:07:17 [-] Proxy build 2022-08-23 15:30:59 started on 127.0.0.1:1237
2022/08/24 09:07:17 [-] Use http://127.0.0.1:1237 as you proxy url
2022/08/24 09:07:17 [-] Proxy pac is available at http://127.0.0.1:1237/proxy.pac
2022/08/24 09:07:17 [-] Proxy will exit automatically in 3600 seconds

# Check it
$ curl -x 127.0.0.1:8888 https://www.google.com -v |& grep HTTP/
> CONNECT www.google.com:443 HTTP/1.1
< HTTP/1.0 200 Connection established
> GET / HTTP/1.1
< HTTP/1.1 200 OK

# Alternative 1 - use proxy function to set proxy
$ proxy 8888
$ proxy curl https://www.google.com

# Alternative 2 - or in one-liner command
$ proxy 8888 curl https://www.google.com

# Alternative 3 - manually set https_proxy and/or http_prpxy
$ export https_proxy=http://127.0.0.1:8888
$ curl https://www.google.com
```

Common usage:
- Bind to all networks and not only 127.0.0.1: `-l 0.0.0.0:8888`

### Complex configuration with configuration file

Configure:
- Create a kpx.yaml file, or use option `-c` to specify configuration file location
- Configuration can contain encrypted password, to encrypt them use `kpx -e`
- Start kpx with configuration file: `kpx [-c CONFIG]`

Configuration example:

```yaml
bind: 127.0.0.1
port: 8888
verbose: true
debug: false

proxies:
  pac-mkt:
    type: pac
    url: http://broproxycfg.int.world.company/ProxyPac/proxy.pac
    credentials: kerberos   # use native OS kerberos
  pac-ret:
    type: pac
    url: http://broproxycfg.int.world.company/ProxyPac/proxy.pac
    credentials: user
  mkt:
    type: kerberos
    spn: HTTP
    realm: EUR.MSD.WORLD.COMPANY
    host: proxy-mkt.int.world.company
    port: 8080
    credential: kerberos   # use native OS kerberos
    pac: proxy-mkt*
  ret:
    type: kerberos
    spn: HTTP
    realm: EUR.MSD.WORLD.COMPANY
    host: proxy-sgt.si.company
    port: 8080
    credential: user
    pac: proxy-sgt*
  ret-o365:
    type: kerberos
    spn: HTTP
    realm: EUR.MSD.WORLD.COMPANY
    host: proxy-sgt-o365.si.company
    port: 8080
    credential: user
  mkt-o365:
    type: kerberos
    spn: HTTP
    realm: EUR.MSD.WORLD.COMPANY
    host: proxy-mkt-o365.si.company
    port: 8080
    credential: user

credentials:
  user:
    login: x123456
    password: encrypted:sfjlqsjfljsdqklfjmsklqjfqzioeuripouzfjklsdjmflsd==

rules:
# broproxycfg must be excluded to allow downloading PAC directly - this is necessary only when working with intellij with a proxy configured 
  - host: "broproxycfg.int.world.company"
    proxy: direct
# redirect all to pac
  - host: "*"
    proxy: pac-ret
```

### Help

```yaml
$ kpx -h

# listen to this ip, use 0.0.0.0 to listen on all ips
bind: 127.0.0.1
# listen to this port
port: 7777
# set verbose to see all requests
verbose: true
# set debug to view all requests and responses headers
debug: true
# set debug to view everything
trace: false
# timeout for connecting
connectTimeout: 10
# timeout for existing connections, waiting for incoming data
idleTimeout: 0
# timeout for closing connections after one side closed it, allowing to flush the remaining buffered data
closeTimeout: 10
# check for updates, defaults to true
check: true
# automatically update, defaults to false
update: false
# exit after update, defaults to false, use only if a restart mechanism is implemented outside
restart: false

# list of proxies
proxies:
# sample of a PAC proxy. 'credentials' is used to list the users we need to get login/password on startup
  pac-mkt:
    type: pac
    url: http://broproxycfg.int.world.company/ProxyPac/proxy.pac
    credentials: user
# another PAC proxy. verbosity can also be set at proxy level. multiple credentials can also be used if resolved proxy may use different credentiels
  pac-ret:
    type: pac
    url: http://broproxycfg.int.world.company/ProxyPac/proxy.pac
    credentials: user user2
    verbose: true
# another PAC proxy. native kerberos can also be used on Windows and Linux (and MacOS?) 
  pac-krb:
    type: pac
    url: http://broproxycfg.int.world.company/ProxyPac/proxy.pac
    credentials: kerberos
# sample of kerberos proxy. 'pac' is used to get the kerberos realm in PAC proxies at runtime
  mkt:
    type: kerberos
    spn: HTTP
    realm: EUR.MSD.WORLD.COMPANY
    host: proxy-mkt.int.world.company
    port: 8080
    credential: user
    pac: proxy-mkt*
  ret:
    type: kerberos
    spn: HTTP
    realm: EUR.MSD.WORLD.COMPANY
    host: proxy-sgt.si.company
    port: 8080
    credential: user
    pac: proxy-sgt*
  krb:
    type: kerberos
    spn: HTTP
    realm: EUR.MSD.WORLD.COMPANY
    host: proxy-krb.si.company
    port: 8080
    credential: kerberos
    pac: proxy-krb*
# sample of anonymous (no authentication) proxy. 'ssl' for HTTPS proxy
  net:
    type: anonymous
    host: 127.0.0.1
    port: 3128
    ssl: false
# sample of socks proxy
  nets:
    type: socks
    host: localhost
    port: 1080
  dev:
    type: anonymous
    host: localhost
    port: 3129
  devs:
    type: socks
    host: localhost
    port: 1081
    ssl: false
# sample of basic (base64 encoding) proxy. 'credential' is the user to get login/password on startup
  basic:
    type: basic
    host: proxy-mkt.int.world.company
    port: 8080
    credential: user

# list of credentials
credentials:
# sample of credential. if no 'password', it will be asked on startup. the same for login
# password can be provided as clear text, or encrypted using '-e' option
# 'kerberos' is automatically created as native Windows/Linux/MacOS? kerberos
  user:
    login: a443939
    password: encrypted:SECRET_KEY
  user2:
    login: USER_NAME
    password: encrypted:SECRET_KEY

# list of rules to determine which proxy to use
rules:
# sample: direct connection for this host
  - host: "test-proxy-pac1"
    proxy: direct
    dns: 127.0.0.1
# sample: alter ip and or port resolution for this host. syntax is [IP]:[PORT]. no port means use the same port as source
  - host: "test-proxy-pac2"
    dns: 127.0.0.1:7777
  - host: "test-socks-conf"
    proxy: devs
    dns: 127.0.0.1
# sample: multiple hosts, separated by '|'. add '!' at the beginning to inverse rule. verbosity can also be set at rule level
  - host: 192.168.2.6|project-homo*
    proxy: devs
  - host: "*.safe.company|*.si.company|*.ressources.company|*.world.company"
    proxy: direct
    verbose: true
# sample: regex can be used for host, add 're:' at the beginning. add '!re:' to inverse rule
  - host: "re:^github\.com$|^gitlab.com$"
    proxy: mkt
    verbose: true
# sample: use mitm to have man-in-the-middle hijacked connections, CA is written in kpx.ca.crt 
  - host: "update.microsoft.com"
    proxy: mkt
    verbose: true
    mitm: true
# sample: proxy 'none' goes nowhere, result is always 400 bad request
  - host: "microsoft.com"
    proxy: none
    verbose: true
# sample: use '*' host as a catch all
  - host: "*"
    proxy: pac-mkt
    verbose: true

# list some domain aliases, allowing to use 'EUR' instead of 'EUR.MSD.WORLD.COMPANY'
domains:
  EUR: EUR.MSD.WORLD.COMPANY
  ASI: ASI.MSD.WORLD.COMPANY
  AME: AME.MSD.WORLD.COMPANY
```

### Notes

#### PAC configuration

Using PAC is a little tricky, these are a few things to know before using it:

- a PAC file can return multiple proxies
- among them, some can require authentication and some not
- to configure authentication, the returned proxy is matched against `pac:` entries in other proxies configuration  
  example: `pac: proxy-mkt*` will match any PAC returning a `PROXY proxy-mkt*`
- once a proxy is identified, it uses the authentication mechanism configured (kerberos, basic, ...)
- the `credentials:` specifies a space separated list of all the credentials that can be used by all returned proxies  
  this allows to identify used credentials, i.e. credentials that must be initialized (ask for a password or do kerberos authentication)

#### Kerberos configuration

Using kerberos authentication can be done in two ways:

- using native OS kerberos where supported (Windows/Linux/MacOS?), by using `kerberos` as `proxy.credential` or `proxy.credentials` (PAC)
- using login/password by using a `credential:` with `login` and `password` to use

#### Credentials settings

While scanning rules and proxies, all `credential:` and `credentials:` settings also mark the target credentials as being used.  
This allows to know which credentials must be initialized on startup, like asking for a missing password.

For credentials used in kerberos proxies, a login will be performed against the associated domain, to ensure password is correct.

To allows cross-domain kerberos authentication, it is possible to add domain information to the login, like this: `login: username@DOMAIN`.
