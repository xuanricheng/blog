---
title: "Introducing SynthID Text"
thumbnail: /blog/assets/synthid-text/thumbnail.png
authors:
  - user: sumedhghaisas
    org: Google DeepMind
    guest: true
  - user: sdathath
    org: Google DeepMind
    guest: true
  - user: RyanMullins
    org: Google DeepMind
    guest: true
  - user: joaogante
  - user: marcsun13
  - user: RaushanTurganbay
---

# Introducing SynthID Text

Do you find it difficult to tell if text was written by a human or generated by
AI? Being able to identify AI-generated content is essential to promoting trust
in information, and helping to address problems such as misattribution and
misinformation. Today, [Google DeepMind](https://deepmind.google/) and Hugging
Face are excited to launch
[SynthID Text](https://deepmind.google/technologies/synthid/) in Transformers
v4.46.0, releasing later today. This technology allows you to apply watermarks
to AI-generated text using a
[logits processor](https://huggingface.co/docs/transformers/v4.46.0/en/internal/generation_utils#transformers.SynthIDTextWatermarkLogitsProcessor)
for generation tasks, and detect those watermarks with a
[classifier](https://huggingface.co/docs/transformers/v4.46.0/en/internal/generation_utils#transformers.SynthIDTextWatermarkDetector).

Check out the SynthID Text
[paper in _Nature_](https://www.nature.com/articles/s41586-024-08025-4) for the
complete technical details of this algorithm, and Google’s
[Responsible GenAI Toolkit](https://ai.google.dev/responsible/docs/safeguards/synthid)
for more on how to apply SynthID Text in your products.

## How it works

The primary goal of SynthID Text is to encode a watermark into AI-generated text
in a way that helps you determine if text was generated from your LLM without
affecting how the underlying LLM works or negatively impacting generation
quality. Google DeepMind has developed a watermarking technique that uses a
pseudo-random function, called a g-function, to augment the generation process
of any LLM such that the watermark is imperceptible to humans but is visible to
a trained model. This has been implemented as a
[generation utility](https://huggingface.co/docs/transformers/v4.46.0/en/internal/generation_utils#transformers.SynthIDTextWatermarkLogitsProcessor)
that is compatible with any LLM without modification using the
`model.generate()` API, along with an
[end-to-end example](https://github.com/huggingface/transformers/tree/v4.46.0/examples/research_projects/synthid_text/detector_training.py)
of how to train detectors to recognize watermarked text. Check out the
[research paper](https://www.nature.com/articles/s41586-024-08025-4) that has
more complete details about the SynthID Text algorithm.

## Configuring a watermark

Watermarks are
[configured using a dataclass](https://huggingface.co/docs/transformers/v4.46.0/en/internal/generation_utils#transformers.SynthIDTextWatermarkingConfig)
that parameterizes the _g_-function and how it is applied in the tournament
sampling process. Each model you use should have its own watermarking
configuration that **_should be stored securely and privately_**, otherwise your
watermark may be replicable by others.

You must define two parameters in every watermarking configuration:

- The `keys` parameter is a list integers that are used to compute _g_-function
  scores across the model's vocabulary. Using 20 to 30 unique, randomly
  generated numbers is recommended to balance detectability against generation
  quality.

- The `ngram_len` parameter is used to balance robustness and detectability; the
  larger the value the more detectable the watermark will be, at the cost of
  being more brittle to changes. A good default value is 5, but it needs to be
  at least 2.

You can further configure the watermark based on your performance needs. See the
[`SynthIDTextWatermarkingConfig` class](https://huggingface.co/docs/transformers/v4.46.0/en/internal/generation_utils#transformers.SynthIDTextWatermarkingConfig)
for more information.

The [research paper](https://www.nature.com/articles/s41586-024-08025-4)
includes additional analyses of how specific configuration values affect
watermark performance.

## Applying a watermark

Applying a watermark is a straightforward change to your existing generation
calls. Once you define your configuration, pass a
`SynthIDTextWatermarkingConfig` object as the `watermarking_config=` parameter
to `model.generate()` and all generated text will carry the watermark. Check out
the [SynthID Text Space](https://huggingface.co/spaces/google/synthid-text) for
an interactive example of watermark application, and see if you can tell.

```py
from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    SynthIDTextWatermarkingConfig,
)

# Standard model and tokenizer initialization
tokenizer = AutoTokenizer.from_pretrained('repo/id')
model = AutoModelForCausalLM.from_pretrained('repo/id')

# SynthID Text configuration
watermarking_config = SynthIDTextWatermarkingConfig(
    keys=[654, 400, 836, 123, 340, 443, 597, 160, 57, ...],
    ngram_len=5,
)

# Generation with watermarking
tokenized_prompts = tokenizer(["your prompts here"])
output_sequences = model.generate(
    **tokenized_prompts,
    watermarking_config=watermarking_config,
    do_sample=True,
)
watermarked_text = tokenizer.batch_decode(output_sequences)
```

## Detecting a watermark

Watermarks are designed to be detectable by a trained classifier but
imperceptible to humans. Every watermarking configuration you use with your
models needs to have a detector trained to recognize the mark.

The basic detector training process is:

- Decide on a watermarking configuration.
- Collect a detector training set split between watermarked or not, and training
  or test, we recommend a minimum of 10k examples.
- Generate non-watermarked outputs with your model.
- Generate watermarked outputs with your model.
- Train your watermark detection classifier.
- Productionize your model with the watermarking configuration and associated detector.

A
[Bayesian detector class](https://huggingface.co/docs/transformers/v4.46.0/en/internal/generation_utils#transformers.BayesianDetectorModel)
is provided in Transformers, along with an
[end-to-end example](https://github.com/huggingface/transformers/tree/v4.46.0/examples/research_projects/synthid_text/detector_training.py)
of how to train a detector to recognize watermarked text using a specific
watermarking configuration. Models that use the same tokenizer can also share
watermarking configuration and detector, thus sharing a common watermark, so
long as the detector's training set includes examples from all models that share
a watermark.

This trained detector can be uploaded to a private HF Hub to make it accessible
across your organization. Google’s
[Responsible GenAI Toolkit](https://ai.google.dev/responsible/docs/safeguards/synthid)
has more on how to productionize SynthID Text in your products.

## Limitations

SynthID Text watermarks are robust to some transformations, such as cropping
pieces of text, modifying a few words, or mild paraphrasing, but this method
does have limitations.

- Watermark application is less effective on factual responses, as there is less
  opportunity to augment generation without decreasing accuracy.
- Detector confidence scores can be greatly reduced when an AI-generated text is
  thoroughly rewritten, or translated to another language.

SynthID Text is not built to directly stop motivated adversaries from causing
harm. However, it can make it harder to use AI-generated content for malicious
purposes, and it can be combined with other approaches to give better coverage
across content types and platforms.