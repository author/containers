# auto-letsencrypt-cloudflare

This "always-on" Linux-only container keeps LetsEncrypt TLS/SSL certificates up to date. Post-create/renew hooks and physical file mirrors differentiate this container from other LetsEncrypt tools.

## Key Features:

- **Always-on** (cron) runs the update process on a regular schedule (configurable).
- **Hooks**: Optionally runs a user-defined script after successful certificate creation/renewal.
- **Cloudflare DNS Verification** (can be configured multiple times to support multiple domains).
- **Physical Files**: For times when an application cannot read LetsEncrypt symlinks (see "Alternative to LetsEncrypt Symlinks" below).

### Example Use cases:

- Auto-refresh LDAP certificates and reload the LDAP server upon completion.
- Auto-renew Node.js certificates.
- Auto-refresh a secure SMTP server.
- Trigger webhooks when certificates are updated.
- Send email notifications when certificates are updated.

While it is possible to use this to refresh certificates for HTTP servers, there are many other dedicated tools better designed for that specific purpose (Caddy, NGINX, etc).

## Running in Docker

**Environment Variables**

| Name                               |   Required   |     Default     | Description                                                                                                  |
| :--------------------------------- | :-----------: | :--------------: | :----------------------------------------------------------------------------------------------------------- |
| *`AFTER_RENEW_SCRIPT`            |      No      |        -        | The command or path of the script_on the host_ to run after the successful creation/renewal of certificates. |
| `DNS_PROPAGATION_SECONDS`        |      No      |      `10`      | Number of seconds to allow for DNS changes before LetsEncrypt verifies the DNS entries.                      |
| **`DOMAIN`**               | **Yes** |        -        | The domain to create/renew LetsEncrypt certificates for                                                      |
| **`CLOUDFLARE_API_TOKEN`** | **Yes** |        -        | The Cloudflare API token. This token must permissions to create DNS entries.                                 |
| `CRON`                           |      No      | `0 */12 * * *` | The schedule to run the renewal process on. Default is every 12 hours.                                       |
| *`NO_RENEW_SCRIPT`               |      No      |        -        | The command or path of the script_on the host_ to run if a certificate is not created/renewed.               |

*Scripts reside on the host, not within the Docker container. They are executed via a [docker-host-ipc](../docker-host-ipc) channel between the container and the host.

**Volumes**

| Target     | Purpose                                                |
| :--------- | :----------------------------------------------------- |
| `/certs` | Maps to the LetsEncrypt directory.**(REQUIRED)** |
| `/ipc`   | Maps to the named pipe. **(SEE NOTES)**         |

**Notes**
If you wish to run scripts in response to creation/renewal (or lack thereof), you can run them in the container or on the host. Running within the container will isolate the script to the container. This is useful for running scripts that trigger webhooks, remote processes, etc. However; the script is limited to running in the confines of the container. There are circumstances where you may wish for the script to run on the Docker host instead of within the Docker container. There are two ways to do this.

1. Use the `--privileged` flag when running the container. This is a brute force approach that grants the container significant privileges (the same as the host kernel). It should be used with caution, in trusted environments only.
2. Use an IPC channel. This container will look for a named pipe at `/ipc` and pass commands to the named pipe, allowing them to run on the host. This requires additional configuration on the host. See the "IPC Communication" section for instructions.

_Example:_

```sh
docker run --rm \
  --restart unless-stopped \
  -e "DOMAIN=my.domain.com" \
  -e "CLOUDFLARE_API_TOKEN=mytoken" \
  -e "CRON=* 0/12 * * * *" \
  -e "AFTER_SUCCESS=/HOST_PATH/to/scriptname" \
  -e "AFTER_ABORT=/HOST_PATH/to/scriptname" \
  -e "IPC=/ipc/channel" \
  -v "/ipc:/ipc" \ # Maps the docker-host-ipc channel directory
  -v "/etc/letsencrypt:/certs" \
  author/autorenew-letsencrypt-cloudflare
```

_or as part of a docker-compose file..._

```sh
version: '3'
services:
  letsencrypt:
    image: author/autorenew-letsencrypt-cloudflare:latest
    container_name: mydomain_ssl_renewer
    environment: autorenew-letsencrypt-cloudflare
      DOMAIN: my.domain.com
      CLOUDFLARE_API_TOKEN: mytoken
      CRON: "0 */12 * * *"
      AFTER_SUCCESS: /handlers/scriptname
      AFTER_ABORT: /handlers/scriptname
    volumes:
      /path/to/letsencrypt/live:/certs
    restart: unless-stopped
```

## Alernative to LetsEncrypt Symlinks

LetsEncrypt generates pretty file names like `cert.pem` and `fullchain.pem`. However, these are symlinks to the most recently archived certificate assets, not true files. For example:

`/etc/letsencrypt/live/my.domain.com/cert.pem -> /etc/letsencrypt/archive/my.domain.com/cert1.pem`

Some applications cannot reliably read symlinks. Using the archive file directly poses a problem because the filename changes every time the certificate is renewed. This container makes a copy of the assets as physical files. Each time the certificate is renewed, the copy is recreated, assuring the same file name is available for each physical file. This avoids.

In addition to the standard LetsEncrypt symlinks, the following files are generated in the `/etc/letsencrypt/live/<domain>` directory:

| Physcial File       | LetsEncrypt Equivalent                                     |
| :------------------ | :--------------------------------------------------------- |
| `trusted.pem`     | `fullchain.pem` + `privkey.pem`(concatenated into one) |
| `fullchain.crt`   | `fullchain.pem`                                          |
| `ca.pem`          | `chain.pem`                                              |
| `certificate.pem` | `cert.pem`                                               |
| `private.key`     | `privkey.pem`                                            |

## IPC Communication

In some cases, you may want to run commands on the host after a certificate is created/renewed. For example, you may wish to use LetsEncrypt certificates to secure an LDAP server running in another container . When the certificates are renewed, you may need to restart the LDAP Docker container or execute a command using `docker exec` on the LDAP container. This can only be done from the Docker host, not within an unprivileged Docker container.

This container supports a [docker-host-ipc](../docker-host-ipc) channel (`/ipc`) to allow the execution of scripts/commands after the creation/renewal of certificates completes (or abort). This channel can be mapped into the `/ipc` volume of this container, allowing post-creation/renewal commands/scripts to run on the host.

---

Copyright (c) 2024 Author Software, Inc.
