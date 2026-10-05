# python requests proxy: Working code, auth, rotation, SOCKS5, and the per-GB math before you scale

Getting a proxy into `requests` is a four-line change. That part is not why your scraper falls over at request 300.

The part that bites is everything around those four lines: a password with an `@` in it that silently breaks the URL, a `socks5://` string that quietly resolves DNS on your own machine, a retry loop that keeps hammering the same dead IP, and a bandwidth budget that turns out to be double what you planned because city-level targeting bills at a different rate. None of that shows up in a ten-line tutorial, and all of it shows up in production.

This walks through the setup, then the stuff that actually breaks, then what the traffic costs per GB if you're running this at any real volume.

## The minimum viable setup

`requests` takes a dictionary with `http` and `https` keys. Both keys point at the same proxy endpoint unless you deliberately split traffic across two.

python
import requests

proxies = {
    "http":  "http://USERNAME:PASSWORD@gw.dataimpulse.com:823",
    "https": "http://USERNAME:PASSWORD@gw.dataimpulse.com:823",
}

r = requests.get("https://httpbin.org/ip", proxies=proxies, timeout=30)
print(r.json())


That host and port pair is DataImpulse's rotating gateway (port 823 for HTTP/HTTPS, 824 for SOCKS5), and it's the endpoint you'd use once you have credentials in hand. Two details in those six lines matter more than they look:

**Always set `timeout`.** Without it, a proxy that accepts the TCP connection and then goes silent will hang your process indefinitely. A tuple like `timeout=(5, 30)` splits connect and read timeouts, which makes debugging faster.

**Always include the scheme.** `"10.10.1.10:3128"` without `http://` raises `requests.exceptions.MissingSchema` on requests 2.x. This changed years ago and still catches people copying old Stack Overflow answers.

## Confirm the traffic is actually leaving through the proxy

`httpbin.org/ip` (or `httpbin.io/ip`) returns the address the server saw. If it's yours, the proxy isn't in play — you configured something that requests ignored, or an environment variable is overriding the argument.

Hit it after every config change, including the boring ones. If you want the full picture, `httpbin.org/headers` shows the request headers as received, and `httpbin.org/get` shows the `Via` and `Proxy-Connection` headers when a proxy is in the path.

## Authentication: the part that breaks quietly

Most paid proxies use username and password embedded in the URL rather than a separate auth step. That's convenient and fragile, for two reasons.

If your password contains `@`, `:`, `#`, or `/`, the URL parser will misread it. Encode it first:

python
from urllib.parse import quote

user = "your_login"
pw   = quote("p@ss:word", safe="")

proxy = f"http://{user}:{pw}@gw.dataimpulse.com:823"


The second issue is credentials sitting in source control. Pull them from the environment instead:

python
import os

user = os.environ["PROXY_USER"]
pw   = os.environ["PROXY_PASS"]
host = os.environ.get("PROXY_HOST", "gw.dataimpulse.com")
port = os.environ.get("PROXY_PORT", "823")

proxy_url = f"http://{user}:{pw}@{host}:{port}"
proxies = {"http": proxy_url, "https": proxy_url}


`requests` also honors `HTTP_PROXY` and `HTTPS_PROXY` environment variables on its own, which is genuinely useful for scripts you don't control. You can opt out per-call by passing `proxies={}`.

The alternative to credentials is IP whitelisting — you register your server's egress IP in the dashboard and drop auth from the URL entirely. That's less error-prone in CI, but it breaks the moment your host changes IPs, and it makes rotating across machines harder.

## Set it once with a Session

A bare `requests.get()` opens a fresh connection every time. A `Session` pools connections and keeps the proxy config in one place:

python
session = requests.Session()
session.proxies = proxies
session.headers.update({"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36"})

for url in urls:
    r = session.get(url, timeout=30)


Sessions also carry cookies across requests, which matters if the target site sets a session token on the first hit and expects it back. For a few dozen requests the difference is negligible. For a hundred thousand, connection reuse is the difference between a job that finishes overnight and one that doesn't.

## SOCKS5, and the DNS leak nobody mentions

`requests` doesn't speak SOCKS out of the box:

bash
pip install "requests[socks]"


Then use `socks5h://`, not `socks5://`:

python
proxies = {
    "http":  "socks5h://USERNAME:PASSWORD@gw.dataimpulse.com:824",
    "https": "socks5h://USERNAME:PASSWORD@gw.dataimpulse.com:824",
}


