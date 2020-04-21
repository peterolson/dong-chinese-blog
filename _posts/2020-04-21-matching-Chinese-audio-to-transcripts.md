---
layout: post
title:  "Synchronizing Chinese audio with a transcript"
---
Recently I added a feature to [Dong Chinese media](https://www.dong-chinese.com/media) that shows the current sentence and word being spoken:
<div class="video-container" style="position: relative;padding-bottom: 56.25%;padding-top: 30px; height: 0; overflow: hidden;">
<iframe width="560" height="315"  style="position: absolute;top: 0;left: 0;width: 100%;height: 100%;" src="https://www.youtube.com/embed/OfFrJ3h5Kdg" frameborder="0" allowfullscreen></iframe>
</div>
I had thought about doing this before, but I quickly dismissed the idea because I thought it would take a lot of manual work to watch every video and align the text and audio myself.

I was inspired to automate this when I watched a [video about automatically making a slideshow video for a spoken monologue](https://www.youtube.com/watch?v=Jr9sptoLvJU) using the Google images API and a program called a [**forced aligner**](https://github.com/pettarin/forced-alignment-tools):

> Given an audio file containing speech, and the corresponding transcript, computing a **forced alignment** is the process of determining, for each fragment of the transcript, the time interval (in the audio file) containing the spoken text of the fragment.

I didn't realize that such a tool existed before, so the idea excited me and I started looking into different forced aligners. There is an abundance of options for aligning English transcripts, but it was harder to find tools that could work with Chinese text.

## What I tried first

First I tried [aeneas](https://www.readbeyond.it/aeneas/), but the results were not as good for Chinese as they were for English and Italian. Then I tried the [Montreal Forced Aligner](https://montrealcorpustools.github.io/Montreal-Forced-Aligner/), which was more promising, but it had an [inexplicable error during alignment](https://github.com/MontrealCorpusTools/Montreal-Forced-Aligner/issues/63) for about 60% of the transcripts I had. It also didn't have good results when there were wordless pauses in the audio, and it was very slow when the audio files were long.

After struggling with these tools for a few days, I started looking for a different approach. I came across this [research article describing an algorithm for forced alignment](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.649.6346&rep=rep1&type=pdf) designed to work well for longer audio files and to be tolerant of some discrepancies between the transcript and the audio. They didn't have any code available that I could use directly, but it gave me an idea of how I could approach it better.

## My final solution

I used the [Amazon Transcribe API](https://aws.amazon.com/transcribe/) to perform speech recognition on the audio files, which not only outputs a transcript, but also gives the timestamps of each recognized word. This reduced the problem to aligning the transcript I had with the timed transcript from the speech recognition output.

I borrowed the recursive approach from the article I read. To show how it works, let me show an example:

```
Original: 大家好我是小高姐今天我们来做炒拉条子先来和面中筋面粉盐和水成团就可以了
  Amazon: 嗯大家好我是想了解今天我们来做朝拉条子先来活面中间面粉盐和水嗯嗯嗯嗯成团就可以了
```

First I found the [longest common substring](https://en.wikipedia.org/wiki/Longest_common_substring_problem) between the original transcription and the Amazon transcription, with some logic added to tolerate different characters that sound the same or similar. Given that, I now have an aligned substring somewhere in the middle, and the remainder on the left and the right that still need to be aligned:

 - Longest common substring: `姐今天我们来做炒拉条子先来和面中筋面粉盐和水`
 - Left remainder: 
   ```
   Original: 大家好我是小高
     Amazon: 嗯大家好我是想了
   ```
 - Right remainder: 
 ```
   Original: 成团就可以了
     Amazon: 嗯嗯嗯嗯成团就可以了
   ```

Now the same process can be applied recursively to both the left and right remainders, until there is a pair of fragments that don't have a common substring.

For anyone interested in going through the alignment code I wrote, [I have uploaded it to GitHub](https://github.com/peterolson/aws-chinese-forced-alignment).

## Limitations

This feature is **not implemented on musical sections** of Dong Chinese media, namely [Chinese Buddy](https://www.dong-chinese.com/media/Chinese%20Buddy), [Popular songs](https://www.dong-chinese.com/media/Popular%20songs), and [Children's songs](https://www.dong-chinese.com/media/Songs%20for%20children). In the future I might try to think of a way to add them, but currently there are a few obstacles preventing the current approach from working well with songs.

 - **Songs often repeat the same text many times.**  
Firstly, this dramatically increases the likelihood of the longest common substring between the original transcript and the Amazon transcript to not actually correspond to the same timestamp.  
Secondly, the original transcript often only shows the text once when it is sung multiple times. Handling this would require allowing a single fragment in the original trancript to align with multiple fragments in the Amazon transcript. Allowing this type of one-to-many mapping would likely make the process much more complicated and error-prone.  
 - **Songs often have long stretches of background music without words.**
Long wordless periods make it more likely for the speech recognition to misidentify background noise as words.
 - **Speech recognition is trained primarily on spoken language**, not on sung language. The rhythm of language in songs is very different from speech, so the speech recognition output will probably be less accurate for songs.
