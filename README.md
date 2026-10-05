# https proxy: what it is, how to set one up in Chrome, curl and Python, and when paying beats scraping free lists

Search "https proxy" and you get two very different audiences jammed into one results page. Half the people want to know what the term means and how to point a script at one. The other half already know, and they're looking for a proxy that works — which is where the free-list rabbit hole starts and most of an afternoon disappears.

This covers both. First the mechanics, because the word "https proxy" gets used for two different things and that mix-up causes real configuration mistakes. Then the practical setup for the tools most people actually use, and finally what a paid provider costs when the free route stops being worth the time.

## Two things "https proxy" can mean

The ambiguity is worth five minutes, because it changes what you configure.

**Meaning one: a proxy you use to reach HTTPS sites.** Almost every HTTP proxy falls into this category. You hand it a request for `https://example.com`, it opens a TCP tunnel to the site via the `CONNECT` method, and your TLS handshake with the site happens inside that tunnel. The proxy is just a courier. This is what people mean 90% of the time.

**Meaning two: a secure web proxy, where the connection from your client to the proxy itself is TLS.** Chromium's documentation describes this as a web proxy the browser talks to over SSL rather than clear text — useful on hostile networks like airport or café Wi-Fi, where plain HTTP to a proxy leaks cookies and session data. Here you'd configure something like `--proxy-server=https://proxy.example.com:443`, or a PAC file returning `HTTPS secure-proxy.example.com:443`.

The practical difference: in meaning one, a "https proxy" is a normal proxy doing its job. In meaning two, the `https://` in front of the proxy address is doing real work — encrypting the hop you control, not just the hop the site controls.

A note on browser support, again from Chromium's own docs: not every browser handles the `HTTPS` proxy type inside a PAC file. System-wide PAC files relying on it can behave inconsistently outside Chrome. If you're on a mixed browser setup, the `--proxy-server=https://` flag or a per-app setting is the safer route.

## What the proxy can and can't see

With CONNECT tunneling, the proxy knows the hostname and port you asked for, plus the volume and timing of the traffic. It does not see your URLs, form data or response bodies, because TLS is negotiated end to end between your client and the target server.

That's the honest version of "HTTPS keeps it private." It keeps the payload private. It doesn't hide where you're going from the proxy operator.

And it only holds if you trust the operator not to install an interception certificate. Free proxy lists have a long, well-documented history of exactly that — a proxy that hands you its own root certificate can read everything, because you agreed to a man-in-the-middle by design. Which is the real argument against free proxies, not just the uptime.

## Setting one up: environment variables, curl, Chrome, Python

The syntax is predictable enough that you can copy-paste your way through it.

**Environment variables.** `https_proxy` (and `http_proxy`) are the standard. GNU Wget's manual documents both, along with `no_proxy` for a comma-separated bypass list. The same variables are honored by curl, most package managers and plenty of CLI tools, which is why exporting them is still the fastest way to shove a whole shell session through a proxy.

bash
export https_proxy="http://USERNAME:PASSWORD@gw.dataimpulse.com:823"
export http_proxy="$https_proxy"
export no_proxy="localhost,127.0.0.1"


**curl.** Pass `-x` (or `--proxy`). The `https://` proxy scheme was added in curl 7.52.0 for the OpenSSL, GnuTLS and NSS builds, and an unrecognized scheme raises an error rather than silently falling back.

bash
curl -x http://USERNAME:PASSWORD@gw.dataimpulse.com:823 https://httpbin.org/ip


**Chrome.** Either a PAC file, or the launch flag:

bash
chrome --proxy-server=http://USERNAME:PASSWORD@gw.dataimpulse.com:823


**Python.** Requests and most HTTP libraries take a dict. With a rotating gateway, every request that constructs a new session can come out on a different exit IP.

python
import requests

proxies = {
    "http": "http://USERNAME:PASSWORD@gw.dataimpulse.com:823",
    "https": "http://USERNAME:PASSWORD@gw.dataimpulse.com:823",
}

print(requests.get("https://httpbin.org/ip", proxies=proxies).json())


One thing that trips people up: when the proxy address uses `http://` but the target is `https://`, that's not a mistake. It means "talk to the proxy in cleartext, then tunnel HTTPS through it." Use `https://` in the proxy string only when the proxy itself terminates TLS on the client connection.

## Where free HTTPS proxy lists fall apart

Public lists are scraped, shared and short-lived. The same IPs get handed to thousands of people, so by the time a list is indexed, a chunk of the entries are dead and the rest are heavily rate-limited or already blocked by any site with serious bot detection.