The `h` means hostname resolution happens on the proxy side. With plain `socks5://`, your machine resolves the domain first and then sends the connection — which means your local DNS resolver sees every domain you're scraping, even though the HTTP traffic goes through the proxy. If the point of the proxy is to look like you're somewhere else, `socks5://` undermines half of it. DataImpulse exposes SOCKS5 on port 824, separate from the HTTP/HTTPS gateway on 823.

## Rotation: you probably don't need a proxy list

There are two models here, and they lead to very different code.

**Gateway rotation.** Every request you send to the single gateway endpoint egresses from a different IP. There's no list to manage, no health checks, no `random.choice()`. This is what the setup above does:

python
for _ in range(5):
    print(requests.get("https://httpbin.org/ip", proxies=proxies, timeout=30).json())


Five calls, five different addresses.

**Self-managed lists.** You fetch an endpoint that returns a block of IPs, then rotate them yourself with `itertools.cycle` or round-robin. This is what most tutorials teach, because it's what free proxy sources require. It also means writing a validator, tracking dead entries, and retrying when an IP dies mid-job. If you're scraping free proxy lists to save money, the developer hours cost more than the traffic.

**Sticky sessions** are the middle ground. Sometimes you need the same IP across several requests — a login flow, a paginated sequence, a multi-step checkout. DataImpulse's sticky windows are configurable from 1 to 120 minutes. The exact session string goes in the username, and you should copy the format from the dashboard rather than assemble it by hand; guessing the token format is how people end up with silent rotation they didn't want.

**Geo-targeting** works the same way — a country suffix on the username rather than a different endpoint. Country selection is included in the rate. City, ZIP, and ASN targeting is a paid add-on, and on standard residential traffic it routes at double the base per-GB rate. That's the single most expensive detail in this article: if you build a city-level crawler at $1/GB and don't read the fine print, your effective rate is $2/GB, and your 5 GB budget buys 2.5 GB of work.

## Retries, timeouts, and the codes you'll actually see

Proxies fail in ways that plain HTTP doesn't. Common ones worth handling explicitly:

- **407** — proxy authentication required. Wrong credentials, or the password wasn't URL-encoded.
- **403 / 429** — the target blocked you. Usually IP quality, request rate, or missing browser headers, not a config error.
- **`requests.exceptions.ProxyError`** — the proxy endpoint itself is unreachable or refusing connections.
- **`requests.exceptions.ConnectTimeout` / `ReadTimeout`** — the proxy took too long.

Automatic retries with backoff are a few lines:

python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(
    total=3,
    backoff_factor=0.8,
    status_forcelist=[429, 500, 502, 503, 504],
    allowed_methods=["GET", "POST"],
)

session = requests.Session()
session.proxies = proxies
session.mount("http://",  HTTPAdapter(max_retries=retry, pool_connections=20, pool_maxsize=20))
session.mount("https://", HTTPAdapter(max_retries=retry, pool_connections=20, pool_maxsize=20))


One caveat: the retry adapter retries on status codes, not on a fresh IP. With a rotating gateway that's fine — a retry gets a new address anyway. With a static list, you need your own loop that pops the next proxy on failure, plus a cap so a permanently bad list doesn't run forever.

Wrap the whole thing in a try/except per URL, not per job. One dead target shouldn't kill a 50,000-request run.

## What the traffic actually costs

Proxies bill per gigabyte of traffic, not per request. That means your cost depends entirely on how big the responses are, and HTTP responses are more predictable than people assume.

Plain HTML pages run tens to low hundreds of kilobytes. At roughly 200 KB per page:

| Response size | Requests per GB | Cost at $1/GB | Cost at $0.50/GB |
| --- | --- | --- | --- |
| 50 KB | ~20,000 | $0.00005 | $0.000025 |
| 100 KB | ~10,000 | $0.0001 | $0.00005 |
| 200 KB | ~5,000 | $0.0002 | $0.0001 |
| 1 MB | ~1,000 | $0.001 | $0.0005 |

Those are arithmetic from the published rates, not measured benchmarks — but the shape of it is what matters. Scraping HTML is cheap. Scraping images, PDFs, or API responses that return megabytes of JSON is not, and that's where a "cheap" $1/GB suddenly matters.

DataImpulse's model is pay-as-you-go with no subscription. You buy GB, they don't expire, and you draw down as you use them. That's a meaningfully different proposition from a monthly plan you either over- or under-use, especially during the phase where you're testing a target and don't yet know your per-page bandwidth cost.

