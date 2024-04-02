---
title: Midnight appointments across timezones
date: 2022-05-04 20:26:00 +0600
categories: [Experiences]
tags: [timezone, UTC]
---

> **When one's availability today translates to another's tomorrow.**

![Desktop View](/assets/img/bg/richard-tao-iM5bGdqB2Gg-unsplash.jpg){: w="700" h="400" }
_Photo by <a href="https://unsplash.com/@richardtao28?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Richard Tao</a> on <a href="https://unsplash.com/photos/white-and-black-kanji-text-signage-iM5bGdqB2Gg?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a>_
  

## Premise
A certain appointment scheduling feature allows a user (the host) to make themselves available during certain periods of their day in a week. Another user (the guest) from a different timezone wants to know the hours in their own timezone on which the host user is available. They may want to see these available hours for a date range, say from a Tuesday to a Thursday. Let’s think about the API I/O.

* _Input:_ start date, end date (in the guest’s timezone)
* _Output:_ a list of the appointment time slots that the host is available on (in the guest’s timezone)

## Setting up a use-case
The host is in Los Angeles, CA, USA. The guest is in Dhaka, Bangladesh. The appointment duration is 1 hour. Suppose the host makes herself available from `0800 to 1200` (_Pacific Standard Time, tz: America/Los_Angeles_) on the following dates:

* March 29, 2022
* March 30, 2022

The guest in Dhaka (_Asia/Dhaka, UTC+6_) wants to know the host’s available hours on the following days of their own calendar:

* March 30, 2022

Assuming Los Angeles is 14 hours behind Dhaka, our guest should see the following appointment slots for March 30, 2022 (_UTC+6_):

* 0000-0100 (equivalent to 1000-1100 in LA on March 29, 2022)
* 0100-0200 (equivalent to 1100-1200 in LA on March 29, 2022)
* 2200-2300 (equivalent to 0800-0900 in LA on March 30, 2022)
* 2300-2400 (equivalent to 0900-1000 in LA on March 30, 2022)

## The formula
The guest sends the following input to the API:

* start_date: March 30, 2022
* end_date: March 30, 2022

The API needs to translate the given date range (in the guest's timezone) to a date range in the host's zone. The API subtracts or adds hours to the _start_date_ of the guest depending upon whether the host is behind or ahead of the guest's timezone. So, to the API, the start date becomes, in this case: `March 30, 2022 at 0000 hours` – `14 hours` = `March 29, 2022 at 1000 hours`.

For the _end_date_ the formula is slightly different: `March 30, 2022 at 0000 hours` + `24 hours` – `14 hours` = `March 30, 2022 at 1000 hours`.

So, now (_in Pacific Standard Time, America/Los_Angeles_) all that the API has to do is, find appointment slots between: `March 29, 2022 at 1000 hours` and `March 30, 2022 at 1000 hours`.