Three failure modes show up over and over:

- **They die fast.** An entry that answered a health check this morning can be gone by the afternoon, which turns a "quick" scraping job into an IP-rotation maintenance project.
- **They're shared.** Reputation is pooled with every other user of that address, including people running far sketchier workloads than yours.
- **They may intercept.** TLS interception with a supplied certificate, credential harvesting from plain-text requests, and redirects to ad pages are all documented behaviors on free infrastructure.

Free is a reasonable way to learn the syntax. It's a bad way to run anything that touches logins, pricing data or a deadline.

## What to verify before you pay anyone

If you're buying, these are the specs that decide whether the thing works for your workload — and the ones marketing pages tend to bury.

| Check | Why it matters |
| --- | --- |
| Protocol support | You need HTTP, HTTPS and SOCKS5 available, not just one |
| Rotating vs sticky | Rotating changes IP per request; sticky holds one IP for a session |
| Maximum sticky duration | Capped at 30 minutes or 24 hours changes what you can automate |
| Targeting cost tiers | Country-level is often free; city, ZIP and ASN often aren't |
| Traffic expiry | "Monthly reset" quietly deletes money you already spent |
| Minimum purchase | A $500 floor makes testing expensive before you know it works |
| Refund window and exclusions | Check crypto exclusions and usage thresholds |
| Auth methods | Username/password is more portable than IP whitelisting alone |

## DataImpulse, checked against that list

DataImpulse runs a first-party pool of 90M+ IPs across 195 countries, split into four product lines: residential, datacenter, mobile, and premium residential. Everything is pay-as-you-go with no subscription, and purchased traffic doesn't expire — you buy gigabytes, they sit in your balance until consumed. The minimum top-up is $5, which is what the Intro tiers are built around.

Protocol and port details, per the provider's own documentation and integration guides:

- HTTP, HTTPS and SOCKS5 all supported
- Rotating HTTP/HTTPS on port **823**, rotating SOCKS5 on port **824**
- Sticky sessions on ports **10000–20000**, using a session token in the username
- Targeting appended to the username string, e.g. `USERNAME:PASSWORD_session-abc123_country-us_city-newyork@gw.dataimpulse.com:823`
- Two authentication modes: username/password, or IP whitelisting
- 24/7 human support, and a 7-day money-back guarantee on first purchases, capped at 80% traffic consumption and excluding crypto payments

The gateway host is `gw.dataimpulse.com`. That's the piece you'll paste into every tool above.

## All current plans, side by side

Every plan published on the official pricing pages, priced per gigabyte of traffic. No subscription attaches to any of these.

| Proxy type | Plan | Traffic | Price | Per GB | Purchase |
| --- | --- | --- | --- | --- | --- |
| Residential | Intro | 5 GB | $5 | $1.00 | start with the 5 GB residential test pack |
| Residential | Basic | 50 GB | $50 | $1.00 | grab the 50 GB residential plan |
| Residential | Advanced | 1 TB | $800 | $0.80 | buy 1 TB of residential traffic at the volume rate |
| Residential | Custom+ | 5 TB+ | from $4,000 | custom | request volume pricing for residential |
| Datacenter | Intro | 10 GB | $5 | $0.50 | try the 10 GB datacenter intro plan |
| Datacenter | Basic | 100 GB | $50 | $0.50 | pick up 100 GB of datacenter traffic |
| Datacenter | Advanced | 1 TB | $450 | $0.45 | order the 1 TB datacenter plan |
| Datacenter | Custom+ | 5 TB+ | from $2,250 | custom | ask for datacenter volume pricing |
| Mobile | Intro | 2.5 GB | $5 | $2.00 | test mobile proxies with the 2.5 GB pack |
| Mobile | Basic | 25 GB | $50 | $2.00 | buy 25 GB of mobile traffic |
| Mobile | Advanced | 1 TB | $1,600 | $1.60 | get the 1 TB mobile plan |
| Mobile | Custom+ | 5 TB+ | from $8,000 | custom | contact sales about mobile volume |
| Premium residential | Intro | 1 GB | $5 | $5.00 | check the premium residential pool |
| Premium residential | Basic | 10 GB | $50 | $5.00 | buy 10 GB of premium residential traffic |
| Premium residential | Custom+ | 5 TB+ | from $20,000 | custom | request custom premium residential pricing |

