


Yesterday I listened in on a call with a major financial services firm, an information session on blockchain technologies. While the call was helpful, I wanted to generate audio recording of the session, as well as a text transcription of the session - then it'd be searchable, much easier to read quickly, etc. Having no experience with the Google Speech-to-Text API, I figured this would be a good chance to try it out. Here's the end to end process.

## Goal

Go from an audio recording we can call in and listen to to a transcribed text in a shareable format. Note I run Mac OS Sierra 10.12.6.

### Step 1: Generate audio recording

Since the audio was only available to be listened to via a conference link, I had to work out a way to record the conversation onto my local machine, so I could provide that file to the Google Speech-to-Text API for transcription. (This isn't exactly true - it is possible to provide microphone input / stream translation, but this felt like a more complicated use case, so I decided to record it).

Using a nifty utility I discovered a few days ago - [Soundflower](https://rogueamoeba.com/freebies/soundflower/) - I rerouted audio output from Internal Speakers to Soundflower (2ch). This meant I couldn't hear the output, but Soundflower could record it. Using the Record New Audio function on QuickTime Player on Mac - and selecting Soundflower (2ch) as the audio source - I was able to call into the conference call recording (using Gmail's native Phone Calls utility) and record the audio directly to my local hard drive. I just had to let it run for the hour-long duration of the call, but it was a set-it and forget-it type of thing.

So, now I had a ~95MB .m4a file with the entire duration of the hour-long call in clear English.

### Step 2: Generate transcription

So I thought this bit would be easy - just drop the file into the Google Speech-to-Text API [demo](https://cloud.google.com/speech-to-text/) and voila ... not so fast. The demo has a size limit (50MB) and a time limit (1 min). After working this out I decided to chop the hour-long M4A into 59 second clips. Here enters another awesome command line utility I discovered for the task: [ffmpeg](https://www.ffmpeg.org/). This appears to be the master audio and video utility, and it did a stellar job.

#### Step 2A: Chop recording

This command took a 57 minute recording (`output.mp3`) and chopped it into 58 59-second recordings:

`ffmpeg -i ./output.mp3 -f segment -segment_time 59 -c copy out%03d.mp3`

Then I went ahead and tested the `gcloud ml speech recognize` command - no luck. Somehow it doesn't recognize MP3s without additional configuration / flags ... I figured that it'd actually be easier (thanks to ffmpeg) to just convert the 59-second .mp3 files into a format that would work natively (.wav) than go through the headache of working out those flags.

#### Step 2B: Convert mp3 to wav

Since we had a batch I wrote this little bash script to automate the conversion - after trial and error it worked like a charm:

`mp3-to-16bit-wav.sh`
```bash
for F in ./subfolder/*; do
	echo "CONVERTING "$F
	SUBSTRING=$(echo $F| cut -d'/' -f 3)
	S=$(echo $SUBSTRING | cut -d'.' -f 1).wav

  ffmpeg -i $F -acodec pcm_s16le -ac 1 -ar 16000 ./subfolder/wav/$S

done
```

Still early days in my bash skills - my string slicing skills need some work, but this did the job. Now I had a folder with 58 59-second .wav files - after testing one to make sure that it was compatible with gcloud ml speech (this was the biggest stumbling point - the API is quite finicky on the format of the audio it accepts. For example, it had to be 16-bit wav, not 8-bit...). It was, which just left automating the API calls for each file in the ./subfolder/wav directory:

```bash
for F in ./subfolder/wav/*; do
  gcloud ml speech recognize $F --language-code='en-US' >> ./subfolder/output/transcription.json
done
```

This appended the API call response - JSON - to a single transcription.json file. It wasn't going to end up to be valid JSON  I'd have to open with a `[` and add commas between each JSON object returned, then close with `]` for it to be a valid array - a quick find and replace.

### Step 3: Clean up transcription

Last step is to get the transcription in a human-readable format. The series of API calls created a 1076-line file of appended JSON objects - adding commas between the `}{` and wrapping it in angle brackets turned it into valid JSON, which could be easily read and iterated through with a python script:

  ```python
import json

json_file = open('./transcription.json').read()
json_data = json.loads(json_file)

# To get an output with indications of the minute we're on:
minute = 0
with open('transcription.txt', 'a') as txt:
    for response in json_data:
        txt.write(str(minute) + ':00\n')
        txt.write(response['results'][0]['alternatives'][0]['transcript'] + '\n\n')
        minute += 1
  ```

  With that, we get an output .txt file with every API response appended - a file of the transcribed audio.

 Looking at it, it leaves a lot to be desired. First, there is no punctuation - looking back I maybe should have included the `enable automatic punctuation` flag - the current output is not massively readable. Furthermore, there is the odd mis-transcribed word, which is to be expected (though I imagine this is getting better every day - right Google?). Finally, I am shocked by how meandering people are when they are talking, rarely coherently finishing thoughts but rather jumping around from point to point. It is easy to follow when listening but makes reading a transcription of a presentation quite tough.

## Next steps

Lots of opportunity to improve on this.

1. Punctuation and readability. Leverage the gcloud ml speech API or some other tools to automate the insertion of punctuation and of paragraph breaks etc.
2. NLTK analysis. Lots to be done in analyzing words, sentiments, etc.
3. Compare different transcription APIs for accuracy, speed etc. - [Watson](https://www.ibm.com/watson/services/speech-to-text/), [Bing](https://azure.microsoft.com/en-us/services/cognitive-services/speech/), [Twilio](https://www.twilio.com/speech-recognition)
3. Wait for Google's AI to improve.
