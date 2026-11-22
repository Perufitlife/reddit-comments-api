# Reddit comments API: every comment of a post as JSON, with score, depth and parent

How to get the comments of any Reddit post as JSON without a Reddit developer app, an OAuth token or a login: the comment text, author, score, date, depth in the thread and the parent comment, plus the post title and subreddit on each row.

The data comes from the Apify Actor [Reddit Comments Scraper](https://apify.com/renzomacar/reddit-comments-scraper). It reads the public thread, the same one a logged-out visitor sees on reddit.com. Paste post links in any form (www, old.reddit.com, with tracking parameters) and get one row per comment.

## Why not the official API

Since 2023 the Reddit Data API needs a registered app, OAuth, and for anything beyond personal use, a commercial agreement with Reddit. The free tier is rate limited to 100 queries a minute per client and its terms forbid using the data to train models. If you only need the comments of some posts, a scraper of the public page skips the app registration and the token handling.

## Example: which expenses small business owners underestimate

Market research from one thread: [What's one business expense you underestimated when you first started?](https://www.reddit.com/r/smallbusiness/comments/1xxoc76/whats_one_business_expense_you_underestimated/) in r/smallbusiness, one of the most discussed posts of the week.

Input:

```json
{
  "postUrls": ["https://www.reddit.com/r/smallbusiness/comments/1xxoc76/whats_one_business_expense_you_underestimated/"],
  "maxCommentsPerPost": 300
}
```

Run on 2026-11-22: 44 comments in 13 seconds, 35 of them top-level answers. Grouping them by what they mention:

| Expense | Comments | Upvotes |
|---|---|---|
| Bank, card and processor fees, chargebacks | 6 | 97 |
| Packaging and supplies (to-go boxes, cups, lids) | 5 | 96 |
| Time: bookkeeping and admin | 6 | 91 |
| Taxes, import duties and compliance | 6 | 79 |
| Rent and build-out | 5 | 63 |
| Insurance | 10 | 58 |
| Software and tech upkeep | 6 | 48 |

Insurance is the most repeated answer, but fees and packaging got the most upvotes. The top comments:

| Score | Comment |
|---|---|
| 65 | To-go boxes. I figured the food was the expensive part, then I watched a whole stack of boxes and cups walk out the door every single night. |
| 61 | sales tax and card fees for me, but the real surprise was how much time the bookkeeping ate. |
| 33 | Build-out. And especially if you're renting it's like giving the landlord a free upgrade. |
| 21 | Software costs have absolutely spiraled in the past 5-10 years. |
| 15 | Bank fees. What the f\*\*\* do you mean it costs me US$30 to make one international transaction, whether it's for $10k or $100? |

The first ten rows in full: [sample-output.json](sample-output.json). One of them:

```json
{
  "dataType": "comment",
  "redditId": "t1_19sq3tvn7miub",
  "postTitle": "What’s one business expense you underestimated when you first started?",
  "author": "Thebypass168official",
  "subreddit": "smallbusiness",
  "score": 65,
  "body": "To-go boxes. I figured the food was the expensive part, then I watched a whole stack of boxes and cups walk out the door every single night. It adds up a lot faster than you think.",
  "url": "https://www.reddit.com/r/smallbusiness/comments/1xxoc76/comment/19sq3tvn7miub/",
  "postUrl": "https://www.reddit.com/r/smallbusiness/comments/1xxoc76/whats_one_business_expense_you_underestimated",
  "createdUtc": "2026-11-16T13:35:59.576Z",
  "parentId": "",
  "depth": 0
}
```

`depth` is 0 for answers to the post and grows by one for each level of reply; `parentId` points to the comment being answered, so the tree can be rebuilt.

## Call it from code

You need an Apify account and its API token.

```bash
curl -X POST "https://api.apify.com/v2/acts/renzomacar~reddit-comments-scraper/run-sync-get-dataset-items?token=$APIFY_TOKEN" \
  -H "content-type: application/json" \
  -d '{"postUrls":["https://www.reddit.com/r/smallbusiness/comments/1xxoc76/"],"maxCommentsPerPost":100}'
```

Python, top-level answers sorted by score:

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("renzomacar/reddit-comments-scraper").call(run_input={
    "postUrls": ["https://www.reddit.com/r/smallbusiness/comments/1xxoc76/"],
    "maxCommentsPerPost": 300,
})
rows = client.dataset(run["defaultDatasetId"]).iterate_items()
answers = sorted((r for r in rows if r["depth"] == 0), key=lambda r: r["score"], reverse=True)
for r in answers[:10]:
    print(r["score"], r["body"][:100])
```

JavaScript, the thread as an indented tree:

```js
import { ApifyClient } from "apify-client";

const client = new ApifyClient({ token: process.env.APIFY_TOKEN });
const run = await client.actor("renzomacar/reddit-comments-scraper").call({
  postUrls: ["https://www.reddit.com/r/smallbusiness/comments/1xxoc76/"],
});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
for (const c of items) console.log("  ".repeat(c.depth) + `[${c.score}] ${c.body.slice(0, 80)}`);
```

## Use it from Claude, ChatGPT or another AI agent (MCP)

```json
{
  "mcpServers": {
    "apify": {
      "url": "https://mcp.apify.com/?tools=renzomacar/reddit-comments-scraper",
      "headers": { "Authorization": "Bearer YOUR_APIFY_TOKEN" }
    }
  }
}
```

Then ask, for example: "Read the comments of this Reddit thread and list the pain points people mention, with how many upvotes each got."

## Input fields

| Field | What it does |
|---|---|
| `postUrls` | Links to Reddit posts, one per line, in any form. |
| `maxCommentsPerPost` | Stop after this many comments per post, in the order the thread shows them (best first). |

## What it does not do

- It reads posts you give it. To find posts first (a subreddit's top posts of the week, or a keyword search), use [Reddit Scraper](https://apify.com/renzomacar/reddit-scraper) and pass its post links here.
- Deleted and removed comments, and comments hidden from logged-out visitors, are not returned, so the count can be below the number Reddit shows on the post.
- No login, no private or quarantined subreddits.

## Price

Pay per comment returned, with the price shown on the Actor page. No start fee and no charge per post.
