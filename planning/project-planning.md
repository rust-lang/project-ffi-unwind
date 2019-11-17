## Current action items

1. [x] Make a [draft charter](charter.md) for the project
1. [x] Create a Zulip channel
1. [x] Close RFCs #2753 and #2699 and point to this repository and the Zulip channel
1. [ ] Create design documents and drafts in this repo
   * create a "sketch" document to start, fill it in over time
1. [ ] Create issues for the outstanding concerns
   * lock issues that are not the current focus topic?
   * unsafe code guidelines just used "focus" labels, maybe do that
   * or maybe use time-based issues (woah!) where the sync meeting marks a
     "pause" to collect, incorporate new findings, and set new topics
1. [ ] Open a new RFC for `extern "C unwind"`

## Goals

* Initial:
  * complete the "C unwind" RFC (a clean start to #2753)
  * refine charter to create a roadmap
    * what are the bits of unspecified behavior yet to be fixed
    * which use cases are we aiming at and which are non-goals?
    * which bits of unspecified behavior must be specified
* Subsequent:
  * work through those bits of unspecified behavior and work toward a final design

## Meta

* Have a sync meeting every week or maybe 2 weeks
  * all are welcome to come, hold on Zulip (?) or Zoom
  * discuss the conversations that occured over last period
  * try to summarize new points that were raised and add to an existing document that contains the "pros/cons" 
    * when this document seems to be at a steady state, time for a decision
    * make our proposal and bring it to the lang team
    * announce result and give some time, then add to FAQ
  * prep a summary to bring to lang team and post publicly
    * Idea: post summary as a new RFC, then close the old one with a link to
      the new one? Keep RFC threads short and time-boxed
* Membership:
    * Avoid having a "full membership" concept, as Niko is not sure that it's
      needed here
    * But if we were going to have members, it should be reserved for folks who
      have "productively contributed" to several design discussions
