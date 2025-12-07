---
layout: distill
title: Why vlms waste their vision
description: Despite the robustness of standalone vision encoders, they often collapse to near-chance performance within Vision Language Models (VLMs) by ignoring visual data in favor of language priors. We investigate this paradox by reconciling conflicting theoretical and empirical literature through the lens of attention budgets and information exchange rates. Ultimately, we propose a new mental model that explains why standard multimodal fusion fails and how to restore effective integration.
date: 2026-04-27
future: true
htmlwidgets: true
hidden: true

# Mermaid diagrams
mermaid:
  enabled: true
  zoomable: true

# Anonymize when submitting
authors:
  - name: Anonymous

# must be the exact same name as your blogpost
bibliography: 2026-04-27-why-vlms-waste-their-vision.bib

# Add a table of contents to your post.
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly.
#   - please use this format rather than manually creating a markdown table of contents.
toc:
  - name: Background
  - name: The image-text exchange rate
  - name: Measuring Modality Usage
  - subsections:
      - name: Dependency Test (Minimal Exchange-Rate Criterion)
      - name: Counterfactual Override Test (Text Dominance Detection)
      - name: Cross-Modal Consistency Test (Fusion Quality Measurement)
      - name: Perturbation Sensitivity Test (Attention Budget Probe)
      - name: Information Contribution Test (Exchange Rate in Bits)
      - name: Uncertainty Reduction Test (Decision-Level Synergy)
  - name: Conclusion 

# Below is an example of injecting additional post-specific styles.
# This is used in the 'Layouts' section of this post.
# If you use this post as a template, delete this _styles block.
_styles: >
  .fake-img {
    background: #bbb;
    border: 1px solid rgba(0, 0, 0, 0.1);
    box-shadow: 0 0px 4px rgba(0, 0, 0, 0.1);
    margin-bottom: 12px;
  }
  .fake-img p {
    font-family: monospace;
    color: white;
    text-align: left;
    margin: 12px 0;
    text-align: center;
    font-size: 16px;
  }
---
## Background

To understand why VLMs behave so strangely, we need both theory that specifies what should happen when we combine modalities, and empirical evidence that reveals what actually happens in current systems. A growing body of work now makes it clear that these two perspectives are in tension.

On the theoretical side, Better Together <d-cite key="gupta2025bettertogetherleveragingunpaired"></d-cite>gives a strikingly optimistic view of multimodality. They show that adding an auxiliary modality can only increase or preserve the Fisher information<d-footnote>
Fisher Information measures how much a model’s observations constrain its belief about the correct answer.
Informally, it quantifies how sharply the model’s output distribution changes in response to changes in the underlying signal. More Fisher information means less uncertainty and more decisive predictions.
</d-footnote> available to a model, and empirically demonstrate that unpaired text, audio, or images consistently improve downstream performance of unimodal encoders. In this picture, modalities behave like complementary measurements of the same underlying signal: more channels should never hurt, and multimodal pretraining becomes a general recipe for building stronger unimodal models. 

The empirical story from large VLMs is far less rosy. Hidden in Plain Sight<d-cite key="fu2025hidden"></d-cite> finds that many state-of-the-art VLMs barely use their visual representations at all: scrambling, masking, or even replacing the image often leads to negligible changes in the output across a range of downstream tasks. Related work in vision–language reasoning and abstract shape recognition echoes the same theme: models that appear multimodal by design often solve benchmarks using predominantly linguistic shortcuts with minimal dependence on pixels<d-cite key="hemmat2024hiddenplainsightevaluating"></d-cite>.  This mirrors older observations from multimodal surveys and VQA: text only baselines can rival full models, and spurious language priors frequently dominate over visual evidence<d-cite key="baltrušaitis2017multimodalmachinelearningsurvey"></d-cite>. These findings suggest that the optimistic guarantees of theory rely on assumptions that real systems routinely violate: that modalities are fused without distortion, that their contributions are balanced, and that the model has enough capacity to process them jointly rather than letting one overwhelm the other.

