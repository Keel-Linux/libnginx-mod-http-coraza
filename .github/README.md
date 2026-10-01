# libnginx-mod-http-coraza

Debian packaging of [coraza-nginx](https://github.com/corazawaf/coraza-nginx)
0.21.0, the [OWASP Coraza](https://github.com/corazawaf/coraza) web
application firewall as a dynamic module of trixie's nginx 1.26.3, for
Keel Linux on Debian trixie. Source and binary package
`libnginx-mod-http-coraza`.

## Why Keel carries it

Keel Web runs Coraza inline in Nginx (handbook decision 0030), and
everything Keel installs is a `.deb` in the Keel repository (0039).
Step 5 of the first implementation of 0041 builds Coraza's packages:
this one, [libcoraza](https://github.com/Keel-Linux/libcoraza) and
[coreruleset](https://github.com/Keel-Linux/coreruleset).

`dh-sequence-nginx` builds the module against `nginx-dev` and makes it
depend on `nginx-abi-1.26.3-1`, as trixie's own `libnginx-mod-*` packages
do, so it must be rebuilt whenever trixie's nginx moves to another
upstream version. The module loads `libcoraza.so.1` with `dlopen` in each
worker, after the fork: the Go runtime does not survive a fork.

## Debian status

No package of the Coraza connector in Debian, and no ITP.

## Layout

[DEP-14](https://dep-team.pages.debian.net/deps/dep14/), as
git-buildpackage repositories on salsa: `upstream/latest` (upstream
tarballs, imported with `gbp import-orig`), `pristine-tar`, and
`keel/trixie` (this packaging, the default branch). Tags are
`upstream/<version>` and `keel/<debian-version>`.

## Building

On trixie. `libcoraza-dev` is not in Debian: build and install it from
[libcoraza](https://github.com/Keel-Linux/libcoraza) first, which needs
trixie-backports for `golang-1.26-go`.

```
sudo apt-get install git-buildpackage pristine-tar
gbp clone https://github.com/Keel-Linux/libnginx-mod-http-coraza.git
cd libnginx-mod-http-coraza
sudo apt-get install ../libcoraza1_*.deb ../libcoraza-dev_*.deb
sudo apt-get build-dep ./
gbp buildpackage -us -uc
```

## Tests

At build time, `nginx -t` with the module loaded. The autopkgtest,
`debian/tests/crs-blocks`, installs the module with coreruleset and
trixie's nginx, checks that the workers loaded libcoraza, and asserts 200
for normal requests and 403 for SQL injection, cross-site scripting and
path traversal, with the CRS rules named in Coraza's audit log. It needs
systemd; in a disposable trixie container or machine booted with systemd:

```
autopkgtest --ignore-restrictions=isolation-container \
    libnginx-mod-http-coraza_*.dsc libcoraza1_*.deb coreruleset_*.deb \
    libnginx-mod-http-coraza_*.deb -- null
```

CI builds libcoraza and coreruleset from their `keel/trixie`, builds this
package in a `debian:trixie` container with trixie-backports, runs
lintian (any error or warning fails), and runs the autopkgtest as above
in a trixie LXC system container on the self-hosted
runner keel-lxc-1. The build runs on a GitHub-hosted runner, as
Keel-Linux/common builds its packages.

## License

Apache-2.0, as upstream (`LICENSE`) and the packaging (`debian/copyright`);
`src/ddebug.h` is BSD-2-clause.