👉 [Compare DataImpulse proxy types and current per-GB rates](https://bit.ly/dataimPulse)

## The full plan line-up

DataImpulse sells four proxy types. Within each, you choose the GB amount in the dashboard and the price recalculates live — the numbers below are the published tiers.

| Proxy type | Package | Price | Effective rate | Best for | Get it |
| --- | --- | --- | --- | --- | --- |
| Residential | 5 GB | $5 | $1.00/GB | Testing targets behind basic anti-bot | [Start with 5 GB residential](https://bit.ly/dataimPulse) |
| Residential | 1 TB | $800 | $0.80/GB | Volume scraping on protected targets | [Buy 1 TB residential](https://bit.ly/dataimPulse) |
| Datacenter | 10 GB | $5 | $0.50/GB | Unprotected targets, speed over stealth | [Get 10 GB datacenter](https://bit.ly/dataimPulse) |
| Datacenter | 100 GB | $50 | $0.50/GB | Large fast crawls, public data | [Buy 100 GB datacenter](https://bit.ly/dataimPulse) |
| Datacenter | 1 TB | $450 | $0.45/GB | Sustained high-volume jobs | [Get 1 TB datacenter](https://bit.ly/dataimPulse) |
| Mobile | 2.5 GB | $5 | $2.00/GB | Mobile-only endpoints, app data | [Try 2.5 GB mobile](https://bit.ly/dataimPulse) |
| Mobile | 25 GB | $50 | $2.00/GB | Mobile testing at moderate scale | [Buy 25 GB mobile](https://bit.ly/dataimPulse) |
| Mobile | 1 TB | $1,600 | $1.60/GB | Hard targets needing 4G/5G IPs | [Get 1 TB mobile](https://bit.ly/dataimPulse) |
| Premium Residential | 1 GB | $5 | $5.00/GB | High-speed, high-trust residential | [Try 1 GB premium residential](https://bit.ly/dataimPulse) |
| Premium Residential | 10 GB | $50 | $5.00/GB | All targeting included, dedicated manager | [Buy 10 GB premium residential](https://bit.ly/dataimPulse) |

Above 5 TB, pricing for every type is custom-quoted. The premium residential tier includes all targeting options without the surcharge that applies to standard residential traffic — which narrows the gap if your workload is city-specific.

A few practical notes on the plans:

- Minimum purchase is $5. There's no free tier and no no-payment trial.
- Intro plans carry a 7-day money-back window, but it's conditional: card payments only, and it requires that you've used less than 80% of the traffic. Crypto purchases on intro plans don't qualify.
- Locally configured amounts are flexible. The tiers are the published package points, not the only option.
- Residential country targeting is free; city/ZIP/ASN is the surcharged one.

## Which one fits your scraper

Three workloads, three answers.

**A script that gets blocked by Cloudflare or a WAF.** Datacenter IPs get flagged fast on protected targets. Residential at $1/GB is the right starting point: buy the 5 GB intro tier, point the gateway at your target, and measure how many successful responses you get per GB. That number — cost per successful request — is the only metric worth comparing across providers.

**A bulk pull of public, unprotected pages.** News archives, public directories, documentation. Datacenter at $0.50/GB is half the cost and faster. There's no stealth requirement to pay for.

**A flow that needs the same IP for several minutes.** A login sequence, a multi-step form, anything stateful. Configure a sticky session window, keep the session ID consistent across calls, and set the TTL only as long as the flow needs. Over-long sticky windows reduce pool rotation and increase your exposure to any single IP's block.

Whatever you pick, the first run should be small and instrumented: log status codes, log bytes consumed, log the egress IP every N requests. Five GB at $1/GB is five thousand requests at 200 KB each — enough to find out whether your target actually cooperates.

👉 [Set up a DataImpulse account and grab your gateway credentials](https://bit.ly/dataimPulse)

## Quick answers to the errors you'll hit

**`MissingSchema: Invalid URL 'ip:port'`** — you left off `http://` in the proxy string.

**`ProxyError: Cannot connect to proxy`** — the gateway is unreachable. Check host, port, and whether your network blocks outbound on that port. Port 823 for HTTP/HTTPS, 824 for SOCKS5.

**`InvalidURL` or a mangled hostname** — a special character in your password. Run it through `urllib.parse.quote()`.

**Everything returns 200 but the IP is still yours** — an `HTTP_PROXY` or `HTTPS_PROXY` environment variable is overriding, or you passed the dictionary to the wrong call. Check with `httpbin.org/ip` before blaming the provider.

**403s that start around the same request count every run** — you're tripping a rate limit, not being blocked on IP. Add jitter between requests, cut concurrency, and check whether your headers look like a browser.

**`ModuleNotFoundError: No module named 'socks'`** — you wrote a `socks5h://` URL without installing `requests[socks]`.

The setup really is short. It's the retries, the encoding, the DNS resolution choice, and the targeting surcharge that decide whether the thing runs for a week or dies on a Tuesday afternoon.
