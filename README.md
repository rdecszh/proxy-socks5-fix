# python requests socks5: Fix the InvalidSchema Error, Get Auth and DNS Right, and Pick a Proxy Plan That Won't Blow the Budget

You wrote four lines of Python, pointed `requests` at a SOCKS5 endpoint, and got back:


requests.exceptions.InvalidSchema: Missing dependencies for SOCKS support.


Nothing about your code is wrong. `requests` just doesn't speak SOCKS on its own, and the error it throws when you try is famously unhelpful about what to do next. Everything below is the short version of getting past that, plus the part most tutorials skip entirely: which proxy plan to actually buy, and how its billing model changes the way you write the script.

## Why requests refuses to talk SOCKS

The `requests` library handles HTTP and HTTPS proxies natively. For SOCKS it delegates to PySocks, and PySocks ships as an optional extra rather than a default dependency — which is why the failure arrives as a schema error instead of an import error.

Fix it with the extras syntax, not a bare `pip install socks`:

bash
pip install "requests[socks]"


If you're inside a virtualenv and want to confirm it landed in the right interpreter:

bash
python -c "import socks; print(socks.__version__)"


A `ModuleNotFoundError` after installing usually means you installed into a different Python than the one running your script — check `which python` / `which pip` before you start debugging anything more exotic.

## The shortest thing that works

Once PySocks is present, the SOCKS5 endpoint goes in the same `proxies` dict you'd use for an HTTP proxy. Two keys, not one:

python
import requests

proxy_url = "socks5://USERNAME:PASSWORD@HOST:PORT"
proxies = {
    "http": proxy_url,
    "https": proxy_url,
}

r = requests.get("https://ipinfo.io/json", proxies=proxies, timeout=15)
print(r.json())


Set the `timeout`. Without it, a dead residential IP will hang your script until the OS gives up on the TCP connection, and you'll spend ten minutes wondering why the program froze.

Filling in only the `https` key is the most common reason people report "the proxy doesn't work" — they send a plain-HTTP request somewhere and it goes out over their real IP.

## socks5:// vs socks5h:// — the DNS question

This is the detail that separates a working scraper from one that leaks your local DNS resolver on every request.

| Scheme | Who resolves the hostname | What it means in practice |
| --- | --- | --- |
| `socks5://` | Your machine | Your local resolver sees every domain you request |
| `socks5h://` | The proxy | The hostname travels to the proxy and gets resolved there |

For scraping, geo-checking, or anything where the target domain shouldn't be visible to your ISP, use `socks5h://`. It also solves a practical problem: if your local DNS blocks or misresolves a domain that the proxy's network resolves fine, `socks5://` fails while `socks5h://` succeeds.

The tradeoff is real but small — remote resolution adds a round trip, so benchmarks usually come out a few milliseconds slower. That's a fair price for not leaking your resolver.

python
proxy_url = "socks5h://USERNAME:PASSWORD@HOST:PORT"


The Chrome team learned this the hard way: Chromium only accepts the `socks5://` scheme and rejects `socks5h://` outright, so scripts that set `ALL_PROXY=socks5h://...` globally tend to break any browser automation in the same environment. If you're mixing requests and a headless browser in one container, keep the schemes straight.

## Credentials with `@`, `:` or other punctuation

Proxy usernames and passwords that contain reserved URL characters will silently break parsing. Percent-encode them:

python
from urllib.parse import quote

user = quote("user@example", safe="")
pwd  = quote("p@ss:word", safe="")
proxy_url = f"socks5h://{user}:{pwd}@HOST:PORT"


Then keep the credentials out of the source file entirely:

python
import os

proxy_url = os.getenv("PROXY_URL")
proxies = {"http": proxy_url, "https": proxy_url} if proxy_url else None


## Sessions and environment variables

If every call in a script uses the same proxy, attach it to a `Session` instead of passing `proxies=` into each `get()`. You get connection reuse and one place to change the config:

python
with requests.Session() as s:
    s.proxies.update({"http": proxy_url, "https": proxy_url})
    s.headers.update({"User-Agent": "Mozilla/5.0 (...)"})
    for url in urls:
        print(s.get(url, timeout=15).status_code)


