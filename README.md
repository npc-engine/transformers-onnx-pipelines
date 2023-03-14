Edge transformers is a Rust implementation of Huggingface's 
[pipelines](https://huggingface.co/docs/transformers/main/en/quicktour#pipeline) 
based on ONNX Runtime backend.

## Features

- C# and C wrappers (proper C++ wrapper is planned)
- Text to output interface abstracting over tokenizers.
- Multiple ORT providers support:
  - CPU
  - CUDA (Requires building onnxruntime for with CUDA provider)
  - DirectML 
  - More planned

## Tasks implemented

| Model export feature/task | Class name                                          |
|---------------------------|-----------------------------------------------------|  
| causal-lm                 | ConditionalGenerationPipeline                       |  
| causal-lm-with-past	    | ConditionalGenerationPipelineWithPKVs               |
| default	                | EmbeddingPipeline                                   |
| seq2seq-lm                | Seq2SeqGenerationPipeline or OptimumSeq2SeqPipeline |
| seq2seq-lm-with-past	    |  OptimumSeq2SeqPipelineWithPKVs                     |
| sequence-classification	| SequenceClassificationPipeline                      |
| token-classification      | TokenClassificationPipeline                         |  



## Usage

Your linker must be able to find onnxruntime.dll and edge-transformers.dll (or *.so on Linux).
You can find C and C# wrappers in `c` and `csharp` folders respectively.

### C#

Documentation is WIP, refer to Rust documentation for now.

```csharp
using EdgeTransformers;

...
    var env = EdgeTransformers.Environment.New();
    var condPipelinePkv = ConditionalGenerationPipelineWithPKVs.FromPretrained(
        env.Context, "optimum/gpt2", DeviceFFI.DML, GraphOptimizationLevelFFI.All);

    string output = condPipelinePkv.GenerateTopkSampling("Hello world", 10, 5, 0.5f);
...
```

Batch processing is supported, but is a bit unintuitive and requires StringBatch class.

```csharp
using EdgeTransformers;
  ...
        var env = EdgeTransformers.Environment.New();
        var condPipelinePkv = ConditionalGenerationPipelineWithPKVs.FromPretrained(
            env.Context, "optimum/gpt2", DeviceFFI.DML, GraphOptimizationLevelFFI.All);
        var string_batch = StringBatch.New();
        string_batch.Add("Hello world");
        string_batch.Add("Hello world");

        var o_batch_pkv = condPipelinePkv.GenerateRandomSamplingBatch(string_batch.Context, 10, 0.5f);

        Debug.LogFormat("Cond generation output 0: {0} 1: {1}", o_batch_pkv[0].ascii_string, o_batch_pkv[1].ascii_string);
  ...
```


### C

TODO

### Rust

```csharp
use std::fs;
use onnxruntime::environment::Environment;
use onnxruntime::{GraphOptimizationLevel, LoggingLevel};
use edge_transformers::{ConditionalGenerationPipelineWithPKVs, TopKSampler, Device};

let environment = Environment::builder()
   .with_name("test")
   .with_log_level(LoggingLevel::Verbose)
   .build()
   .unwrap();

let sampler = TopKSampler::new(50, 0.9);
let pipeline = ConditionalGenerationPipelineWithPKVs::from_pretrained(
    &environment,
    "optimum/gpt2".to_string(),
    Device::CPU,
    GraphOptimizationLevel::All,
).unwrap();

let input = "This is a test";

println!("{}", pipeline.generate(input, 10, &sampler).unwrap());
```

## Roadmap

- [x] C# wrapper
- [x] C wrapper
- [x] Migrate to [ort](https://github.com/pykeio/ort) crate
- [ ] Proper CI/CD to test and build for more execution providers
- [ ] C++ wrapper
- [ ] More pipelines (e.g. extractive QA, ASR, etc.)
- [ ] Better huggingface config.json parsing

## Building ONNX Runtime fork

While proper CI/CD for [ONNX Runtime fork](https://github.com/npc-engine/onnxruntime-rs) 
that is used in this project is not set up, you have to build correct 
[onnxruntime](https://onnxruntime.ai/) dlls and replace them manually:

- Clone ONNX Runtime fork
```bash
git clone https://github.com/npc-engine/onnxruntime-rs
```
- Replace `onnxruntime.dll` and `onnxruntime.lib` with the ones built for your platform/execution provider.
```bash
cp $PATH_TO_ONNXRUNTIME_DLL/onnxruntime.* onnxruntime-rs/onnxruntime-sys/lib/onnxruntime.dll
```
- Point `edge-transformers` to your local `onnxruntime-rs` folder by modifying `Cargo.toml`.
```cargo
onnxruntime = { path = "<path to repo>/onnxruntime-rs" }
onnxruntime-sys = { path = "<path to repo>/onnxruntime-rs" }
```
- Build `edge-transformers` with correct feature flags (add if needed)
- (Optional) add new provider to `onnxruntime-rs/onnxruntime/src/session.rs`.  
You can find correct OrtSessionOptionsAppendExecutionProvider_{} function name in 
[onnxruntime repo](https://github.com/microsoft/onnxruntime)
```csharp
e.g.
    /// Set the session to use cuda if feature cuda and not tensorrt
    #[cfg(feature = "your_provider_feature")]
    pub fn use_my_provider(self, device_id: i32) -> Result<SessionBuilder<'a>> {
        unsafe {
            sys::OrtSessionOptionsAppendExecutionProvider_YOURPROVIDER(self.session_options_ptr, device_id);
        }
        Ok(self)
    }
```

## Testing 

Tests require to be running on a single thread at least for the first time. 
The reason is that they use *::from_pretrained function that downloads data 
from Huggingface Hub and some tests rely on the same files being downloaded.
Second time they can run in parallel because they use cached files.

e.g. First time command:
```bash
cargo test -- --test-threads=1
```

e.g. Second time:
```bash
cargo test 
```
