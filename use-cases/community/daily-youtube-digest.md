# Daily YouTube Digest

Start your day with a personalized summary of new videos from your favorite YouTube channels — no more missing content from creators you actually want to follow.

## Pain Point

YouTube notifications are unreliable. You subscribe to channels, but their new videos never show up in your home feed. They're not in notifications. They just... disappear. This doesn't mean you don't want to see them — it means YouTube's algorithm buried them.

Plus: it's fun to start the day with curated content insights instead of doom-scrolling a recommendation feed.

## What It Does

- Fetches the latest videos from a list of your favorite channels
- Summarizes or extracts key insights from each video's transcript
- Delivers a digest to you daily (or on demand)

## Skills You Need

Install the [youtube-full](https://clawhub.ai/therohitdas/youtube-full) skill.

Just tell your OpenClaw:

```text
"Install the youtube-full skill and set it up for me"
```
or 

```bash
npx clawhub@latest install youtube-full
```

That's it. The agent handles the rest — including account creation and API key setup. You get **100 free credits on signup**, no credit card required.

> Note: After creating the account, the skill auto-stores the API key securely in correct locations based on the OS, so the API will work in all contexts.

![youtube-full skill installation](https://pub-15904f15a44a4ea69350737e87660b92.r2.dev/media/1770620159490-e41e7baa.png)

### Why TranscriptAPI.com over yt-dlp?

| CLI tools (yt-dlp, etc.) | TranscriptAPI |
|--------------------------|---------------|
| Verbose logs flood agent context | Clean JSON responses |
| Doesn't work on GCP/cloud OpenClaw | Works everywhere, fast |
| Gets blocked randomly by YouTube | Powers [YouTubeToTranscript.com](https://youtubetotranscript.com) serving millions. Cached and reliable. |
| Requires binary installation | No binaries, just HTTP |

## How to Set It Up

### Option 1: Channel-based digest

Prompt OpenClaw:

```text
Every morning at 8am, fetch the latest videos from these YouTube channels and give me a digest with key insights from each:

- @TED
- @Fireship
- @ThePrimeTimeagen
- @lexfridman

For each new video (uploaded in the last 24-48 hours):
1. Get the transcript
2. Summarize the main points in 2-3 bullets
3. Include the video title, channel name, and link

If a channel handle doesn't resolve, search for it and find the correct one.
Save my channel list to memory so I can add/remove channels later.
```

### Option 2: Keyword-based digest

Track new videos about a specific topic:

```text
Every day, search YouTube for new videos about "OpenClaw" (or "Claude Code", "AI agents", etc).

Maintain a file called seen-videos.txt with video IDs you've already processed.
Only fetch transcripts for videos NOT in that file.
After processing, add the video ID to seen-videos.txt.

For each new video:
1. Get the transcript
2. Give me a 3-bullet summary
3. Note anything relevant to my work

Run this every morning at 9am.
```

This way you never waste credits re-fetching videos you've already seen.

## Tips

- `channel/latest` and `channel/resolve` are **free** (0 credits) — checking for new uploads costs nothing
- Only transcripts cost 1 credit each
- Ask for different digest styles: key takeaways, notable quotes, timestamps of interesting moments
- This already exists as a product - [Recapio - Daily YouTube Recap](https://recapio.com/features/daily-recaps)