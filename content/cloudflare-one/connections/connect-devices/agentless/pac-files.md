---
pcx_content_type: how-to
title: HTTP
weight: 1
---

# Enable Gateway proxy with PAC files

{{<Aside type="note">}}

Only available on Enterprise plans.

{{</Aside>}}

You can apply Gateway HTTP and DNS policies at the browser level by configuring a Proxy Auto-Configuration (PAC) file.

{{<glossary-definition term_id="PAC file" prepend="A PAC file is ">}}

When end users visit a website, their browser will send the request to a Cloudflare proxy server associated with your account to be filtered by Gateway. Note that Gateway [cannot filter every type of HTTP traffic](#limitations) proxied using PAC files.

## Prerequisites

Install the [Cloudflare certificate](/cloudflare-one/connections/connect-devices/warp/user-side-certificates/) on your device.

## 1. Generate a proxy endpoint

You can generate a proxy endpoint on the Zero Trust dashboard or through the Cloudflare API.

{{<Aside type ="warning">}}
All devices you add to the proxy endpoint will be able to access your Cloudflare Tunnel applications and services. If you only want to proxy web traffic, you can build a network policy that blocks those source IPs from connecting to your internal resources.
{{</Aside>}}

{{<tabs labels="Dashboard | API">}}

{{<tab label="dashboard" no-code="true">}}

1. In [Zero Trust](https://one.dash.cloudflare.com/), go to **Gateway** > **Proxy Endpoints**.
2. Select **Create endpoint**.
3. Give your endpoint any name.
4. Enter the public source IP address of your device(s) in CIDR notation. For example:

   - **IPv4**: `192.0.2.0/8`
   - **IPv6**: `2001:0db8:0000:0000:0000:1234:5678:0000/109`

   {{<Aside type="note">}}

   Gateway limits the prefix length of source networks for proxy endpoints to `/8` for IPv4 networks and `/32` for IPv6 networks.

   {{</Aside>}}

5. Select **Save endpoint** and confirm the endpoint creation.

Your Cloudflare proxy server domain is of the form:

```txt
https://<SUBDOMAIN>.proxy.cloudflare-gateway.com
```

{{</tab>}}

{{<tab label="api" no-code="true">}}

1. [Create a proxy endpoint](/api/operations/zero-trust-gateway-proxy-endpoints-create-proxy-endpoint) with the following call:

   ```bash
   curl https://api.cloudflare.com/client/v4/accounts/<ACCOUNT_ID>/gateway/proxy_endpoints \
   --header "Authorization: Bearer <API_TOKEN>" \
   --header "Content-Type: application/json" \
   --data '{"name": "any_name", "ips": ["<PUBLIC_IP>", "<PUBLIC_IP2>", "<PUBLIC_IP3>"]}'
   ```

   Replace `<PUBLIC_IP>` with the source IP address of your device in CIDR notation. For example:

   - **IPv4**: `192.0.2.0/8`
   - **IPv6**: `2001:0db8:0000:0000:0000:1234:5678:0000/32`

   {{<Aside type="note">}}

   Gateway limits the prefix length of source networks for proxy endpoints to `/8` for IPv4 networks and `/32` for IPv6 networks.

   {{</Aside>}}

2. After running the call, you should see an output similar to

   ```bash
   ---
   header: Example output
   highlight: 10
   ---
   {
   "result": {
       "id": "d969d7bf-ec28-4291-9af0-86825f472c21",
       "name": "test",
       "created_at": "2022-03-02T10:57:18.094789Z",
       "updated_at": "2022-03-02T10:57:18.094789Z",
       "ips": [
       "90.90.241.229/8"
       ],
       "subdomain": "3ele0ss56t"
   },
   "success": true,
   "errors": [],
   "messages": []
   }
   ```

   Note the `subdomain` value returned by the API. Your Cloudflare proxy server domain is of the form:

   ```txt
   <SUBDOMAIN>.proxy.cloudflare-gateway.com
   ```

   In the example above, the subdomain is `3ele0ss56t` and the proxy server domain is `3ele0ss56t.proxy.cloudflare-gateway.com`.

{{</tab>}}

{{</tabs>}}

## 2. Test your proxy server

1. In [Zero Trust](https://one.dash.cloudflare.com/), create an [HTTP policy](/cloudflare-one/policies/gateway/http-policies/) for testing purposes. For example:

   | Selector | Operator | Value         | Action |
   | -------- | -------- | ------------- | ------ |
   | Domain   | in       | `example.com` | Block  |

2. Verify that nothing is returned by a `curl` command:

   ```sh
   $ curl -4 -p -x https://3ele0ss56t.proxy.cloudflare-gateway.com https://example.com
   ```

If `curl` returns a `403` code, it means the public IP of your device does not match the one used to generate the proxy server. Make sure that WARP is turned off on your device and double-check that curl is not using IPv6 (use the `-4` option to force IPv4).

## 3. Create a PAC file

A PAC file is a text file that specifies which traffic should redirect to the proxy server.

Below is the default PAC file. You can [customize the file](https://developer.mozilla.org/en-US/docs/Web/HTTP/Proxy_servers_and_tunneling/Proxy_Auto-Configuration_PAC_file) and host it somewhere your browser can access, such as on an internal web server or on [Cloudflare Workers](/workers/).

```js
function FindProxyForURL(url, host) {
  // No proxy for private (RFC 1918) IP addresses (intranet sites)
  if (
    isInNet(dnsResolve(host), "10.0.0.0", "255.0.0.0") ||
    isInNet(dnsResolve(host), "172.16.0.0", "255.240.0.0") ||
    isInNet(dnsResolve(host), "192.168.0.0", "255.255.0.0")
  ) {
    return "DIRECT";
  }

  // No proxy for localhost
  if (isInNet(dnsResolve(host), "127.0.0.0", "255.0.0.0")) {
    return "DIRECT";
  }

  // Proxy all
  return "HTTPS 3ele0ss56t.proxy.cloudflare-gateway.com:443";
}
```

{{<Aside type="note">}}

- Make sure the directive used for the endpoint is `HTTPS` and not `PROXY`.
- You must use a PAC file instead of configuring the endpoint directly in the proxy configuration of the browser. This is because modern browsers still do not support HTTPS proxies without PAC files.
- Use a proper text editor such as VS Code to avoid added characters.
  {{</Aside>}}

## 4. Configure your browser

All major browsers support PAC files. You can configure individual browsers, or you can configure system settings that apply to all browsers on the device. Multiple devices can call the same PAC file as long as their source IP addresses were included in the proxy endpoint configuration.

The following example demonstrates the setup procedure for Firefox.

1. In Firefox, go to **Settings** and scroll down to **Network Settings**.

   ![Going to Network Settings menu in Firefox](/images/cloudflare-one/connections/firefox-network-settings.png)

2. Select **Settings**.
3. Select **Automatic proxy configuration URL**.
4. Enter the URL where your PAC file is hosted, for example `https://proxy-pac.cflr.workers.dev/3ele0ss56t.pac`.

   ![Enter PAC file URL into Firefox](/images/cloudflare-one/connections/firefox-pac-file.png)

5. Select **OK**. HTTP traffic from Firefox is now being filtered by Gateway.

## 5. Test your HTTP policy

You can test any [supported HTTP policy](#limitations), such as the example policy created in [Step 2](#2-test-your-proxy-server). When you go to `https://example.com` in your browser, you should see the Gateway block page.

## Limitations

### HTTP policy proxy endpoint specificity

The Gateway HTTP policy does not have a [Proxy Endpoint selector](/cloudflare-one/policies/gateway/network-policies/#proxy-endpoint) to match specific proxy endpoints. Despite this, Gateway will still apply HTTP policies to traffic sent to Cloudflare proxy servers.

### Traffic limitations

At this time, the agentless HTTP proxy does not support [identity-based policies](/cloudflare-one/policies/gateway/identity-selectors/), mTLS authentication, or UDP traffic. To enforce HTTP policies for UDP traffic, you must [disable QUIC](/cloudflare-one/policies/gateway/http-policies/http3/#prevent-inspection-bypass) in your users' browsers.
