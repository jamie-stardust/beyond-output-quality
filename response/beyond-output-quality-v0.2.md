# Beyond Output Quality: Structural Guarantees, Authority, and Auditability in ParliamentRAG

**Research response · Version 0.2 · 22 August 2026**\
jamie albert · jamiestardust@icloud.com

*Response to Tritella, Pozzi, and Palmonari, "Who Speaks Matters: Authority-Aware Multi-View RAG over Italian Parliamentary Proceedings" (arXiv:2608.13410)*

<style>
.dbox { margin: 1.1em 2.5em; padding: 0.65em 1.1em; border-left: 3px solid #2a7f7f; background: #f5f8f8; text-align: center; font-style: italic; }
.dbox.hl { font-style: normal; font-weight: bold; background: #eef4f4; }
mjx-container[display="true"] { margin: 1.1em 0 !important; }
.keep { break-inside: avoid; page-break-inside: avoid; }
</style>

## Introduction

*Who Speaks Matters* presents ParliamentRAG, a retrieval-augmented generation architecture for parliamentary proceedings that explicitly attempts to preserve properties that ordinary semantic retrieval and monolithic generation do not reliably protect: political-group representation, topic-dependent speaker authority, and quotation fidelity.

I find the paper compelling precisely because it treats some desired system behaviors as architectural concerns rather than prompt-level aspirations. ParliamentRAG does not merely instruct a language model to represent multiple political perspectives or cite faithfully. It decomposes retrieval and generation into stages, introduces an explicit authority model, selects evidence across parliamentary groups, generates group-specific perspectives, integrates those perspectives, and uses deterministic mechanisms to preserve quotations.

::: keep
This suggests a broader design principle:

$$\text{desired property} \;\to\; \text{architectural mechanism}$$

rather than merely:

$$\text{desired property} \;\to\; \text{model instruction}$$
:::

The paper demonstrates the practical value of this distinction. However, I think its architecture raises several questions that its evaluation does not yet pursue. In particular, ParliamentRAG makes explicit decisions about authority and evidence selection that can potentially be inspected, challenged, reproduced, or altered. Its comparison with NotebookLM — automated checks and expert preferences alike — evaluates properties of the resulting answers. This leaves largely unexplored a different and potentially consequential class of properties: provenance, auditability, sensitivity to policy choices, and the preservation of the intermediate structure from which an answer was produced.

My interest, therefore, is less in whether ParliamentRAG "beats" NotebookLM than in what ParliamentRAG makes possible to know about a generated answer that a monolithic generation pipeline does not.

## Authority Is Explicit — but Explicit Does Not Mean Neutral

::: keep
One of ParliamentRAG's most interesting mechanisms is its query-dependent authority model. Speaker authority is represented as a weighted combination of heterogeneous signals:

$$A(s, q) = \sum_i w_i\, c_i(s, q)$$

where the components include signals derived from professional background, education, committee membership, institutional role, legislative activity, and speech interventions.
:::

Making these components and their weights explicit is valuable. A system designer can inspect them, explain their intended purpose, and alter them. This is substantially preferable to allowing an opaque model to infer an undefined notion of "expertise."

::: keep
At the same time, the weighted-sum formulation does more than identify which characteristics matter. It establishes a policy for how those characteristics may compensate for one another. If authority is represented as

$$A(s, q) = w_1 c_1 + w_2 c_2 + \cdots + w_n c_n$$

then sufficiently high values in one component can offset low values in another. The representation therefore assumes not merely that the dimensions have differing importance, but that they are commensurable enough to be exchanged according to fixed rates. That may be appropriate. But it is a substantive modeling decision.
:::

::: keep
An alternative representation would preserve the authority components as a vector:

$$\mathbf{A}(s, q) = \big(c_1(s, q),\ c_2(s, q),\ \ldots,\ c_n(s, q)\big)$$
:::

::: keep
A separate policy could then determine what kinds of authority are required for a particular use. For example, some dimensions might act as thresholds:

$$c_i(s, q) \ge \tau_i$$

rather than as compensable contributions to a single score. A speaker who fails a required evidentiary or institutional criterion could then be treated differently regardless of strength elsewhere. This would separate two questions that a scalar ranking tends to combine:
:::

::: keep
<div class="dbox">Is this candidate admissible?</div>

and

<div class="dbox">Among admissible candidates, which do we prefer?</div>
:::

The expert-assigned weights in ParliamentRAG are therefore interesting not because expert judgment makes the system arbitrary, but because the judgment has been made sufficiently explicit that its consequences become available for study. That invites experiments the present paper does not perform. The authors acknowledge as much in their Limitations: "the study compares the complete system against a strong external baseline without internal ablation experiments, leaving the contribution of architectural components to future work."

How stable is expert selection under plausible changes to the authority weights? Which components most frequently determine the selected representative? Where do small changes in policy produce discontinuous changes in speaker selection? Are there candidates that are incomparable across dimensions but become totally ordered only because the architecture requires a scalar?

A sensitivity analysis of this kind would tell us something different from whether experts find the final response useful. It would characterize the decision surface produced by the authority policy itself.

## The NotebookLM Comparison Stops Too Early

The paper's comparison between ParliamentRAG and NotebookLM is particularly interesting because the systems differ architecturally, not merely in model capability.

::: keep
ParliamentRAG decomposes the production of an answer into identifiable stages. At a high level, we can think of its execution as something like:

$$q \to q^{\prime} \to R \to A \to E \to G \to I \to C \to Y$$

where $q$ is the query, $q^{\prime}$ its rewritten form (the paper's query-rewriting stage), $R$ retrieved evidence, $A$ authority evaluation, $E$ selected evidence and experts, $G$ per-group generation, $I$ integration, $C$ citation or quotation resolution, and $Y$ the final answer.
:::

::: keep
NotebookLM is intentionally much more monolithic from the evaluator's perspective:

$$(q,\ \text{context},\ \text{instructions}) \to Y$$
:::

In the reported protocol, even that context slot was filled under the system under test: the paper discloses that NotebookLM's curated evidence — roughly 150 relevant chunks mixed with an equal number of distractors — was selected by ParliamentRAG's own retrieval pipeline, authority-aware reranking included. Both arms of the comparison therefore ran under one retrieval policy; nothing computed on the answers can surface that, and it is visible at all only because the authors disclosed their protocol — which is exactly the auditability behavior at issue.

::: keep
The paper acknowledges this architectural distinction. It also observes that NotebookLM performs particularly well on prose-oriented qualities while ParliamentRAG's explicit structure provides stronger guarantees around properties such as quotation fidelity and political-group coverage. But the evaluation then largely compares $Y$ with $Y$. That misses a property specific to ParliamentRAG:

<div class="dbox">the path by which <em>Y</em> was produced</div>
:::

If an expert speaker is selected by ParliamentRAG, the architecture can in principle expose why that speaker was selected, what evidence contributed to the decision, which authority components were involved, and what alternatives were available. If a statement appears in the final synthesis, the system potentially has intermediate artifacts through which an evaluator can investigate how the statement entered the answer.

That suggests a different family of evaluation questions:

1. Can a final claim be traced to the evidence and transformations that produced it?
2. Can an evaluator distinguish retrieval failure from authority-selection failure from generation failure from integration failure?
3. Can the effect of changing one authority assumption be propagated through the pipeline and observed?
4. Can a previous output be reconstructed under the policy and evidence state that existed when it was generated?
5. Can disagreement between intermediate views be identified after those views have been integrated into fluent prose?

These are not ordinary response-quality metrics. They are properties of an inspectable decision system. The comparison with NotebookLM therefore seems incomplete if it remains confined to whether users prefer one final answer or another. ParliamentRAG may possess value that a monolithic system cannot express at the output layer at all.

## Quotation Fidelity Points Toward a Larger Principle

::: keep
The paper's handling of quotation fidelity provides perhaps the clearest example of this distinction. ParliamentRAG does not merely reward the model for producing accurate quotations; it introduces machinery intended to ensure that quotations correspond to source material. This is fundamentally different from assigning quotation quality another weight in an aggregate objective. Conceptually, the architecture says something closer to:

$$\neg\, \text{source-backed quotation} \;\Rightarrow\; \text{inadmissible quotation}$$
:::

::: keep
A property judged sufficiently important has moved from optimization into constraint. That raises what I think is the most interesting question produced by ParliamentRAG:

<div class="dbox hl">Which other properties of machine-mediated institutional synthesis should be optimized, and which should instead be structurally guaranteed?</div>
:::

Quotation integrity is one candidate — and the paper has already classified it: "quotation faithfulness is the strongest invariant enforced by the architecture," and "[o]ther properties, such as balanced group coverage, should instead be interpreted as strong design objectives rather than absolute guarantees." But parliamentary synthesis also involves authority provenance, representation of disagreement, uncertainty, policy identity, evidentiary support, and historical accountability. Some of these may be poorly represented as components of a single quality score.

::: keep
For example, a system could produce beautifully balanced prose while obscuring a disagreement that existed in the intermediate evidence. It could generate a factually plausible claim whose supporting quotation is authentic but does not actually entail the surrounding synthesis. It could select a defensible expert under one authority policy whose selection would change substantially under another equally plausible policy. These distinctions matter because:

$$\text{authentic quotation} \;\nRightarrow\; \text{supported claim}$$

and:

$$\text{high-quality answer} \;\nRightarrow\; \text{auditable judgment process}$$
:::

The architectural decomposition in ParliamentRAG gives us somewhere to begin investigating those differences.

## From Multi-View Generation to Preserved Disagreement

The multi-view generation pipeline is therefore, to me, one of the most important contributions of the paper. Generating parliamentary-group perspectives separately before integration creates an intermediate representation in which disagreement still exists structurally. That intermediate state is valuable.

::: keep
Once multiple views are integrated into a single fluent narrative, distinctions between them can be softened, compressed, or lost. The quality of the final prose tells us little about whether the transformation preserved the significant structure of the inputs. This suggests another possible evaluation target:

$$\text{views before integration} \;\to\; \text{integrated representation}$$

could itself be examined for information loss. Which disagreements survive integration? Which become generalized? Which disappear? Can each substantive synthesis claim be attributed to one view, several views, or an explicit reconciliation between them?
:::

If the system cannot reconcile two legitimate perspectives without distortion, perhaps incomparability or unresolved disagreement should itself be an admissible output. In political and institutional settings, forcing disagreement into artificial consensus may be a more serious failure than producing less elegant prose. The multi-view architecture makes that failure observable in a way that a monolithic generation pipeline may not.

## Toward Evaluation of Governed Transformations

For these reasons, I read ParliamentRAG as demonstrating something broader than authority-aware retrieval. It is an example of a system in which normative choices are beginning to move out of implicit model behavior and into explicit architecture. Authority weights express one policy; political-group coverage expresses another; expert-selection rules express another; quotation verification expresses another.

The interesting next step is not necessarily to eliminate those normative choices. Institutional information systems cannot avoid them. The more tractable objective is to make them identifiable:

- A policy can be explicit rather than latent.
- A threshold can be distinguished from a preference.
- A disagreement can survive rather than being averaged away.
- An intermediate decision can retain its provenance.
- A historical output can remain attributable to the evidence and policy state under which it was produced.

Seen this way, the important distinction between ParliamentRAG and a monolithic system is not simply modularity. It is whether consequential transformations become observable and governable.

The current evaluation establishes that ParliamentRAG can generate useful parliamentary summaries while providing strong performance on several source-oriented properties. A compelling continuation would evaluate the architecture as an inspectable judgment process: testing sensitivity to authority policy, tracing claims through intermediate transformations, measuring information loss during integration, and examining whether system outputs remain reproducible and interpretable as evidence or policy changes.

::: keep
ParliamentRAG already demonstrates that some important properties should not be left to prompting alone. The question it leaves me with is whether that principle can be carried further:

<div class="dbox hl">When an AI system mediates institutional knowledge, which properties may safely remain optimization objectives, and which must become invariants of the system that produces the answer?</div>

That seems to me like a consequential research direction hidden inside an already interesting paper.
:::

## References

Tritella, M., Pozzi, R., & Palmonari, M. (2026). *Who Speaks Matters: Authority-Aware Multi-View RAG over Italian Parliamentary Proceedings*. arXiv:2608.13410. https://doi.org/10.48550/arXiv.2608.13410

*The ISWC 2026 proceedings version will be cited here once published, per the authors' note on the preprint.*