Taken together, this body of work exposes a conceptual gap. We have strong theoretical results predicting monotonic gains from adding modalities, and equally strong empirical evidence that large models often ignore, underweigh or sometimes even misuse those modalities. We lack principles and metrics for characterizing how information from each modality is integrated, and how much influence each modality actually has on the final prediction. We aim to address this gap by viewing multimodal models through the lens of an information exchange rate i.e how many “bits” of effective evidence an image contributes relative to text and a finite attention budget which is how much representational and computational capacity the model allocates to each modality as context grows. Having these tools, we can begin to diagnose not just whether multimodal fusion fails, but why it fails and when it silently collapses into text-only behavior despite having a powerful vision backbone.

## The image-text exchange rate
If multimodality is supposed to help, how much does each modality contribute? And can we say anything quantitative about when one modality should override another? A useful way to think about this is through what we’ll call the information exchange rate: how many bits of useful information does an image contribute relative to text, and how does a VLM decide which to trust when the two compete. 
{% include figure.liquid path="assets/img/2026-04-27-why-vlms-waste-their-vision/diagram_info_exchange_rate.png" class="img-fluid" caption="Figure 1: A mental model for the image-text exchange rate presented with the analogy of a balance scale. Image generated with Nano banana pro." %}


The optimistic view, grounded in theory like Better Together<d-cite key="gupta2025bettertogetherleveragingunpaired"></d-cite>, is that modalities should behave like complementary information channels. Under this lens, an image and a sentence each provide different perspective of the same world. So combining them should give the model more Fisher information and in turn, a better estimate of the target. In an ideal multimodal system, adding auxiliary text is like adding more windows to the same reality, in other words it should never make the model worse. However prior work suggest a more complicated reality. Studies on modality dominance<d-cite key="baltrušaitis2017multimodalmachinelearningsurvey"></d-cite>, linguistic priors in VQA<d-cite key="agrawal2018dontjustassumelook"></d-cite>, and error analysis of image–to-text benchmarks<d-cite key="jabri2016revisitingvisualquestionanswering"></d-cite> consistently show that models learn uneven “exchange rates” between modalities. In many architectures, textual signals are treated as cheap and abundant, while visual signals though rich are compressed into a handful of tokens and passed through shallow fusion layers. The result is that text often overwhelms vision, even when the image provides the decisive evidence for more complex tasks.

This perspective helps reconcile the tension between theory and practice. The theory assumes each modality contributes information cleanly; the empirical literature shows that models frequently assign skewed exchange rates, making one modality effectively “expensive” and the other “free". When this happens, adding more text doesn’t just fail to help it actively distorts the model’s internal representation, drowning out the visual evidence entirely.

## Measuring Modality Usage

Up to this point, we’ve framed multimodal behavior in terms of exchange rates: how much effective evidence an image contributes relative to text. But an exchange rate does not exist in a vacuum. In any real system, information must be processed through a finite computational pathway and in transformer based VLMs, that pathway is governed by attention.

This is where the notion of an attention budget becomes essential. A VLM does not have unlimited capacity to process every modality equally. Instead, it allocates a finite amount of representational and computational capacity across tokens. As context grows, this budget must be divided implicitly among text tokens and visual tokens. When one modality is abundant and native to the architecture (text) and the other is sparse and compressed (images), this allocation can become highly skewed.

Seen this way, the information exchange rate between image and text is enforced mechanically by attention. A modality whose tokens consistently receive less attention will have less opportunity to influence intermediate representations and, ultimately, the model’s predictions. In other words, a modality’s effective exchange rate is bounded by how much attention budget it receives.

We propose that the missing link is that modern VLMs may violate the implicit assumptions of multimodal theory not at the level of representation quality, but at the level of capacity allocation. Visual information may exist, but never meaningfully participate in the computation that produces the output. This brings us to a natural question:
<blockquote>
How can we tell whether a model is actually spending attention budget on vision, and whether the image has a meaningful exchange rate relative to text?
</blockquote>

Answering this does not require new benchmarks or training regimes. It requires is a way to probe whether visual information changes the model’s internal computation and output distribution in measurable ways. Below, we outline a small set of diagnostic tests as ways of operationalizing exchange rates and attention budgets in real models.

Broadly, within this framework, a model “uses” an image only if visual input meaningfully participates in the computation under a finite attention budget. Concretely:

A fair exchange rate means the image shifts predictions or reduces uncertainty relative to text alone.

