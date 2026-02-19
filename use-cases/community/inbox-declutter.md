# Inbox De-clutter

Newsletters can take up the inbox like nothing else. Often times they pile-up without being opened at all. 

## Skills you Need
[Gmail OAuth Setup](https://clawhub.ai/kai-jar/gmail-oauth).

## How to Set it Up
1. [optional] Create a new gmail specifically for OpenClaw.
2. [optional] Unsubscribe from all newsletters from your main email and subscribe to them using the OpenClaw email.
3. Install the skill and make sure it works. 
4. Instruct OpenClaw:
```txt
I want you to run a cron job everyday at 8 p.m. to read all the newsletter emails of the past 24 hours and give me a digest of the most important bits along with links to read more. Then ask for my feedback on whether you picked good bits, and update your memory based on my preferences for better digests in the future jobs.
```