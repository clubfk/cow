# COW (Climb Over the Wall) proxy
(English translation from https://github.com/cyfdecyf/cow)

COW (Climb Over the Wall) is a simple and multifunction HTTP proxy. It can detect firewall blocked site and use second level upstream proxy to by passe that.


Current version: 0.9.1 [CHANGELOG](CHANGELOG)
[![Build Status](https://travis-ci.org/cyfdecyf/cow.png?branch=master)](https://travis-ci.org/cyfdecyf/cow)

**Contributions are welcome and please use develop branch for pull request :)**

## Features

The main goal of COW is "automatic". It's transparent to users whether a site accessible directly by cow or blocked (which will be escaped then via upstream proxies).

- HTTP proxy for mobile devices
- Support HTTP, SOCKS5, [shadowsocks](https://github.com/clowwindy/shadowsocks/wiki/Shadowsocks-%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E) as second level proxies (including COW it-self)
  - can load-balance between multiple second level proxies
- automatic detection for blocked sites，and escape to upstream proxies
- automatic PAC generation
  - Built-in [no-proxy](site_direct.go) list，for banking, media streaming sites（can be configured manually）

# Quickstart

Installation：

- **OS X, Linux (x86, ARM):** run as follwoing（apply to update also）

        curl -L git.io/cow | bash

- **Windows:** [download here](http://dl.chenyufei.info/cow/)
- If you have Go compiler, you can use also `go get github.com/cyfdecyf/cow` to build from source

Simple configuration: 

Edit `~/.cow/rc` (Linux) or `rc.txt` (Windows)，as the following sample:

    #This is a comment
    # local http proxy address to be used in browser for HTTP/HTTPS proxy
    # or you can use the automatic proxy configuration URL http://127.0.0.1:7777/pac
    listen = http://127.0.0.1:7777

    # SOCKS5 upstream proxy
    proxy = socks5://127.0.0.1:1080

    # HTTP upstream proxy
    proxy = http://127.0.0.1:8080
    proxy = http://user:password@127.0.0.1:8080

    # shadowsocks upstream proxy
    proxy = ss://aes-128-cfb:password@1.2.3.4:8388

    # cow upstream proxy
    proxy = cow://aes-128-cfb:password@1.2.3.4:8388
    # on the upstream proxy, use the following:
    #listen = cow://aes-128-cfb:password@0.0.0.0:8388

Then simply run cow and enjoy it ;)

# Configuration details

The configuration file is readed from `~/.cow/rc` on Unix systems or `rc.txt` within the cow binary on Windows systems. **[The samples directory](doc/sample-config/rc) content all possible configuraiton options with detailed instructions**, just download, change and use them.

Start COW：

- Unix: from any shell run `cow &`
  - [Linux init](doc/init.d/cow) for Debian based systems. Other distros may varie.
- Windows
  - double-click `cow-taskbar.exe`, or
  - double-click `cow-hide.exe` for an invisible launch
  - both will launch `cow.exe`

##PAC (Proxy Auto Config)
Use `http://<listen address>/pac` in your browser.

PAC is usefull as it can advise the browser to bypass proxy for some URLs (thus better performance). But if one site got blocked afterward, the browser will not fall back to cow. For this reason, **it's better NOT to use PAC but always go thru cow.**

Some command line options can override configuration file values, run `cow -h` for more details。

## Configure manually blocked sites and direct sites

**In noraml situation you don't need to do that, but for some special case:**

`~/.cow/blocked` and `~/.cow/direct` many define blocked and direct sites（`direct` host will be part of PAC）：

- One line per site or URL (cow will check for site prior to check URL)
  - second level domain like `google.com` equals to `*.google.com`
  - sub-domain under some generic second level domain (like `google.com.hk` for `com.hk`, `edu.cn`) equals to all sub-sub-dmains like `*.google.com.hk`
  - other domains will be exact match like `plus.google.com`

Caution: All private ip (RFC 1918) **and** non-fqdn names will be considered as `diret site` and will not use cow.

# Technical details

## Proxy stat

COW store direct/blocked json formatted statistics in `~/.cow/stat`.

- **if a site is unseen, direct access will be tried first, if failed then use upstream proxy，Restart direct check 2 minutes later**
  - Internal[kkown blocked sites](site_blocked.go) to reduce check load（may be ajustable manually）
- 直连访问成功一定次数后相应的 host 会添加到 PAC
- host 被墙一定次数后会直接用二级代理访问
  - 为避免误判，会以一定概率再次尝试直连访问
- host 若一段时间没有访问会自动被删除（避免 `stat` 文件无限增长）
- 内置网站列表和用户指定的网站不会出现在统计文件中

## How COW detect the Great FireWall

COW 将以下错误认为是墙在作怪：

- 服务器连接被重置 (connection reset)
- 创建连接超时
- 服务器读操作超时

无论是普通的 HTTP GET 等请求还是 CONNECT 请求，失败后 COW 都会自动重试请求。（如果已经有内容发送回 client 则不会重试而是直接断开连接。）

用连接被重置来判断被墙通常来说比较可靠，超时则不可靠。COW 每隔半分钟会尝试估算合适的超时间隔，避免在网络连接差的情况下把直连网站由于超时也当成被墙。
COW 默认配置下检测到被墙后，过两分钟再次尝试直连也是为了避免误判。

如果超时自动重试给你造成了问题，请参考[样例配置](doc/sample-config/rc)高级选项中的 `readTimeout`, `dialTimeout` 选项。

## Caveats

- No cache
- No HTTP pipeline support（Chrome, Firefox do not use pipeline by default，there's no evidence to support it）

# Thanks

Sources：

- @tevino: http parent proxy basic authentication
- @xupefei: for cow-hide.exe and windows service for cow.exe
- @sunteya: startup and installation

Bug reporter:

- GitHub users: glacjay, trawor, Blaskyy, lucifer9, zellux, xream, hieixu, fantasticfears, perrywky, JayXon, graminc, WingGao, polong, dallascao
- Twitter users: special thanks to @shao222 for test and bug report, @xixitalk

@glacjay 对 0.3 版本的 COW 提出了让它更加自动化的建议，使我重新考虑 COW 的设计目标并且改进成 0.5 版本之后的工作方式。