An attention budget failure means that as textual context grows, visual tokens lose influence even when they are informative.

A fusion bottleneck means that image signals never couple with language representations in a way that affects downstream reasoning.

The tests below should be read as different lenses on the same underlying phenomenon: whether vision is allowed to compete for attention and influence on equal terms with text.

## Dependency Test (Minimal Exchange-Rate Criterion)

{% include figure.liquid path="assets/img/2026-04-27-why-vlms-waste-their-vision/dependency_failure.jpeg" class="img-fluid" caption="Figure 2: Visual representation of a model's functional dependence failure where changing the image does not alter the model's output logits. Image generated with Nano banana pro." %}

The most basic requirement for visual usage is **functional dependence**:

$$

f(x_{\text{img}}, x_{\text{text}}) \neq f(x'_{\text{img}}, x'_{\text{text}})
$$

for $(x_{\text{img}} \neq x'_{\text{img}})$.

Here:
- $f(\cdot)$ denotes the model’s forward computation (e.g., the output logits),
- $x_{\text{img}}$ and $x'_{\text{img}}$ denote two different images, and
- $x_{\text{text}}$ denotes the textual input, held fixed.


By *functional dependence*, we mean that the model’s output must change as a function of the image input—holding all other inputs constant. If varying the image does not alter the output, then the image does not exert a causal influence on the computation, regardless of how rich its internal representation may be. The effective exchange rate of vision is zero, no amount of visual information survives the fusion pathway. This is the extreme failure mode documented in *Hidden in Plain Sight*<d-cite key="fu2025hidden"></d-cite>, where image scrambling or replacement leaves predictions almost unchanged.

From the attention-budget perspective, this corresponds to a system where visual tokens receive so little attention that their contribution is effectively erased before reaching the decision layers.


## Counterfactual Override Test (Text Dominance Detection)

{% include figure.liquid path="assets/img/2026-04-27-why-vlms-waste-their-vision/coounterfactual_failure.jpeg" class="img-fluid" caption="Figure 3: Counterfactual Override test scenario showing the tension between text-implied class and vision implied class" %}
In less extreme cases, the model may attend to vision weakly, but not strongly enough to override text when the two disagree. Let $c_i$ denote the class implied by the image and $c_t$ denote the class suggested by the text. If the system is genuinely using visual information, we expect:

$$

P(c_i \mid x_{\text{img}}, x_{\text{text}}) > P(c_t \mid x_{\text{img}}, x_{\text{text}})
$$


Here, $P(c \mid x_{\text{img}}, x_{\text{text}})$ denotes the model’s predicted probability for candidate output $c$ given both visual input $x_{\text{img}}$ and textual input $x_{\text{text}}$.

Failure here signals that the **exchange rate skews heavily toward text**, a phenomenon repeatedly observed in VQA models<d-cite key="goyal2017makingvvqamatter"></d-cite><d-cite key="agrawal2018dontjustassumelook"></d-cite>. Even large VLMs still display this “linguistic gravity” i.e text drags predictions toward strong priors regardless of what the pixels say.

## Cross-Modal Consistency Test (Fusion Quality Measurement)

Even if vision sometimes shifts outputs, it may still fail to interact with language representations in a meaningful way. If a model truly integrates modalities, image and text representations should be informationally entangled, rather than processed in isolation. Let $h_{\text{img}}$ and $h_{\text{text}}$ denote the internal representations derived from the image and text, respectively, and let $y$ denote the target prediction.  
We use $I$ to denote *conditional mutual information*, which measures how much knowing one representation reduces uncertainty about the other given the task outcome.

Formally, for effective multimodal fusion:

$$

I(h_{\text{img}} ; h_{\text{text}} \mid y) > 0.
$$


Low conditional mutual information indicates late-fusion collapse: modalities are processed independently and combined only superficially. In exchange-rate terms, this is a broken market i.e the modalities never negotiate shared evidence. In attention-budget terms, it suggests that attention flows remain separated across modality-specific pathways.

## Perturbation Sensitivity Test (Attention Budget Probe)

Attention budgets are not directly visible, but they leave fingerprints. If vision receives meaningful attention, then small, structured perturbations to the image should affect the output. Let $f(\cdot)$ denote the model’s forward computation, $x_{\text{img}}$ the original image input, and $\delta$ a small, semantically meaningful image perturbation (e.g., masking, cropping, or color shift).  
We define the sensitivity of the output to visual perturbations as:

$$

\Delta f = \lVert f(x_{\text{img}}) - f(x_{\text{img}} + \delta) \rVert > 0.
$$


Here, $\lVert \cdot \rVert$ denotes a suitable norm measuring change in the model’s output (e.g., logit distance or probability shift).
When outputs are insensitive to such perturbations, it suggests that the model allocates little effective capacity to processing visual tokens. This provides a behavioral proxy for attention-budget collapse and connects naturally to robustness and saliency analyses with the key distinction that the focus here is modality-level influence, not pixel-level explanation.

## Information Contribution Test (Exchange Rate in Bits)

From the perspective of theoretical works like Better Together<d-cite key="gupta2025bettertogetherleveragingunpaired"></d-cite>, adding a modality should never increase uncertainty. Let $\(Y\)$ denote the target prediction (e.g., a class label or answer), and let  
$\(x_{\text{text}}\)$ and $\(x_{\text{img}}\)$ denote the textual and visual inputs, respectively.  
We use $\(H(\cdot)\)$ to denote *entropy*, which measures uncertainty in the model’s predictive distribution.

For a well-functioning fusion mechanism:

$$

H(Y \mid x_{\text{text}}, x_{\text{img}}) < H(Y \mid x_{\text{text}}).
$$


When adding an image (or additional text) increases entropy or reduces accuracy, this indicates negative exchange rates, a regime where the modality injects noise rather than signal. This directly violates the guarantees assumed in works like Better Together<d-cite key="gupta2025bettertogetherleveragingunpaired"></d-cite> and exposes where theory and practice diverge.

## Uncertainty Reduction Test (Decision-Level Synergy)

Finally, even when accuracy remains unchanged, visual information should make the model more certain about its prediction rather than less.

Let $c$ index candidate outputs (e.g., class labels or answer choices), and let  
$P(c \mid x_{\text{text}}, x_{\text{img}})$ denote the model’s predicted probability for output $c$ given both the textual input $x_{\text{text}}$ and the visual input $x_{\text{img}}$.  
We define the model’s confidence under multimodal input as:

$$

C_{\text{img+text}}
\;=\;
\max_{c} P(c \mid x_{\text{text}}, x_{\text{img}}).
$$

A model can be said to *use* vision if adding the image does not reduce this confidence, i.e.,

$$

C_{\text{img+text}}
\;\ge\;
C_{\text{text-only}},
$$

consistently across samples.

Here, $C$ represents the model’s confidence in its most likely output.  
If confidence instead decreases or the predictive distribution becomes more diffuse when vision is added, the model is operating with a **malformed exchange rate**, a regime where visual input increases epistemic uncertainty instead of reducing it.  
From an attention-budget perspective, this suggests that capacity is being spent on conflicting or weakly integrated visual cues rather than sharpening the model’s belief.


## Conclusion
The paradox of "blind" Vision Language Models is not merely a failure of training data or encoder quality; it is a structural failure of resource allocation. As we have explored, the optimistic guarantees of works like Better Together crash against the harsh reality of Hidden in Plain Sight<d-cite key="fu2025hidden"></d-cite> because current architectures lack the mechanisms to enforce a fair information exchange rate.

By viewing multimodal fusion through the lens of attention budgets, we can see that text and vision do not simply cooperate but compete. In this internal economy, language priors are often "cheap" and abundant, while visual evidence is "expensive" to process and easy to discard. When the exchange rate is skewed, models inevitably learn to bankrupt their visual pathways, collapsing into text-only reasoning even while retaining powerful vision backbones.

The diagnostic tests proposed here, ranging from functional dependence to perturbation sensitivity, offer a way to move beyond deceptive top-line metrics. They allow us to ask not just if a model got the right answer, but how it arrived there. Ultimately, solving this paradox requires more than just scaling up. It demands a shift in design philosophy. We must move away from architectures that assume fusion just works, and toward systems that explicitly manage the attention economy. Future work must focus on enforcing a non-zero exchange rate for vision, ensuring that pixels are not just present in the input, but are solvent in the computation. Only then can we build VLMs that don't just look, but actually see.