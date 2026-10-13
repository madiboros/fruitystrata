# FruityStrata

I made this fork to run [UnbiasedStrata](https://github.com/robertkarl/UnbiasedStrata) on my Alienware and to keep the setup easy to rebuild. UnbiasedStrata is based on [Strata](https://github.com/Niko1221/Strata) and llama.cpp.

My main machine for this is a headless Alienware laptop with a mobile RTX 2080 Super (8 GB, Turing / `sm_75`), a 10th-generation Intel Core i7, and 64 GB of system memory. I use it to run Qwen3.8-Flash-Next. This repo holds the source and build settings. It does not include the model weights, expert pack, or MTP draft files.

## What I changed for my setup

- I set the default CUDA target to Turing (`sm_75`) for the RTX 2080 Super.
- I added an explicit CUDA driver library link needed by the native-expert CUDA code.
- I kept the upstream model-serving and API implementation. This is a hardware-focused fork, not a new inference engine.

## Build

I build this on Linux x86-64 with AVX2, a C++20 compiler, CMake 3.24 or newer, the NVIDIA driver, and a compatible CUDA toolkit.

```sh
git clone --recurse-submodules https://github.com/lolmadi/fruitystrata.git
cd fruitystrata
make build
```

The default CUDA target is `75`, which matches my RTX 2080 Super. To build for other NVIDIA GPUs, I can override it. For example:

```sh
make build CUDA_ARCHS='86;89;120'
```

The build creates `build/unbiased-strata`. It uses the pinned llama.cpp submodule and does not download model or runtime files. I can cap parallel build jobs with `JOBS=N`.

## Model files I use

I run the full Qwen3.8-Flash-Next GSQ-RCO model at IQ3_XXS quantization, with its prepared expert pack and MTP draft head. The preparation instructions and artifact details are in [UPSTREAM.md](UPSTREAM.md). I keep both model shards together because the second shard is the n-gram table the engine expects.

Before starting the server, I need paths to:

1. `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf`. I pass this first shard as the model path and keep shard 2 beside it.
2. The prepared IQ3_XXS pack directory.
3. The prepared MTP directory.

These files are large, so I keep them out of Git.

## Run it

For a local-only server, I can use the defaults, which bind to `127.0.0.1:8080`:

```sh
./build/unbiased-strata \
  --model /path/to/IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf \
  --pack /path/to/pack-iq3-xxs \
  --mtp /path/to/mtp
```

My Alienware profile uses a 131,072-token context and these engine options:

```sh
./build/unbiased-strata \
  --model /path/to/IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf \
  --pack /path/to/pack-iq3-xxs \
  --mtp /path/to/mtp \
  --ctx 131072 \
  --host <ALIENWARE_LAN_IP> --port 8081 \
  -- --kv-resident 20480 --ple-io direct
```

I have Pithagoras's `nova-psycho` profile pointed at `http://<ALIENWARE_LAN_IP>:8081/v1` with model ID `qwen3.8-flash-next-iq3_xxs`.

The LAN listener has no API key. I only bind it to a trusted private network, keep access restricted with the host firewall, and do not port-forward it to the public internet.

## API and options

- `POST /v1/chat/completions` serves OpenAI-compatible chat requests, with or without streaming, and supports tool calls.
- `GET /v1/models` reports the model ID and configured context length.
- `GET /health` is my readiness check.
- `--ctx N` sets the context length. The default is 32,768. I can also set `--threads N`, `--host`, and `--port`.
- Reasoning is returned as `reasoning_content`. I can turn it off with `reasoning_effort: "none"`.
- Image input needs the model's vision encoder through `--mmproj`. I have not included that in my current voice-free coding setup.
- Engine-specific options go after `--`.

## My test results

I have run a full 128K context successfully on the Alienware, at roughly 18 to 20 tokens per second. That is my real-use result, not an independently repeated benchmark.

I also recorded these separate smoke-test measurements on an earlier Alienware lab build:

| Test | Result |
| --- | ---: |
| Full 128K context run | roughly 18 to 20 tokens/s |
| Prefill for a 32,420-token prompt | 146.8 tokens/s |
| Decode for a separate 125-token reply | 23.8 tokens/s |

The prefill and short-reply numbers came from separate requests. They are not directly comparable to each other or to the full-context run. Speed depends on the prompt, cache state, and settings, so I treat these as observations rather than guarantees.

## Tests

I can run the protocol and child-process tests without loading a model or using CUDA:

```sh
make test-frontend
```

After building, I can check the chat template and tool-call parsing against the first model shard:

```sh
make test-chat-parser MODEL=/path/to/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf
```

## Upstream and license

I track upstream UnbiasedStrata and keep my local changes focused on my hardware and build. [UPSTREAM.md](UPSTREAM.md) has the project history and model preparation notes.

The inherited project is MIT-licensed. Model files and other third-party assets have their own terms. See [LICENSE](LICENSE).
