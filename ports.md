# Port rules

The ports every app of Beyonda Labs listens on, in development and once deployed, so two of them never collide on
the same machine. A port is taken here before an app uses it.

## Rules

- **Every port is reserved here first**: a new app, a new deployment or a new dev server adds its row to the table
  below in the same change that makes it listen, and no port is reused while its row stands.
- **A product deploys on one port in the 8080–8099 range**: its service serves the front and the API under `/api` from
  one image, as [deployment.md](deployment.md) describes. Inside the container it keeps the dev port of its service;
  the reserved one is the port the host publishes.
- **Development keeps the ports of the tools**: 4200 for an Angular dev server and 3000 upwards, one per app, for an
  Express service. Two Angular apps share 4200 and never run at once; `ng serve --port` moves one when they must.
- **The port of a deployment is a default, never a constant**: the compose takes it from `PORT`, so a machine with a
  conflict changes the variable, not the image.
- **On the server (192.168.1.20) only Caddy publishes a port**, 443: every service there is reached by its
  `.home.arpa` name over HTTPS, also from outside through Tailscale, whose ACL allows 443 to that address. The ports
  of its containers are not mapped to the host, so `http://192.168.1.20:<port>` never answers, and an address such
  as the registry is always written by name and without a port (`https://verdaccio.home.arpa/`).

## Reserved ports

| Port  | App                      | Where                                                          |
| ----- | ------------------------ | -------------------------------------------------------------- |
| 443   | Caddy                    | Server: the only port published, HTTPS for every service       |
| 3000  | express-components-demo  | Development                                                    |
| 3100  | document-builder-service | Development                                                    |
| 4200  | document-builder-front   | Development (Angular dev server)                               |
| 4200  | angular-components-demo  | Development (Angular dev server), never with the other         |
| 4873  | Verdaccio                | Server: inside its container, reached at `verdaccio.home.arpa` |
| 8080  | Jenkins                  | Server: inside its container, reached at `jenkins.home.arpa`   |
| 8084  | document-builder         | Deployment: the front and the API under `/api`                 |
| 50000 | Jenkins agents           | Server: inside its container, unused                           |

The ports inside the containers of the server are listed so that no product publishes one of them on that machine:
8080 sits in the deployment range and stays out of it. The agent of Jenkins connects over WebSocket through 8080, so
50000 is never used.

## Example

```yaml
# docker/compose.yaml of a product
services:
  document-builder:
    image: document-builder:${VERSION:-latest}
    ports:
      - "${PORT:-8084}:3100"
```
