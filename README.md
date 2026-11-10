# GPTokenizer

My notes and experiments on how tokenizers work in GPT-style language models, and why they matter more than most people expect.

## Why bother with tokenizers?

A tokenizer is the layer that sits between raw text and the model. The LLM never actually sees your text. It sees a sequence of integer IDs, and everything it knows about spelling, whitespace, numbers or other languages has to be learned through that lens.

That has some very visible consequences:

- **Odd failures on "simple" tasks.** Models can struggle to spell words, count letters, reverse strings or do arithmetic, partly because a word like `strawberry` may arrive as two or three opaque chunks rather than individual characters.
- **Uneven treatment of languages.** Text in languages that were rare in the tokenizer's training data gets split into many more tokens. That means higher cost, less usable context, and usually worse quality.
- **Whitespace and casing quirks.** `"hello"`, `" hello"` and `"Hello"` can map to completely different tokens.
- **Ambiguity and safety issues.** If tokenization is handled carelessly, you get real problems: inputs that tokenize differently than you assumed, special tokens injected through user text, glitch tokens that were barely seen during training, and filters that are bypassed because they check the text but the model reads the tokens. Getting this wrong can quietly weaken your guardrails.

## Keep the training sets separate

The data used to train the tokenizer should be thought of separately from the data used to train the LLM. The tokenizer's training data decides what the vocabulary looks like, so it directly shapes which strings become a single token and which get chopped up. If the two datasets are badly mismatched, the model can end up with tokens that it almost never saw during training. Those under-trained tokens tend to behave unpredictably.

## Choosing a vocabulary size

There isn't one correct number. Vocabulary size is a trade-off, and the sweet spot depends on what you are building:

| Smaller vocab | Larger vocab |
| --- | --- |
| Longer sequences for the same text | Shorter sequences, so more text fits in the context window |
| Small embedding and output layers | Bigger embedding and output layers (more parameters and memory) |
| Each token is seen often, so it is well trained | Rare tokens get few examples and are poorly learned |
| Handles odd text gracefully, closer to characters or bytes | Can overfit to the training distribution |

Some practical thoughts:

- A small, narrow task (one language, a limited domain, code in a single language) usually does fine with a modest vocabulary.
- A general multilingual model needs more room, otherwise non-English text gets fragmented badly.
- Past a certain point, extra tokens buy very little compression and cost you in model size and under-trained entries.
- The best approach is to measure it: train a few vocab sizes on your own data and compare compression (tokens per character or byte) and downstream results.

## What's in this repo

[`BaseTokenizer.ipynb`](BaseTokenizer.ipynb) is a from-scratch, byte-level BPE (Byte Pair Encoding) tokenizer in plain Python, with no libraries. The training text is Nathan Reed's *A Programmer's Introduction to Unicode*, which is a nice stress test because it's full of emoji, fullwidth letters and mixed scripts.

How it works:

1. **Start from bytes.** The text is encoded as UTF-8, so the base vocabulary is just the 256 possible byte values. Nothing is ever "unknown", and any string can be represented.
2. **Count pairs.** `get_stats` counts how often each adjacent pair of tokens occurs.
3. **Merge the most frequent pair.** `merge` replaces every occurrence of that pair with a new token ID (256, 257, ...). This is repeated `vocab_size - 256` times, and the merge order is stored in `merges`.
4. **Decode.** `decode` builds a `vocab` table of bytes for each ID, concatenates them and decodes UTF-8 with `errors='replace'`. The replacement is needed because an arbitrary sequence of IDs isn't guaranteed to be valid UTF-8.
5. **Encode.** `encode` turns new text into bytes, then repeatedly applies the earliest-learned merge available until none apply. The order matters here: merges have to be replayed in the same order they were learned.
6. **Measure.** The notebook prints the compression ratio (bytes before vs. tokens after), which is the simplest way to see what a given `vocab_size` buys you.

It's currently set to `vocab_size = 276` (20 merges), which is tiny and meant for seeing the mechanics. Real tokenizers use tens of thousands of merges.

Known limitations, which are the next things to explore:

- **No pre-tokenization.** GPT-2 and GPT-4 style tokenizers split text with a regex first, so merges can't cross boundaries like letters, digits, punctuation and spaces. Here, merges can span anything, so you can get tokens like `"e."` or `"d t"`.
- **No special tokens.** Things like `<|endoftext|>` aren't handled, and that's exactly where injection problems can show up if you add them carelessly.
- **Slow.** `encode` recounts all pairs on every loop. That's fine for a demo, but not for large texts.
- **Single training text.** Everything is learned from one article. As mentioned above, the tokenizer's training data shapes the vocabulary, so this one is heavily biased towards English and Unicode terminology.

## What I looked at extra to understand tokenizer more comprehensively

- **[Tiktokenizer](https://tiktokenizer.vercel.app/)** is a handy web tool for seeing how different models split the same text into tokens. It's great for building intuition about whitespace, numbers and non-English text.
- **[SentencePiece](https://github.com/google/sentencepiece)** is Google's library for training and running tokenizers (BPE and Unigram) directly on raw text, without relying on pre-tokenization by whitespace. It is widely used in models like Llama and T5.

## What I took away from this

The biggest surprise was how much weird LLM behavior comes down to the tokenizer.  A lot of "why can't it spell this?" moments are just tokenization showing through.

If you're building on top of one, two habits are worth it: look at what your input actually turns into (especially when filtering or security is involved), and don't settle to a default vocab size. Try a few on your own data and see what works.
