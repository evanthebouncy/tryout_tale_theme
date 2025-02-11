---
layout: post
title:  "fine-tuned llm as program writers"
permalink: /generation-with-llm/
---

A program-writer is a conduit between specifications and programs. In this post, we will explain how large language models (llm) can be an universal program-writer for different domains.

## a menagerie of synthesizers

The synthesizer has a difficult job. It takes in semantics -- what the program should _do_, and produces syntax -- how the program should _look_[^repl]. <ins>Synthesis is running an interpreter backwards</ins>.

![Image with caption](/program-synthesis-minimal/assets/llm-generation/dual.png){: width="75%" }

Given this hard problem, it seems reasonable that for every problem domain there needs to be a domain-specific synthesizer tailored to it. This has indeed been the case. Here's a collection of program-writer architectures in some of my works.

![Image with caption](/program-synthesis-minimal/assets/llm-generation/zoo.png){: width="95%" }

While they look cool, significant manual engineering efforts and domain expertise were required to ensure that:
1. the tasks from a domain can be concisely expressed in the programming DSL
2. the synthesizer can reliably interpret the specification in its given form
3. the synthesizer can reliably generate legal programs in the DSL
4. the synthesizer can actually learn the structure of the meaning matrix M through training

This results in a messy co-evolution of DSL, interpreter, specification, and synthesizer -- as the program-writer architecture is highly coupled with the specification language and the DSL.

# large language models

A **large language model** (llm) gives a distribution over strings (which can be) conditioned on a prefix string. Trained on an enormous amount of textual data, it can make some pretty surprising conditional generations. Below is an example from the openai-codex model: prompted with a textual input, it generates a likely string completion as output (shown in blue highlight).

![openai codex](/program-synthesis-minimal/assets/llm-generation/codex.png){: width="95%" }

The capabilities and limitations of llm are (as of 2022-aug) still being investigated. As I am learning about it myself, I encourage you to become familiar with them[^stanford]. 

## llm for synthesis ?
These are my own assessments on why llm are useful for program synthesis. To make my case, I will show some interactions through prompting with the openai-codex model. 

### llm can pick-up programmatic patterns
Programs are stylized, patterned texts with distinct rules governing its generation and interpretation. It seems llm are predisposed to pick up these programmatic patterns.

![openai codex](/program-synthesis-minimal/assets/llm-generation/codex-prog.png){: width="70%" }

### llm has basic conventional knowledge of human language
llm has some knowledge of human conventions, expressed in large volumes of internet text.

![openai codex](/program-synthesis-minimal/assets/llm-generation/codex-convention.png){: width="95%" }

### llm are reliable in generating syntactically complex programs
Practitioners of program synthesis have avoided free-form text generation, as they're rarely coherent. However, llm are very good at generating stylistically consistent texts[^tweet1].

![openai codex](/program-synthesis-minimal/assets/llm-generation/codex-coherent1.png){: width="60%" }

### llm operates over text -- a universal datatype
The specifications and programs are often easily represented as structured texts, this allow you to easily integrate program synthesis with an existing programming system, or iterate through different DSLs and specification languages -- no parser required.

![openai codex](/program-synthesis-minimal/assets/llm-generation/codex-reframe.png){: width="80%" }

## where prompting falls short
For our rectangle example, prompting cannot be used to generate a valid rectangle for the spec. In fact, llm+prompting fails very often for the specific problems we want to solve out of the box -- the llm is trained on internet text data that are too different from our task.

![openai codex](/program-synthesis-minimal/assets/llm-generation/prompt-fail1.png){: width="100%" }

For more complex tasks, we turn to fine-tuning, which supplies the llm with in-domain data.

