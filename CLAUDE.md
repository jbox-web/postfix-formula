# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

The parent `salt-formulas/CLAUDE.md` describes the toolchain, the CI and the
`template-formula` layout shared by the collection. This file only records how
`postfix-formula` differs from it — and it differs a lot: the state tree predates
the template and was never migrated.

## Commands

```bash
bundle install
bin/kitchen list                                  # 3 instances: default-<platform>
bin/kitchen test default-debian-13-3008-2         # converge + verify one instance
bin/kitchen converge default-debian-13-3008-2     # then iterate with verify
bin/kitchen verify default-debian-13-3008-2 --controls 'Postfix maps'

pre-commit run --all-files
```

There is a single Kitchen suite, `default`. Its `state_top` applies
`postfix._mapdata`, `postfix` and `postfix.config`, with the pillar
`test/salt/pillar/default.sls`. The InSpec controls live in
`test/integration/default/controls/` (`pkgs`, `postfix`, `postfix_maps`,
`postfix_maps_type`, `service`). There are **no** `_mapdata` reference dumps:
the `_mapdata` state runs, but nothing compares its output.

pre-commit here passes the matching file list to each hook (no `always_run`).

## Layout — not the template

- **Flat states, no `package/` / `config/` / `service/` split.** `postfix/init.sls`
  installs the package, runs the service and manages `/etc/aliases`;
  `postfix/config.sls` includes `postfix` and manages `main.cf`, `master.cf`,
  TLS certificates, the `postfix:mapping` lookup tables and the `vmail` SQL map
  files. The other states are standalone add-ons, none of them exercised by
  Kitchen: `mysql`, `pcre`, `policyd-spf`, `postgrey`, `postsrsd`,
  `iptables-input`.
- **Paths are literal.** States import `"postfix/map.jinja"` and use
  `salt://postfix/files/...` directly; only `_mapdata/init.sls` derives `tplroot`.
  There is no `libtofs.jinja` and no `files/default/`: templates sit flat in
  `postfix/files/`.
- **Cross-state ordering uses state IDs, not `sls:` requisites.** Everything
  hangs off `pkg: postfix-init-pkg-installed-postfix` and
  `service: postfix-init-service-running-postfix` (IDs follow
  `postfix-<sls>-<module>-<function>-<name>`). Renaming either breaks every
  add-on state and any external `watch_in`.
- **`map.jinja` has only two layers**: `defaults.yaml` <- `osfamilymap.yaml`
  (by `os_family`) <- pillar `postfix:lookup`. No `osarchmap`, `osmap` or
  `osfingermap`. The resulting dict carries packages and paths only
  (`config_path`, `root_grp`, `xbin_prefix`, `daemon_directory`,
  `default_database_type`, ...).
- **The configuration itself bypasses `map.jinja`.** Templates and states read
  `salt['pillar.get']('postfix:config')`, `postfix:mapping`,
  `postfix:master_config`, ... straight from pillar. Defaults for `main.cf`
  parameters are therefore in `files/main.cf`, not in `defaults.yaml`.

## How the configuration is rendered

- **`main.cf`** — `set_parameter(name, default)` emits `postfix:config:<name>`
  or the default and records it as processed. The template builds
  `smtpd_relay_restrictions` / `smtpd_recipient_restrictions` from the SASL,
  relayhost, policyd-spf and postgrey pillar switches, then appends every
  `postfix:config` key not yet emitted. A list value is rendered one item per
  continuation line.
- **Lookup tables** — each key of `postfix:mapping` is a `main.cf` parameter
  name. Its file path and type come from `postfix:config:<same key>`
  (`hash:/etc/postfix/transport`, `proxy:` prefix stripped, relative paths
  resolved under `config_path`); the same logic is duplicated in `config.sls`
  and `files/main.cf`, keep them in sync. The data is a list of single-key
  dicts (order preserved) or a dict, rendered by `files/mapping.j2`; list
  values are joined with `, `. `postmap` runs `onchanges` for indexed types
  (`hash`, `btree`, `cdb`, `cidr`, `dbm`, `pcre`, `regexp`, `sdbm`). Maps whose
  name ends in `_sasl_password_maps` get mode `0600` and `show_changes: False`.
- **`master.cf`** — rendered only when `postfix:manage_master_config` (default
  true). `postfix/services.yaml` holds every service's `defaults` and the
  `order` they are written in; `postfix:master_config:services:<name>` overrides
  a service, `enable: false` comments it out. Legacy switches
  (`enable_submission`, `enable_dovecot`, `policyd-spf:enabled`, ...) are
  translated into extra services inside the template for backwards
  compatibility. A new built-in service needs an entry in both `defaults` and
  `order`.
- **`/etc/aliases`** — `postfix:aliases:use_file` (default true) writes the
  whole file and runs `newaliases`; false manages entries one by one with
  `alias.present` / `alias.absent`.

`docs/README.rst` and `pillar.example` describe the supported pillar keys. The
certificates and SASL credentials in the test pillar are placeholders.

`postfix/files/mapping.j2` contains literal tabs on purpose (key/value
separator); salt-lint rule 203 is disabled for that file only.