Session reuse does **not** guarantee a stable exit IP. Whether the IP changes depends on the proxy provider's rotation rules, not on your session object.

`requests` also reads `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY` and their lowercase variants from the environment automatically — including `ALL_PROXY=socks5h://...`. If you want a script to ignore ambient proxy settings entirely, pass `trust_env=False`. That one flag explains a lot of "works on my laptop, fails in CI" bugs.

## How the plan you buy changes the code

Here's where most `python requests socks5` tutorials stop, and where the actual friction lives. Your `proxies` dict has exactly two shapes, and which one you need is decided by the provider's billing model.

9Proxy runs residential proxies across 20M+ IPs in 90+ countries, and supports HTTP, HTTPS and SOCKS5, so it drops straight into the dict above. It splits its product into two models with genuinely different setup paths — and one of them requires a small change to how you think about the endpoint address.

**Residential by GB** is the one that works directly from the dashboard. You get a fixed hostname and port, plus a sub-user password, and all the targeting lives inside a structured username:


<subaccount>-country-<country_code>-st-<state>-city-<city>-isp-<isp_code>-sst-<minutes>-ssid-<session_id>


So rotating mode looks like this — no `sst`, no `ssid`, a fresh IP per request:

python
user = "myacct-country-us"
pwd  = "YOUR_SUBUSER_PASSWORD"
proxy_url = f"socks5h://{user}:{pwd}@HOST:PORT"


And sticky mode adds a session timer, which is what you want for logins, carts or anything that breaks when the IP changes mid-flow:

python
user = "myacct-country-us-sst-15-ssid-bot01"


`sst-15` holds that IP for 15 minutes. `ssid-bot01` gives you a named session — run four threads with `ssid-bot01` through `ssid-bot04` and each gets its own sticky IP while sharing the same country and city config. If you can't remember the format, 9Proxy's own docs show the same username layout used across Proxifier, FoxyProxy and Hidemyacc, and it's identical in a `requests` dict.

**Residential by IPs** works differently and catches people out. It bills per IP with unlimited bandwidth on each, and the unused IPs don't expire — but authentication runs through the 9Proxy desktop app, which forwards a chosen residential IP to a local port. Your `requests` call then points at localhost:

bash
# forward a US proxy to local port 60000
9proxy proxy -c US -p 60000


python
proxies = {
    "http":  "socks5://127.0.0.1:60000",
    "https": "socks5://127.0.0.1:60000",
}


That's the whole trick: `127.0.0.1:<port>` instead of a remote host. The CLI version above is documented for Linux, and it's a good fit for a headless box where you'd rather not click through a GUI. Forward a second proxy to `60001` and you have a two-entry rotation pool without touching the provider's API.

