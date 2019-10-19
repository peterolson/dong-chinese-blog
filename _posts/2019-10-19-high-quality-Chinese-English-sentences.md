---
layout: post
title:  "Searching for high-quality Chinese-English sentence pairs"
---
![Sentence filtering](/images/SentenceFiltering.png)  
The main philosophy of [Dong Chinese](https://www.dong-chinese.com/) is that language should be learned through **context**, rather than memorizing individual words in isolation. This requires a large collection of complete sentences.  

To that end, I collected millions of English-Chinese sentence pairs from a variety of corpora, including:

 - [Tatoeba](https://tatoeba.org/)
 - [UN documents](http://opus.nlpl.eu/MultiUN.php)
 - [Open Subtitles](https://www.opensubtitles.org/en/search/subs)
 - [UM corpus](http://nlp2ct.cis.umac.mo/um-corpus/) (Education, Microblog, News, Science, Spoken, Subtitles)
 - [AI Challenger translation dataset](https://challenger.ai/dataset/translation)

One of the biggest ongoing issues with Dong Chinese has been that many of these sentences pairs are not suitable due to any combination of these reasons:
 - The English is low-quality. (e.g. spelling, grammar, punctuation, nonsensical, unidiomatic)
 - The Chinese is low-quality. 
 - The sentences don't match. (e.g. different meanings, key words missing in one language)

Of course, it is not feasible to manually sort through millions of sentences to try to fix these problems. It would take a lifetime for me to complete such an enormous task by hand. For that reason, I had to try to find ways to automatically filter out the worst sentence pairs.

Since the beginning I have been developing a motley collection of different filtering algorithms that subjectively seemed to improve the accuracy, but until recently I had no good way of evaluating how effective they actually were. Last month I finally realized that I should have a better way of measuring the accuracy of my filters.

I took a sample of 5000 random sentence pairs from the different sources, and over the course of a few weeks I manually labelled them as "good-quality" or "bad-quality". Once I had these labels, I could evaluate how much my algorithmic "low-quality" detection coincided my own subjective opinion of what was "low-quality".

## Baseline quality

Before filtering, this was the percentage of "high-quality" sentences according to my subjective labelling of the random sample:
 - Tatoeba: **85.9%**
 - UM Corpus:
   - Subtitles: **72.3%**
   - Microblog: **71.2%**
   - News: **53.9%**
   - Spoken: **52.4%**
   - Education: **46.8%**
   - Science: **45.6%**
  - AI Challenger: **46.0%**
  - Open Subtitles: **20.9%** (oh my...)

Upon seeing these numbers, I decided to completely throw out sentences from Open subtitles because of the very poor latent quality. I suspect that a large portion of the Chinese sentences from Open Subtitles were translated from English via machine translation, and not very good machine translation at that. Google translations were generally better than the text in the Open Subtitles dataset.

I also decided to throw out the sentences from UN documents. This was not due to the quality (I didn't measure it), but because there were relatively few beginner-level sentences, and the more advanced sentences tended to be rather boring (e.g. "He asked who the members of the Procurement Task Force were.").

## Filtering techniques

I trained a [simple neural network](https://en.wikipedia.org/wiki/Multilayer_perceptron) with one hidden layer to classify sentence quality on a spectrum between 0 (worst quality) and 1 (best quality) based on these 8 features:

### 1. Punctuation and formatting issues
This was a simple pattern-matching function to detect anomalies in punctuation and spacing. For example, 
 - if the Chinese sentence has a question mark and the English sentence does not, then it is likely that they do not match.
 - if there are mismatched quotation marks or parentheses, then the sentence is likely incomplete or low-quality.
 - sentences should end with punctuation (one of ".?!") and English sentences should begin with a capital letter.

### 2. Length ratios
If the Chinese sentence is very short and the English sentence is very long, or vice-versa, it is likely that the sentences do not completely match. English sentences are generally between 2.33 and 3.94 times longer than Chinese sentences.

### 3. Profanity
Filters out sentences with some common profanities in English and Chinese. The reason for this was not primarily, as one might expect, to only show G-rated sentences on Dong Chinese (I cannot guarantee that), but because translations with profanity had a tendency to be loose or inaccurate.

### 4. Readability
Many sentence pairs were grammatically correct and matched each other, but contained jargon that many people wouldn't know. (e.g. "The broth of Cordyceps militaris fermentation contained antimicrobial active substances") This issue was especially prevalent in the Education and Science corpora. Sentences with a high [Flesch-Kincaid readability score](https://en.wikipedia.org/wiki/Flesch%E2%80%93Kincaid_readability_tests) were more likely to be too difficult to read.

### 5. Name detection
There were many transliterated English names in the Chinese sentences. This is not technically a problem with the dataset, but it is a problem for Dong Chinese because many characters were shown more often in transliterations than in their native context. For example, there were many sentences where "简" (jiǎn; simple) was a transliteration of the name "Jane". Similarly, there were many more sentences with 汤 (tāng; soup) used in the name 汤姆 (Tāngmǔ; Tom) than sentences that were actually related to soup.

I used the [Hanzi Tools](https://github.com/peterolson/hanzi-tools) part-of-speech tagging tool to detect transliterated names in Chinese, and the [wink-pos-tagger](https://winkjs.org/wink-pos-tagger/) to do the same detection in English.

### 6. Missing verbs
Using the same part-of-speech tagging mentioned above, I filtered out sentences that didn't have any verbs, since they tended to be incomplete sentence fragments.

### 7. English grammar checking

I used the [LanguageTool API](https://languagetool.org/dev#api) to detect grammar and spelling mistakes in the English sentences. I also tried the grammar checker for Mandarin, but it wasn't nearly as useful as the English one and did not offer any improvement over the baseline accuracy.

### 8. Sentence similarity

To calculate the similarity between the Chinese and English sentence, I followed this process:

1. Run the Chinese sentences through machine translation.  
  Now there are two English sentences: one from the dataset and one automatically translated from the Chinese sentence
2. Compare the similarity of the two English sentences using [Tensorflow's Universal Sentence Encoder](https://tfhub.dev/google/universal-sentence-encoder/2).  
  This calculation is rather computationally expensive. I ran similarity calculations for the sentences in the corpora overnight every day for about a week.

This still allows overly literal word-for-word translations to slip through the cracks, and eliminates some good translations, but in general this was the most powerful filtering technique I had.

There is now a [multilingual sentence encoder](https://tfhub.dev/google/universal-sentence-encoder-multilingual/1) that came out in July 2019. This might allow skipping the machine translation step, but I have not evaluated it yet, since I wrote my sentence similarity script before this model was released.

### Results

I ran the classifier network on the sentences in the testing sample, and rejected sentences that it graded below 0.8. It rejected about four out of five sentences. This was the percentage of "high-quality" sentences in the remainder according to my manual labels:
 - Tatoeba: **94.4%** (+**8.1%**)
 - UM Corpus:
   - Subtitles: **94.4%** (+**22.0%**)
   - Education: **93.9%** (+**47.1%**)
   - Spoken: **92.5%** (+**40.1%**)
   - Science: **89.7%** (+**44.1%**)
   - News: **84.7%** (+**30.8%**)
   - Microblog: **77.1%** (+**5.9%**)
  - AI Challenger: **85.7%** (+**39.7%**)
  - Open subtitles: **65.0%** (+**44.1%**, still not good enough)

It's a big improvement. I'm proud of it, but I still can't say that the problem is "solved". There is still a lot of potential for improvement, and I'm always open to hearing suggestions. As always, if you see a low-quality sentence on Dong Chinese, you can report it with the feedback button.

![Feedback button](/images/feedback_button.png)