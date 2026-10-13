# fruitystrata

fruitystrata is a hardware-focused deployment fork of [UnbiasedStrata](https://github.com/robertkarl/UnbiasedStrata), itself based on [Strata](https://github.com/Niko1221/Strata) and llama.cpp. This configuration targets a dedicated Alienware laptop with a mobile RTX 2080 Super (8 GB, Turing / `sm_75`), a 10th-generation Intel Core i7, and 64 GB of RAM.

The repository contains source and build configuration, not model weights, the prepared expert pack, or the MTP draft files. Those large artifacts must be prepared separately and are not uploaded here.

## what this fork changes

- Builds for Turing (`sm_75`) by default, matching the Alienware's 2080 Super GPU.
- Links CUDA's driver library explicitly for the native-expert CUDA code.
- Keeps the upstream project’s model-serving and API implementation; this is a machine-specific configuration, not a rewritten inference engine.

## Build

On Linux x86-64 with AVX2, install a C++20 compiler, CMake 3.24 or newer, the NVIDIA driver, and a compatible CUDA toolkit. Then:

```sh
git clone --recurse-submodules https://github.com/lolmadi/fruitystrata.git
cd fruitystrata
make build
```

The default CUDA target is `75`. To build for other NVIDIA architectures, override it, for example:

```sh
make build CUDA_ARCHS='86;89;120'
```

The resulting executable is `build/unbiased-strata`. The build uses the pinned llama.cpp submodule and does not download models or other runtime assets. `JOBS=N` can cap parallel build jobs.

## Model files

The tested configuration uses the full Qwen3.8-Flash-Next GSQ-RCO model at IQ3_XXS quantization, with its prepared expert pack and MTP draft head. See [UPSTREAM.md](UPSTREAM.md) for upstream preparation instructions and artifact details. Keep the two model shards together; the second shard is the n-gram table expected by the engine.

You need paths to:

1. `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf` (the model argument; shard 2 must be beside it)
2. The prepared IQ3_XXS pack directory
3. The prepared MTP directory

These files are large and intentionally excluded from Git.

## Run

Basic local-only example (the server defaults to `127.0.0.1:8080`):

```sh
./build/unbiased-strata \
  --model /path/to/IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf \
  --pack /path/to/pack-iq3-xxs \
  --mtp /path/to/mtp
```

The Alienware’s tested profile uses a 131,072-token configured context and the following engine options:

```sh
./build/unbiased-strata \
  --model /path/to/IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf \
  --pack /path/to/pack-iq3-xxs \
  --mtp /path/to/mtp \
  --ctx 131072 \
  --host <ALIENWARE_LAN_IP> --port 8081 \
  -- --kv-resident 20480 --ple-io direct
```

The current Pithagoras `nova-psycho` profile points to `http://<ALIENWARE_LAN_IP>:8081/v1` and uses model ID `qwen3.8-flash-next-iq3_xxs`. The LAN listener has no API authentication: bind only to a trusted private interface, restrict access with the host firewall, and do not port-forward or expose it to the public internet.

### Server interface

- `POST /v1/chat/completions` — OpenAI-compatible chat, streaming or non-streaming; supports tool calls.
- `GET /v1/models` — reports the model ID and configured context length.
- `GET /health` — readiness check.
- `--ctx N` sets the context length (default 32,768); `--threads N`, `--host`, and `--port` are also available.
- Reasoning is returned as `reasoning_content`; `reasoning_effort: "none"` disables it.
- Optional image input requires the model’s vision encoder via `--mmproj`; see upstream documentation before enabling it.
- Engine-specific options go after `--`.

## Validation and performance notes

The deployed Alienware profile has started successfully with the 131,072-token context setting, the IQ3_XXS pack, and MTP enabled. The owner reports successfully testing a full 128K context at roughly 18–20 tokens/s. The separate smoke-test measurements below came from an earlier Alienware lab build, not a repeatable benchmark of every commit in this repository:

| Test | Result |
| --- | ---: |
| Full 128K context run (owner-reported) | roughly 18–20 tokens/s |
| Prompt prefill, 32,420-token prompt (separate smoke test) | 146.8 tokens/s |
| Decode, separate 125-token reply (separate smoke test) | 23.8 tokens/s |

These figures come from different requests and workloads, so they are not an apples-to-apples benchmark. The 18–20 tokens/s result is an owner-reported successful full-context test; the prefill and short-reply numbers are single-run measurements from the earlier lab build. Re-test after changing the model, context settings, or build before treating the figures as a performance guarantee.

## Tests

The protocol and child-process tests do not require a model or CUDA:

```sh
make test-frontend
```

After a full build, the chat-template/parser test can be run against the first model shard:

```sh
make test-chat-parser MODEL=/path/to/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf
```

## Lineage and license

FruityStrata tracks upstream UnbiasedStrata and carries a small set of local hardware/build changes. See [UPSTREAM.md](UPSTREAM.md) for provenance and upstream preparation notes. The inherited project is MIT-licensed; model and other third-party assets have their own terms. See [LICENSE](LICENSE).
