# KodaBot
<!-- [<img src="https://open.autocode.com/static/images/open.svg?" width="192">](https://autocode.com/app/petersopko/kodabot-v1/) -->

1. Make sure to adjust environment variables
2. GitHub Personal Key needs to be generated for bot and GitHub account linked on Autocode
3. Branch for bot needs to be created on GitHub and settings adjusted in settings.js
4. Cloudflare Account specs need to be set in environment variables, namespaces for Devs, Pulls and Tables need to be created in KV Workers section manually at
   https://dash.cloudflare.com/${yourAccountId}/workers/kv/namespaces
6. Comment messages and settings need to be adjusted in settings.js
7. Bot has currently two main parts: **assign** and **payout**.

### Assign:
- bot gets triggered by one of the `goPhrases` in form of comment under issue
- bot assigns issue to user and allocates time for completion under `assignmentPeriod`
- during another user's `assignmentPeriod` triggering bot will add user to queue
- after `assignmentPeriod` expires, next user in queue has `optionPeriod` to pick issue, currently set to flat 12 hours during which issue can't be picked up by another user
- bot periodically unassignes expired issues
- user may choose to pass on chance to pick up issue during `optionPeriod` or during him being in queue by triggering bot with one of the `passPhrases`
- each user has one chance of completing an issue within given bounty time frame, where successfully open PR stops bot from checking `assignmentPeriod`
- dropping out of queue / passing option to pick up issue or not finishing in time, disqualifies you from getting assigned to the same issue again
- each user has `5 assigned issues limit` set


### Payout:
- bot is tracking:
    - pull requests
    - payments for pull requests their completion (merging/closing),
    - possible `finishedStreaks` of users (3 issues done within 7 days for one dev)
- bot generates `leaderboard` which is periodically pushed to main as `LEADERBOARD.MD`
- bot is verifying payout data with Subscan API
- `LEADERBOARD.MD` is generated every 10 merged and paid pull requests


### Objects
**Autocode's temp storage:**
```
issueObject:
   {
     assignee: string, // assigned user
     timeOfAssignment: string, // storing for possible finishedStreaks
     assignmentPeriod: string, // time at which assignment time expires
     prOpened: number // number of PR opened for this issue
     optionHolder: string // user who can pick issue during option period
     optionPeriod:string // time at which option period expires
     queue: [] // queue of users waiting for their chance
   },
  
mergedAndPaid: [] // list of merged and paid PR numbers,
used to track time of pushing LEADERBOARD.MD

All of this data is temporary and eventually deleted / overwritten.   
```  
**Cloudflare KV Workers:**
```
pullObject: // stored under key: prNumber, value: pullObject, at pulls namespace
    {  
      prLeaderboard: boolean, // showing if this pull was already accounted for in LEADERBOARD.md 
      prNumber: number,  // this is also a key, but temporarily storing this inside as well
      prAuthor: string,  
      prState: string,  //open, merged, closed...
      githubLink: string,  
      transactions: [   
        {  
          transactionSuccess: boolean,
          paidUsd: number,  
          paidKsm: number,  
          subscanLink: string,
          subscanHash: string,
        }
      ],
      prMergedDate: string,
      commits: number,
      linesAdded: number,
      linesRemoved: number
      }
    }

devObject: : // stored under key: devLogin, value: devObject, at devs namespace
   {
      assigned: [], // assigned issues
      finished: [], // assigned + merged
      unfinished: [], // assigned + unassigned
      droppedQueue: [], // queued up + dropped queue / option
      finishedStreak: [
        {
          timeOfAssignment, // time of assignment
          prPaidTime // time of payment
          issueNumber // number of concerned issue
          prNumber // number of concerned PR
          amountUsd // total amount USD received for issue
        }
      ]
    }
    
leaderboardObject: [ // stored under key: leaderboard, value: 
   leaderboardRecord: { // record summarizing all the pulls for dev
      devLogin,
      totalAmountReceivedUSD,
      totalAmountReceivedKSM,
      numberOfOpenPrs,
      mergedPrs,
      closedPrs,
      linesAdded,
      linesRemoved,
      numOfTotalCommitsMerged,
      linkToLastSubscan,
      lastMergedPrDate,
      commentsCount,
      linkedIssues
   }
]
```
