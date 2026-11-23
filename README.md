# SavantAPI

SavantAPI is the API plane for Outpost 2 implementations.

An implementation is one version of the game. The original GOG executable, that executable with the OPU 1.4.1 patches, the classic rebuild (OP2CL), the community rebuild (OP2CE), and later ports each keep their own code. This repository defines the APIs those implementations implement, so the same play can be run on each of them and the results compared.

The plane is a set of surfaces. A surface is one shared API. New surfaces are added here as they become part of the contract, and each surface is versioned.

## Surfaces

- **Savant API.** The Savant surface itself.
- **AI agents.** Agents written against the Savant API.
- **Harness.** The enveloping definitions for a play. A gameplay names the rules, the map, the seed, and the orders. An implementation provides the calls that start the play, advance it, issue orders, and return a recording. Comparing two implementations is comparing two recordings of the same gameplay.

The calls and schemas for these surfaces land in this repository as they are defined.

## Adapters

An adapter is an implementation's side of a surface.

[OP2TestHarness](https://github.com/TethysGame/OP2TestHarness) is the Windows adapter for the original executable. It loads inside that process and implements the harness surface for it. A port implements the same surface in its own runtime and depends on this repository.

## License

MIT. See [LICENSE](LICENSE).
