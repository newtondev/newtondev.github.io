---
title: How to run a Kubernetes cron job manually as a one time job
date: 2020-05-28 12:00:00
categories: [kubernetes]
tags: [kubernetes,cron]
---

![How to run a Kubernetes cron job manually as a one time job](/assets/img/2020/05/ocean_swell.jpg)

I recently had to create a cron job in Kubernetes that would run a data archive script that would archive the immutable MySQL table data for transactions for the previous day and upload the result to AWS S3.

The job failed today as there was a network connectivity issue to the database and my code tried 3 times and then gave up for today. I then needed to have the job execute so I could have it process yesterdays data and archive it. So I did what I normally do, I take the YAML for the cronjob and modify it to become a standard type: `Job` and then I just apply the YAML to run the job and once completed I just run `kubectl delete -f my-onetime-job.yaml`.

It works but seems a bit tedious. Recently I discovered how to do this without needing a YAML file, I could just run it from the command line directly.

Here is the command I used:
```shell
kubectl create job --from=cronjob/s3-transactions-data-archiver s3-transactions-data-archiver-otj
```

Where with placeholders:
```shell
kubectl create job --from=cronjob/<name-of-cron-job> <name-of-job>
```

This will create a job and run it, you can verify it’s completion by running the following command and inspecting the output:

```shell
kubectl get jobs
```

The output will show you the jobs you currently have running and my job for `s3-transactions-data-archiver-otj` was listed there as completed.

To cleanup after all we now need to do is delete the job by running the command:
```shell
kubectl delete job s3-transactions-data-archiver-otj
```

Where with placeholders:
```shell
kubectl delete job <name of job>
```

This is a great way to quickly run a failed cron job, or even if you create your cron job and it does not immediately run, you can trigger this manual run.

Hope this helps you and thank you for reading this snippet.

Medium Article: [https://medium.com/swlh/how-to-run-a-kubernetes-cron-job-manually-as-a-one-time-job-5622f1d6a3b0](https://medium.com/swlh/how-to-run-a-kubernetes-cron-job-manually-as-a-one-time-job-5622f1d6a3b0)