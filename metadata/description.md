# Forteller — description

A push-only, vault-first CLI for tending the relationship between a configuration vault and the machines it describes — declarative intent pushed outward from a single source of truth.

Most infrastructure-as-code tooling assumes agents on every machine, YAML files multiplying past anyone's ability to keep them coherent, and a model of configuration as an ongoing conversation with the fleet. Forteller refuses the model. There is one source of truth — the vault — and intent flows outward, not inward: no agents, no state drift. The deeper argument is about the dominant assumption that complexity is necessary; for the homelab practitioner it is not, and for most production environments at most scales it is not either, and Forteller demonstrates the simpler alternative. It is planned as a plugin within Pālana, the operational workbench, and belongs to the Kṣetra-Ops suite — the same ethos of transparency and responsibility without ceremony, expressed as configuration.

A command-line tool, planned as an integrated capability within Pālana; pre-build.
