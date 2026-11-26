# OpenTofu packages path omits `zh:` hashes from `SHA256SUMS`

Reproduction for [renovatebot/renovate#46517](https://github.com/renovatebot/renovate/pull/46517).

## Current behavior

For providers on `registry.opentofu.org`, Renovate builds the lock file hashes from the `packages` field of the registry download endpoint.
That field has one `zh:` hash per platform zip, but some providers' `SHA256SUMS` lists more files, for example `integrations/github` also lists `terraform-provider-github_<version>_manifest.json`.

`tofu init` records a `zh:` hash for every line of `SHA256SUMS`, while Renovate leaves out the manifest one:

```diff
     "zh:f7c02f762e4cf1de7f58bde520798491ccdd54a5bd52278d579c146d1d07d4f0",
+    "zh:fbd1fee2c9df3aa19cf8851ce134dea6e45ea01cb85695c1726670c285797e25",
   ]
```

The next `tofu init` or `tofu providers lock` adds it back, so the lock file churns between Renovate and local runs.
The zip download fallback (used for other registries) already includes it.

## Expected behavior

The Renovate lock file has the same set of `zh:` hashes as the one written by `tofu init`.

## Reproduction

- [`main.tf`](main.tf) pins `registry.opentofu.org/integrations/github` to `6.5.0`.
- [`.terraform.lock.hcl`](.terraform.lock.hcl) was written by `tofu init` (OpenTofu 1.10.6) and has 15 `zh:` hashes.
- [`renovate.json`](renovate.json) limits the update to `6.6.0`.

Renovate was run locally against this repository, once from each commit:

| PR | Renovate commit | `zh:` hashes | Matches `tofu init` |
| :-- | :-- | :-- | :-- |
| [#1](../../pull/1) | `main` at [`c63cf0d`](https://github.com/renovatebot/renovate/commit/c63cf0dcb8d0bb2c47bb5dede5d3552ff8b2a860) | 14 | No, missing the `_manifest.json` hash |
| [#2](../../pull/2) | #46517 at [`791feab`](https://github.com/renovatebot/renovate/pull/46517/commits/791feab9f42b02a4a0b53d15ca25884286158bb3) | 15 | Yes |

Both runs log `Using OpenTofu packages API for integrations/github@6.6.0`.

`tofu init` with `version = "6.6.0"` gives this `zh:` set, identical to PR #2:

```
zh:0b1b5342db6a17de7c71386704e101be7d6761569e03fb3ff1f3d4c02c32d998
zh:2fb663467fff76852126b58315d9a1a457e3b04bec51f04bf1c0ddc9dfbb3517
zh:4183e557a1dfd413dae90ca4bac37dbbe499eae5e923567371f768053f977800
zh:48b2979f88fb55cdb14b7e4c37c44e0dfbc21b7a19686ce75e339efda773c5c2
zh:5d803fb06625e0bcf83abb590d4235c117fa7f4aa2168fa3d5f686c41bc529ec
zh:6f1dd094cbab36363583cda837d7ca470bef5f8abf9b19f23e9cd8b927153498
zh:772edb5890d72b32868f9fdc0a9a1d4f4701d8e7f8acb37a7ac530d053c776e3
zh:798f443dbba6610431dcef832047f6917fb5a4e184a3a776c44e6213fb429cc6
zh:cc08dfcc387e2603f6dbaff8c236c1254185450d6cadd6bad92879fe7e7dbce9
zh:d5e2c8d7f50f91d6847ddce27b10b721bdfce99c1bbab42a68fa271337d73d63
zh:e69a0045440c706f50f84a84ff8b1df520ec9bf757de4b8f9959f2ed20c3f440
zh:efc5358573a6403cbea3a08a2fcd2407258ac083d9134c641bdcb578966d8bdf
zh:f627a255e5809ec2375f79949c79417847fa56b9e9222ea7c45a463eb663f137
zh:f7c02f762e4cf1de7f58bde520798491ccdd54a5bd52278d579c146d1d07d4f0
zh:fbd1fee2c9df3aa19cf8851ce134dea6e45ea01cb85695c1726670c285797e25
```

To compare the registry data directly:

```sh
curl -s https://registry.opentofu.org/v1/providers/integrations/github/6.6.0/download/linux/amd64 \
  | jq -r '.packages[].hashes[] | select(startswith("zh:"))' | wc -l   # 14
curl -sL https://github.com/integrations/terraform-provider-github/releases/download/v6.6.0/terraform-provider-github_6.6.0_SHA256SUMS \
  | wc -l                                                              # 15
```

## Link to the Renovate PR

https://github.com/renovatebot/renovate/pull/46517
