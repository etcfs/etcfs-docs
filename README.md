# etcfs-docs

Documentation site for the [EtcFS](https://github.com/etcfs/etcfs) project,
built with [MkDocs](https://www.mkdocs.org/) + Material and published to
GitHub Pages at <https://etcfs.github.io/etcfs-docs/>.

Split out of the main [etcfs/etcfs](https://github.com/etcfs/etcfs)
repository's `docs/` directory; commit history for the docs was preserved
in the split. It covers the whole project, including the pieces that now
live in their own repositories:

- [etcfs/etcfs](https://github.com/etcfs/etcfs) — core filesystem
- [etcfs-csi-driver](https://github.com/etcfs/etcfs-csi-driver) — Kubernetes CSI driver
- [etcfs-terraform-modules](https://github.com/etcfs/etcfs-terraform-modules) — Terraform
- [etcfs-tla-specs](https://github.com/etcfs/etcfs-tla-specs) — TLA+ protocol models

## Building locally

```bash
pip install -r requirements-docs.txt
mkdocs serve
```