👉 [Set up a 9Proxy account and generate your first SOCKS5 endpoint](https://bit.ly/9-Proxy)

## All current 9Proxy packages

Pricing below reflects the adjustment that took effect 1 June 2026 (00:00 UTC), which raised IP-based and bundle prices while leaving GB-based packages untouched. If you only need bandwidth, the sticker price didn't move.

| Plan | What you get | Price | Billing / validity | Purchase |
| --- | --- | --- | --- | --- |
| **IP-based** — 100 IPs | 100 residential IPs, unlimited bandwidth | $24 ($0.24/IP) | Pay per IP, IPs never expire | [Get 100 IPs](https://bit.ly/9-Proxy) |
| **IP-based** — 500 IPs | 500 residential IPs, unlimited bandwidth | $72 ($0.144/IP) | Pay per IP, IPs never expire | [Get 500 IPs](https://bit.ly/9-Proxy) |
| **IP-based** — 1,000 + 500 IPs | 1,500 IPs total (500 bonus), unlimited bandwidth | $126 ($0.084/IP) | Pay per IP, IPs never expire | [Get 1,500 IPs](https://bit.ly/9-Proxy) |
| **IP-based** — 2,500 IPs | 2,500 residential IPs, unlimited bandwidth | $210 ($0.084/IP) | Pay per IP, IPs never expire | [Get 2,500 IPs](https://bit.ly/9-Proxy) |
| **IP-based** — 5,000 IPs | 5,000 residential IPs, unlimited bandwidth | $360 ($0.072/IP) | Pay per IP, IPs never expire | [Get 5,000 IPs](https://bit.ly/9-Proxy) |
| **IP-based** — 15,000 IPs | 15,000 residential IPs, unlimited bandwidth | $720 ($0.048/IP) | Pay per IP, IPs never expire | [Get 15,000 IPs](https://bit.ly/9-Proxy) |
| **IP-based** — 25,000 IPs | 25,000 residential IPs, unlimited bandwidth | $863 ($0.035/IP) | Pay per IP, IPs never expire | [Get 25,000 IPs](https://bit.ly/9-Proxy) |
| **IP-based** — 50,000 IPs | 50,000 residential IPs, unlimited bandwidth | $1,438 ($0.029/IP) | Pay per IP, IPs never expire | [Get 50,000 IPs](https://bit.ly/9-Proxy) |
| **Business IP** — 100,000 IPs | 100,000 residential IPs | $2,300 ($0.023/IP) | Pay per IP, IPs never expire | [Get 100,000 IPs](https://bit.ly/9-Proxy) |
| **Business IP** — 200,000 IPs | 200,000 residential IPs | $4,140 ($0.021/IP) | Pay per IP, IPs never expire | [Get 200,000 IPs](https://bit.ly/9-Proxy) |
| **Business IP** — 500,000 IPs | 500,000 residential IPs | $8,625 ($0.018/IP) | Pay per IP, IPs never expire | [Get 500,000 IPs](https://bit.ly/9-Proxy) |
| **GB-based** — 5 GB | Rotating or sticky IPs, unlimited endpoints | $15 ($3.00/GB) | 180-day validity | [Get 5 GB](https://bit.ly/9-Proxy) |
| **GB-based** — 50 + 5 GB | Rotating or sticky IPs, unlimited endpoints | $105 ($2.10/GB) | 180-day validity | [Get 55 GB](https://bit.ly/9-Proxy) |
| **GB-based** — 100 GB | Rotating or sticky IPs, unlimited endpoints | $150 ($1.50/GB) | 180-day validity | [Get 100 GB](https://bit.ly/9-Proxy) |
| **GB-based** — 200 GB | Rotating or sticky IPs, unlimited endpoints | $200 ($1.00/GB) | 180-day validity | [Get 200 GB](https://bit.ly/9-Proxy) |
| **GB-based** — 1,000 GB | Rotating or sticky IPs, unlimited endpoints | $800 ($0.80/GB) | 180-day validity | [Get 1,000 GB](https://bit.ly/9-Proxy) |
| **GB-based** — 2,000 GB | Rotating or sticky IPs, unlimited endpoints | $1,500 ($0.75/GB) | 180-day validity | [Get 2,000 GB](https://bit.ly/9-Proxy) |
| **Enterprise GB** — 3,000 GB | Rotating or sticky IPs, unlimited endpoints | $2,160 ($0.72/GB) | Unlimited validity | [Get 3,000 GB](https://bit.ly/9-Proxy) |
| **Enterprise GB** — 6,000 GB | Rotating or sticky IPs, unlimited endpoints | $4,200 ($0.70/GB) | Unlimited validity | [Get 6,000 GB](https://bit.ly/9-Proxy) |
| **Enterprise GB** — 10,000 GB | Rotating or sticky IPs, unlimited endpoints | $6,800 ($0.68/GB) | Unlimited validity | [Get 10,000 GB](https://bit.ly/9-Proxy) |
| **Bundle Starter** | 100 IPs + 5 GB | $30 | Balance-based, 180-day traffic validity | [Get the Starter bundle](https://bit.ly/9-Proxy) |
| **Bundle Popular** | 1,500 IPs + 50 GB | $180 | Balance-based, 180-day traffic validity | [Get the Popular bundle](https://bit.ly/9-Proxy) |
| **Bundle Pro** | 5,000 IPs + 500 GB | $720 | Balance-based, 180-day traffic validity | [Get the Pro bundle](https://bit.ly/9-Proxy) |

All prices in USD.

## IP-based or GB-based for a requests script?

The decision comes down to what your script does per IP, not how much data it moves.

**Pick GB-based if** your loop rotates IPs constantly — a new endpoint per request, a few hundred kilobytes each. You're billed on traffic, endpoints are unlimited, and the structured username above gives you country, city, state and ISP targeting without any client-side bookkeeping. At $1.00/GB on the 200 GB tier, a scraper pulling 15 GB a month costs about $15 of balance spread across three 180-day windows. Over-filtering the username (state *and* city *and* ISP together) shrinks the available pool and slows things down, so add parameters only when you need them.

**Pick IP-based if** you need the same exit IP across many requests and heavy transfer. Bandwidth is unlimited, so pulling 40 GB through one IP costs the same as pulling 40 MB. That's also the model where unused IPs never expire — buy 5,000 now, spend them over a year if you want. The catch is the desktop app requirement: your `requests` code talks to `127.0.0.1:<forwarded port>`, so any deployment needs the app running alongside it. Fine on a VPS you control, awkward in serverless.

For a personal account-farming script holding 20 sessions, the math favours IP-based. For a price-monitoring job hitting 200,000 product pages, it favours GB-based.

👉 [Compare 9Proxy's current packages before you commit to a billing model](https://bit.ly/9-Proxy)

## Confirm the proxy is actually being used

Don't trust a 200 response. Check the exit IP:

python
r = requests.get("https://ipinfo.io/json", proxies=proxies, timeout=15)
data = r.json()
print(data["ip"], data.get("city"), data.get("country"))


Run it twice against a rotating GB-based endpoint and the IP should change. Against a sticky session it shouldn't, until `sst` expires. If the IP comes back as your own every time, the `proxies` dict isn't reaching the request — check for a `Session` overriding it, or `trust_env` interactions.

For a second opinion, `https://httpbin.org/ip` is fine but returns less metadata. `ipinfo.io/json` gives you country and city, which is how you confirm that `country-us` in the username actually did something.

## Troubleshooting the failures you'll actually hit

| Symptom | Cause | Fix |
| --- | --- | --- |
| `InvalidSchema: Missing dependencies for SOCKS support` | PySocks not installed in the running interpreter | `pip install "requests[socks]"` |
| `InvalidURL` on a working credential | `@`, `:` or `/` in the username or password | Percent-encode with `urllib.parse.quote` |
| HTTP requests leak your real IP | Only the `https` key set in `proxies` | Set both `http` and `https` |
| Works with curl, times out in requests | curl lets the proxy resolve DNS; requests may resolve locally | Switch to `socks5h://` |
| `ReadTimeout` on every request | `timeout` too short for a residential hop, or a dead IP | Raise to 10–15s; split connect/read as `timeout=(5, 15)` |
| Credentials rejected | Target site IP-mismatched your authenticated session | Keep the same `ssid` across the login and the follow-up request |
| `SSLError` behind a corporate MITM proxy | TLS interception, not a SOCKS problem | Point `verify` at the corporate CA bundle |

That "works with curl, fails in requests" row deserves a note. curl defaults to letting the proxy resolve hostnames; the `requests` stack can end up resolving locally and handing an IP to the proxy. If the target does any IP-based routing or DNS filtering, the two tools behave completely differently against the same endpoint. `socks5h://` removes the ambiguity.

## If you're scraping at scale

Synchronous `requests` is fine up to a few hundred concurrent workers, but past that the GIL and per-request overhead start to bite. `aiohttp` plus `aiohttp_socks` is the usual migration:

python
import aiohttp
from aiohttp_socks import ProxyConnector

connector = ProxyConnector.from_url("socks5h://USER:PASS@HOST:PORT")
async with aiohttp.ClientSession(connector=connector) as session:
    async with session.get("https://ipinfo.io/json") as resp:
        print(await resp.json())


Same SOCKS5 endpoint, same structured username, same `socks5h` caveat. The connector is per-session, and each sticky `ssid` still gives you a distinct IP.

## The short version

Install the SOCKS extra before you write any proxy code, set both keys in the dict, use `socks5h://` unless you have a reason not to, and always set a timeout. Then pick a billing model that matches your traffic shape — per-IP with unlimited bandwidth when sessions are long and data is heavy, per-GB when you rotate constantly and requests are small.

The error message that started this whole search is, at least, honest about the problem. It's just missing the second half — the part where you decide which proxy to buy.
