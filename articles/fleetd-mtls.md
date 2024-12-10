# mTLS support in fleetd

The [Fleetd Authentication](https://fleetdm.com/guides/fleetd-authentication#basic-article) article shows how Fleet's agent, fleetd, authenticates to Fleet and [TUF](https://fleetdm.com/guides/fleetd-updates#basic-article) servers.

Additionally, Fleet Premium admins can configure fleetd to use [mTLS](https://en.wikipedia.org/wiki/Mutual_authentication) on top of the existing authentication scheme to further increase the security of agent to server communication.

> The Fleet server itself does not currently provide support for mTLS. 
> Admins that want to use mTLS on their endpoints must setup a load balancer or TLS terminator like AWS's ELB or nginx that support mTLS.

## Configuration

Admins can either generate the fleetd installer with the client certificate files included, or, can deploy the client certificate files to devices where fleetd is already installed.

The client certificates must be in PEM format.

### Generating fleetd installers with client certificates

When generating the packages, admins can use the following flags to configure the client certificates:
```sh
fleetctl package \
  [...]
  # Client certificate to connect to Fleet servers.
  --fleet-tls-client-certificate=/path/to/fleet-client.crt \
  --fleet-tls-client-key=/path/to/fleet-client.key \
  # Client certificates can be provided when connecting to custom TUF servers that require mTLS.
  --update-tls-client-certificate=/path/to/update-client.crt \
  --update-tls-client-key=/path/to/update-client.key \
  --update-url=https://example.tuf.com \
  [...]
```
When `--update-tls-client-certificate` and `--update-tls-client-key` are provided,`fleetctl` will use them when downloading the fleetd components from the custom TUF server (`--update-url`).

If you are using fleetd with `Fleet Desktop` enabled, you may need to specify an alternative host for the "My device" URL (in the Fleet tray icon).
Such alternative host should not require client certificates on the TLS connection.
```sh
fleetctl package
  [...]
  --fleet-desktop \
  --fleet-desktop-alternative-browser-host=fleet-desktop.example.com \
  [...]
```
If `--fleet-desktop-alternative-browser-host` is not used, you will need to configure client TLS certificates on devices' browsers.

### Deploying client certificates to devices

> Fleet currently does not natively support deploying client certificates to devices. Tooling like Chef, Ansible, or Puppet could be used for this purpose.

Once fleetd is installed, admins can force fleetd to use mTLS to communicate with Fleet and custom [TUF](https://fleetdm.com/guides/fleetd-updates#basic-article) servers by deploying the client certificates to the devices on the following locations:
- macOS and Linux:
  - Connection to Fleet servers:
    - `/opt/orbit/fleet_client.crt`
    - `/opt/orbit/fleet_client.key`
  - Connection to custom TUF servers:
    - `/opt/orbit/update_client.crt`
    - `/opt/orbit/update_client.key`
- Windows:
  - Connection to Fleet servers:
    - `C:\Program Files\Orbit\fleet_client.crt`
    - `C:\Program Files\Orbit\fleet_client.key`
  - Connection to custom TUF servers:
    - `C:\Program Files\Orbit\update_client.crt`
    - `C:\Program Files\Orbit\update_client.key`

If you are using fleetd with `Fleet Desktop` enabled, you may need to specify an alternative host for the "My device" URL (in the Fleet tray icon).
Such alternative host should not require client certificates on the TLS connection.
The `ORBIT_FLEET_DESKTOP_ALTERNATIVE_BROWSER_HOST` environment variable in `orbit`'s configuration can be used to configure the Fleet deskto alternative host.
If `ORBIT_FLEET_DESKTOP_ALTERNATIVE_BROWSER_HOST` is not set, you will need to configure client certificates on devices' browsers.

## fleetd components using mTLS

Once configured, fleetd will use the provided client certificates on all components so that all communication from the endpoints to Fleet and TUF servers use mTLS.
- `orbit` will use the provided client certificates to connect to Fleet servers.
- `orbit` will use (if provided) client certificates to connect to custom TUF servers.
- `orbit` will configure `osqueryd` and `Fleet desktop` to use the provided client certificate to connect to the Fleet server.

```mermaid
graph LR;
    tuf[Custom TUF server];
    fleet_server[Fleet server];
    nginx["TLS terminator with<br> mTLS support<br>(e.g. nginx/ELB)"];

    nginx -- HTTPS --> fleet_server;
    nginx -- HTTPS --> tuf;

    subgraph fleetd
        subgraph orbit
            subgraph server[Client certificates];
            fleet_client_certificate_orbit[fleet_client.crt<br>fleet_client.key];
            update_client_certificate_orbit[update_client.crt<br>update_client.key];
            end
        end
        subgraph osquery[osquery];
            client_certificate_osquery[--tls_client_cert=fleet_client.crt<br>--tls_client_key=fleet_client.key];
        end
        subgraph desktop [Fleet desktop]
            client_certificate_desktop["(from env)<br>fleet_client.crt<br>fleet_client.key"];
            server_certificate_desktop["(from env)<br>fleet.pem"];
        end

        desktop_browser[Fleet Desktop<br> from Browser];
    end

    orbit -- "Fleet Orbit API mTLS" --> nginx;
    desktop -- "Fleet Desktop API mTLS" --> nginx;
    osquery -- "osquery<br>remote API mTLS" --> nginx;
    desktop_browser -- "My Device URL TLS" --> fleet_server;

    orbit -- TUF mTLS --> nginx;

    subgraph fleetctl[fleetctl package command]
        update_client_certificate_fleetctl[update_client.crt<br>update_client.key];
    end
    fleetctl -- TUF mTLS --> nginx;
```

If you have suggestions for how to improve mTLS functionality in Fleet, please share them with us in the osquery Slack [#fleet channel](https://fleetdm.com/slack) or open an issue in Github.

<meta name="authorGitHubUsername" value="lucasmrod">
<meta name="authorFullName" value="Lucas Rodriguez">
<meta name="publishedOn" value="2024-12-06">
<meta name="articleTitle" value="mTLS support in fleetd">
<meta name="category" value="guides">