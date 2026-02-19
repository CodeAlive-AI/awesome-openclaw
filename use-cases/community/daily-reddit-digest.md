# Daily Reddit Digest
Run a daily digest everyday to give you the top performing posts from your favourite subreddits.

What to use it for:

• Browsing subreddits (hot/new/top posts)
• Searching posts by topic
• Pulling comment threads for context
• Building shortlists of posts to manually review/reply to later

> It's read-only. No posting, voting, or commenting.

## Skills you Need
[reddit-readonly](https://clawhub.ai/buksan1950/reddit-readonly) skill. It doesn't need auth. 

## How to Set it Up
After installing the skill, prompt your OpenClaw:
```text
I want you to give me the top performing posts from the following subreddits.
<paste the list here>
Create a separate memory for the reddit processes, about the type of posts I like to see and every day ask me if I liked the list you provided. Save my preference as rules in the memory to use for a better digest curation. (e.g. do not include memes.)
Every day at 5pm, run this process and give me the digest.
```