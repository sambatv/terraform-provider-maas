# This is a fork

`sambatv/terraform-provider-maas` is a fork of
[`canonical/terraform-provider-maas`](https://github.com/canonical/terraform-provider-maas)
that carries **one change**, and exists only until that change is upstream.

## The change

`maas_raid` exports a new read-only attribute, `virtual_device_id`.

MAAS lets a volume group sit on a RAID — the Storage tab in the web UI does it in
two clicks — but the API accepts only the ID of the **virtual block device** the
array exposes. The released provider never publishes that ID, and it cannot be
derived from anything else it does publish:

* `maas_raid`'s own `id` is the **filesystem group's** ID, which the volume group
  endpoint does not accept;
* `data "maas_machine"` filters virtual devices out deliberately
  (`if bd.Type == "physical"`);
* adopting the array as a `maas_block_device` fails inside MAAS itself, on a bare
  `assert self.size == self.filesystem_group.get_size()` in
  `VirtualBlockDevice.clean()`.

So LVM on a RAID is expressible in the UI and not in Terraform. That is the whole
of the gap, and `entity.RAID.VirtualDevice` already arrives in the read path
carrying the ID — `resourceRAIDRead` uses the same struct for `fs_type`,
`mount_point` and `mount_options`. The patch adds it to the schema and to the
state map. Two lines of behaviour; the rest of the diff is `gofmt` realigning a
map literal.

```hcl
resource "maas_volume_group" "vg0" {
  machine       = data.maas_machine.host.id
  name          = "vg0"
  block_devices = [maas_raid.md0.virtual_device_id]
}
```

Note that the array has to be **unformatted** for this: a volume group cannot be
built on a filesystem, so an `md0` that carries `fs_type`/`mount_point` is not a
candidate. `TestAccResourceMAASRAID_volumeGroup` builds the whole stack — RAID,
volume group, logical volume — and is the test that justifies the attribute;
`TestCheckResourceAttrSet` alone would only prove the field is non-empty.

## Why a fork rather than waiting

Upstream is the right home for this and the patch is written to be mergeable
as-is. The fork is what lets `las-terraform` describe a machine's storage
completely in the meantime. **When the change lands upstream, this fork goes
away** and consumers move back to `canonical/maas`.

## Versioning

Our own version line in our own namespace, so it can never collide with an
upstream release:

| ours                 | upstream base |
| -------------------- | ------------- |
| `sambatv/maas 2.10.1` | `v2.10.0`     |

`master` is left alone as a mirror of upstream, so `gh repo sync` keeps working.
All our work is on **`samba`**, which is the default branch.

## Rebasing onto a new upstream release

```sh
gh repo sync sambatv/terraform-provider-maas --source canonical/terraform-provider-maas
git fetch origin --tags
git rebase v2.11.0 samba          # or whatever the new tag is
go build ./... && go vet ./maas/... && go test ./maas/...
```

Then tag `v2.11.1` and let the release workflow run. Check first whether upstream
has merged the attribute — if it has, the rebase conflicts, and that conflict is
the signal to retire the fork.

## Releasing

Tagging `v*` runs `.github/workflows/release.yml`: goreleaser builds every
platform, signs `SHA256SUMS` with the GPG key in the repository's secrets
(`GPG_PRIVATE_KEY`, `PASSPHRASE`) and opens a **draft** release to be published by
hand. That signature is what the OpenTofu registry verifies, so without those two
secrets a tag produces nothing usable.