<br>
<hr>
[Let's take a break and vibe out](https://youtu.be/eB_k8rtRGwo). We'll be fine-tuning for real next.
<hr>
<br>

# fine-tuning large language models for synthesis

When one **fine-tunes** a model, they take an existing model with reasonable weights, and _continue to train it_ on a specific dataset. <ins>In synthesis, we take a llm trained on English and Python, and continue to train it on D</ins> -- a sample of the meaning matrix M of our (rectangle) domain. 

## the model and the tokenizer

The model only operates over sequences of ids, and relies the tokenizer to translate between strings and ids. Here is how the [codex model turns a string into a sequence of token-ids](https://beta.openai.com/tokenizer):

![Image with caption](/program-synthesis-minimal/assets/llm-generation/tokenizer-model1.png){: width="100%" }

We'll fine-tune the light-weight `byt5-base` model (2.17Gb). It treats each character (such as `a` or `]`) as its own token (total 256 token-ids). One does not need a "truly large" model, since we'll be giving it on plenty of in-domain datapoints. Getting the model and the tokenizer is fairly easy.

{::comment}
Compared to codex (>50k token-ids), it does not need to maintain a hefty embedding layer that maps tokens to a continuous representation.
{:/comment}

{% highlight python %}
!pip install transformers datasets
from transformers import T5ForConditionalGeneration, AutoTokenizer, TrainingArguments
tokenizer = AutoTokenizer.from_pretrained('google/byt5-base')
model = T5ForConditionalGeneration.from_pretrained('google/byt5-base')
{% endhighlight %}

## data

We'll be using the same kind of dataset D as the previous post.

{% highlight python %}
# upload the following 2 files onto the colab vm instance
from rectangle import *
from rectangle_lm import *
import random

D_train = sample_D(5000)
D_test = sample_D(1000)
{% endhighlight %}

## encoding the spec and prog
Encoding requires creativity -- you want to accentuate the information for the model to make the correct decision. The run-time of the transformer algorithm is O(n^2), which is sensitive to the length of the strings. Thus, I simply try to minimize the length of the spec *represenation*.

Do keep in mind that re-naming the tokens comes at a cost of how the model can interpret natural language -- for instance, if you rename 'blue' as '+'. This is usually fine if you have enough fine-tuning examples (for program synthesis, we can usually generate as many `(prog,spec)` pairs as we want).

{% highlight python %}
def spec_to_str(spec):
    return repr(spec).replace('True','+').replace('False','-').replace(')','').replace('(','').replace(' ','').replace(',','')

# before, length 261
[((1, 1), True), ((5, 0), False), ((3, 4), True), ((4, 5), False), ((0, 4), True), ((2, 3), True), ((3, 3), True), ((4, 5), False), ((0, 3), True), ((4, 3), True), ((0, 0), True), ((5, 2), False), ((4, 1), True), ((1, 2), True), ((5, 5), False), ((0, 3), True)]
# after, length 50
[11+50-34+45-04+23+33+45-03+43+00+52-41+12+55-03+]

{% endhighlight %}


## minibatch to tensor with collator

The collator takes a batch of input-outputs of different lengths and pad it into nice "rectangular" tensors that can be fed into the model for training.

![Image with caption](/program-synthesis-minimal/assets/llm-generation/collator.png){: width="80%" }

This is what it looks like in code:
{% highlight python %}
class Collator:
  def __init__(self, tokenizer):
    self.tokenizer = tokenizer

  def __call__(self, batch):
    # entry[1] is the spec, need to call repr to turn it into a string. entry[0] is the prog_str already
    ret = {"input_ids": self.tokenizer([spec_to_str(entry[1]) for entry in batch], padding=True, return_tensors='pt').input_ids, 
            "labels": self.tokenizer([entry[0] for entry in batch], padding=True, return_tensors='pt').input_ids}
    return ret
{% endhighlight %}

## training, and interpreting the training loss

Training with the huggingface API is quite simple, simply call the Seq2Seq trainer class.
{% highlight python %}
trainer = Seq2SeqTrainer(model=model,args = training_args,train_dataset=dataset,eval_dataset=None,tokenizer=tokenizer,compute_metrics=None,data_collator=Collator(tokenizer))
trainer.train()
{% endhighlight %}

As the model is training, you will see the loss

![Image with caption](/program-synthesis-minimal/assets/llm-generation/training_loss.png){: width="40%" }

The loss tells us how closely does the model's distribution match the training data, smaller loss means closer match. Quantitatively, <ins>the loss can be interpreted as a **per-token-perplexity**</ins>, we'll explain this with an example.

Let's say we have a training data-point `("11+33+04-41-","[1,3,1,4]")`, and it has a loss of `1.18` at iteration-500. It means that given the input `"11+33+04-41-"`, on average, each token in `[1,3,1,4]` has a perplexity of `e^1.18 = 3.25`, or a 1 in 3.25 chance of being correctly generated. As `[1,3,1,4]` has a total of 9 tokens, and our model generates it 1 token at a time, the entire sequence has a `1 / 3.25^9 = 1 / 40453` chance of being correctly generated by our model.

At iteration-1000 we have a loss of `0.5`, or a token-perplexity of `e^0.5 = 1.64`. Thus, the model has a `1 / 1.64^9 = 1 / 86` chance of generating the entire correct sequence. However, given a spec, there may be _multiple_ satisfying programs, and the "1 in 86 chance" only speaks of a particular program, `[1,3,1,4]`. The model may very well find a different program that is also correct, making `1 / 86` a _lower bound_.

## inference

After training (fine-tuning) on D, we can sample a few program-strings from specification -- these are the candidates for rejection sampling down the line. The key parameter to sampling is **temperature**, a temperature of 1.0 is typically good -- if you trust your model has fully learned the ground-truth distribution, a temperature of 1.0 is exactly it. A too-low temperature will ruin diversity, a too-high temperature will be too chaotic.

{% highlight python %}
def generate_samples_with_temp(txt, n_samples, temp):
    to_tokenizer = [txt for i in range(n_samples)]
    outputs = model.generate(tokenizer(to_tokenizer, return_tensors='pt', padding=True).input_ids.to('cuda'), do_sample=True, max_length=128, temperature = temp)
    results = tokenizer.batch_decode(outputs, skip_special_tokens=True)
    return results

generate_samples_with_temp(repr(D_test[0][1]), 5, 1.0)
# ['[4,6,1,6]', '[1,5,3,5]', '[2,3,4,6]','[4,5,0,2]','[1,4,4,5]']
generate_samples_with_temp(repr(D_test[0][1]), 5, 3.0)
# ['[Tb2,2,6}', 'ee[>1,6,265,741F4]', '[0o9ѕdr_a|4,80,7]6ٓ5]і-$4732"r-H,', '[2˴в[4,4,3', '[A518,2/8žrev0B;D']
{% endhighlight %}

This result is quite incredible, after fine-tuning, a llm that in theory can generate any string, is quite confidently generating correct looking programs/rectangles.

## making a synthesizer with fine-tuned llm

We simply integrate the `llm_writer` into the synthesizer. As during training we have roughly 1 in 86 chance to recover the correct program (this being a lower bound), then a search budget of 20 is reasonable. <ins>As of today, inference from llm is still relatively slow compared to the tens of thousands of samples typically required for a SOTA program synthesizer</ins>, but I don't think this will be a problem few years down the road -- we just wait.

{% highlight python %}
def llm_writer(spec):
    # in practice please batch this instead of 1 at a time lol
    return generate_samples_with_temp(spec_to_str(spec), 1, 1.0)[0]
synthesizer6 = get_synthesizer(llm_writer, is_correct, 20)
{% endhighlight %}

## results

As we can see, by simply manipulating some strings, we were able to beat the other baselines in performance. Since llm are fully expressive, it does not have the issue of under-fitting (unlike the unigram-writer) and continued training will only increase performance.

![Image with caption](/program-synthesis-minimal/assets/llm-generation/result.png){: width="80%" }

## code

[All colab notebook of this post can be found here](https://colab.research.google.com/gist/evanthebouncy/7470b3e5facfb2d659145bddf47836c2/rectangle_v2.ipynb)

It requires [rectangle.py](https://gist.github.com/evanthebouncy/25114aaf0be20df21468735aa7103bef) and [rectangle_lm.py](https://gist.github.com/evanthebouncy/1703d3e9aee71ba9124405fdb30bd967) which were defined in previous posts.

## exercise

Use different sampling methods such as top-p sampling or beam-search to take samples from our fine-tuned llm [using this guide for reference](https://huggingface.co/blog/how-to-generate). Which sampling algorith is more efficient given different search budgets?

# conclusion future works
This concludes my 3 part series, I hope you enjoyed reading it as much as I had fun writing it. Most importantly, I hope you are comfortable starting a program synthesis project. 

I will post more synthesis topics as they become "mature enough" for a succinct blog post. I'll be announcing these updates on my [twitter (gimme a follow!)](https://twitter.com/evanthebouncy)[^tweet]. You can suggest new topics for me to write about, and please let me know if I was mistaken or unclear in my writings.

-- evan 2022-08-31

### notes
[^tweet1]: [gpt3 generating coherent codes](https://twitter.com/goodside/status/1563989550808154113)

[^repl]: The gap between syntax and semantics motivated some of our works on execution-guided synthesis such as [this](https://arxiv.org/abs/1906.04604) and [this](https://arxiv.org/abs/2012.12964).

[^stanford]: [CS324, a stanford course on understanding and developing large language models](https://stanford-cs324.github.io/winter2022/lectures/introduction/)

[^tweet]: [follow my twitter for more program synthesis contents](https://twitter.com/evanthebouncy). Also, feel free to DM me with anything program synthesis related.