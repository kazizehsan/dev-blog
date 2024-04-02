---
title: Maximizing asynchronous OCR operations with Tesseract and Quartz
date: 2019-12-12 17:55:00 +0600
categories: [Experiences]
tags: [tesseract-ocr, OCR, concurrency, quartz, java, multi-threading, scheduler, job]
---

![Desktop View](/assets/img/bg/sear-greyson-K-ZsC7YdJ6Y-unsplash.jpg){: w="700" h="400" }
_Photo by <a href="https://unsplash.com/@seargreyson?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Sear Greyson</a> on <a href="https://unsplash.com/photos/brown-binder-lot-K-ZsC7YdJ6Y?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a>_
  

## The Objective
The application has to let a user upload multiple PDFs together. And on each of those PDFs, an Optical Character Recognition (OCR) operation needs to be run to glean photos and some information.

## Restrictions

1. The application is deployed on-premise, on a machine that is restricted from connecting to the internet while the software functions.
2. The machine is limited to 4 CPU cores.

## Tesseract and the catch
We decided to use [Tesseract](https://tesseract-ocr.github.io/) to help us out with the OCR. Our server application was written in Spring (Java), from which a wrapper ([Tess4j](https://mvnrepository.com/artifact/net.sourceforge.tess4j)) would invoke the `tesseract-ocr` engine.

Our original plan was to let `tesseract-ocr` manage its own multithreading to get a PDF OCRed as quickly as possible, and then move on to the next one in the queue of uploaded PDFs. But tesseract-ocr in multithread mode was [significantly slower](https://github.com/tesseract-ocr/tesseract/issues/3109) than in single-thread mode at the time this application was being made.

So we forced each spawned process of `tesseract-ocr` to use one thread only by setting `OMP_THREAD_LIMIT=1` in the environment. But now, it would be great if we could launch 4 of those processes together to get through the PDFs faster.

## Quartz the Scheduler
[Quartz](http://www.quartz-scheduler.org/) allows us to create jobs and then run those jobs concurrently if needed. So, every time a PDF was successfully uploaded synchronously at the request of the user, we scheduled a job for it. This asynchronous job would actually invoke the `tesseract-ocr`. When done with a PDF, the job updates a record on our database so that the user can learn about the OCR completion.

We told Quartz to keep it to 4 concurrent jobs at maximum. And this combination of single-threaded Tesseract and a multi-threaded Quartz, was the sweet spot for our application.