Two things worth pulling out of that table. First, the Intro tiers are all $5 regardless of product, so testing a proxy type costs the same whether you're checking $1/GB residential or $5/GB premium. Second, volume discounts kick in at 1 TB on residential and mobile — 20% off, which is where the $0.80/GB and $1.60/GB numbers come from. Premium residential has no 1 TB tier; it jumps from 10 GB straight to custom enterprise pricing at 5 TB+.

## The targeting surcharge nobody reads

This is the detail that turns a $1/GB plan into a $2/GB plan if you're not paying attention.

Country-level targeting is included in the base price across the residential product. State, city, ZIP and ASN targeting on standard residential traffic is billed at **double the per-GB rate**. Premium residential includes all targeting at no surcharge, which is part of why it's priced at $5/GB. Datacenter product pages list state, city, ZIP and ASN targeting as included rather than surcharged — worth confirming with support before you build a budget around it, since that treatment differs from the residential line.

Practical consequence: if your project needs city or ZIP precision, run the math before defaulting to the cheapest tier. At 2× billing, standard residential costs the same as premium residential per targeted gigabyte — and premium residential throws in a dedicated account manager.

## Which plan actually fits

Skip the upsell logic and go by workload:

**Datacenter, $0.50/GB** — public data, unprotected news sites, open databases, price feeds that don't fight back. The 10 GB Intro is $5, so the cost of being wrong is a coffee.

**Residential, $1/GB** — the default for anything with real bot detection: e-commerce, SERPs, ad verification, social platforms. Start at 5 GB, measure success rate, then decide. This is the tier where country targeting is free, so geo-restricted content checks are cheap.

**Mobile, $2/GB** — when residential keeps getting flagged. Carrier IPs from 3G/4G/5G/LTE networks carry fingerprints that anti-bot systems weight differently. It's four times the residential rate, so use it where the extra legitimacy changes the outcome, not as a blanket upgrade.

**Premium residential, $5/GB** — high-stakes or latency-sensitive jobs where you need precision targeting included and someone to call. Below that, the surcharge math above usually makes it redundant.

One more consideration: traffic never expires, so buying 100 GB today and spreading it across several months costs nothing extra. That's the opposite of subscription billing, where an idle month is money burned.

## A sane first-run sequence

1. Create the account and add the $5 minimum — no sales call, no approval queue.
2. Pick one product line. Residential if you're unsure; datacenter if your targets are forgiving.
3. Set authentication. Username/password is portable across machines; IP whitelisting is fine for a fixed server.
4. Build the credential string with your targeting: `USER:PASS_country-us` for country, add `_city-...` or `_session-...` as needed.
5. Fire one `curl -x` request at an IP echo endpoint and confirm the exit location.
6. Only then wire it into Scrapy, Selenium, Playwright or Puppeteer.

If the second request comes back on a different IP, rotation is working. If it comes back on the same IP across several minutes, you've likely hit a sticky port rather than a rotating one — check that you're on 823 or 824, not in the 10000–20000 range.

## Straight answers to the questions that come up

**Is an HTTPS proxy the same as a VPN?** No. A VPN routes your whole device at the OS level. A proxy is per-app or per-request, which is why you can send one Python script through a residential exit while your browser stays on your home connection.

**Do I need a proxy that speaks HTTPS to reach HTTPS sites?** No. A plain `http://` proxy tunnels HTTPS fine via CONNECT. Use an `https://` proxy address only when you specifically want the client-to-proxy leg encrypted.

**Does DataImpulse traffic expire?** No. Balances persist until consumed, and there's no monthly reset.

**Is there a free trial?** Not a no-payment one. The entry point is $5, and first purchases come with a 7-day money-back guarantee, subject to the 80% usage cap and the crypto exclusion.

**Why not just use a free list?** Because you're trusting whoever runs it with your traffic metadata, and the addresses are shared with everyone else on the internet. Fine for learning `-x` syntax, poor for anything with a deadline.

## The short version

"HTTPS proxy" describes both the everyday case — a proxy tunneling your HTTPS traffic — and the narrower case of a TLS-encrypted client-to-proxy hop. Get the distinction right and the configuration stops being guesswork.

Paste the credentials into `https_proxy`, curl's `-x`, Chrome's `--proxy-server`, or a Python dict and you're running. The real decision is whether you're pointing at a provider whose pricing you can predict. At $1/GB residential with non-expiring traffic and country targeting included, DataImpulse is one of the few where the number on the pricing page is the number you pay — provided your workflow doesn't need city or ZIP filtering, where the 2× surcharge applies.
