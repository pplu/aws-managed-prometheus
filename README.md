# Understanding Amazon Managed Prometheus service

Amazon Managed Prometheus (AMP) service is in Public preview, meaning that we can use it today. Lets have a
look at what this service provides.

If you've ever used Prometheus, you will know that Prometheus is a modern monitoring system. When you
run Prometheus, it will poll your systems, recovering metrics from them. Normally, when we configure Prometheus, 
we take care of the polling configuration (what metrics to gather). Underneath, Prometheus has a metrics 
database where it stores all the data that it gathers. Other systems (like the alerting subsystem) will poll
the database to see if any of the configured thresholds are matched, producing an alert.

## Simple Prometheus architectures

You may be surprised that Amazon Managed Prometheus service is not about the polling part of Prometheus. It is
about the storage part of Prometheus. The modular design of Prometheus lets the data gathering happen on one system, 
and the storage on another. By default, when we run Prometheus, it will store all data on the local file system of 
the machine where it is running. In the Cloud world we like the concept of stateless / immutable systems. Storing all
the metrics data on the local filesystem impedes applying this pattern, making us have to take care of managing
disk space, I/O, etc on the instance running Prometheus.

AMP really helps us manage our monitoring solution in a different way, since it enables us to write our monitoring metrics 
in an external, managed, Prometheus database.

## I thought AMP was for Kubernetes only

The AWS documentation for AMP right now only talks about configuring AMP for Kubernetes. I suppose that was their
top priority, so getting your non-Kubernetes Prometheus setup to use AMP can be a bit confusing, but the hints
are there. So lets start.

## AMP Workspaces

Each AMP Workspace that you create is an independent Prometheus database. Each workspace has a unique id of the form
`ws-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX`. Workspaces can have an alias (a non-unique name) to identify them.

## Configuring Prometheus to write to AMP

Prometheus has a `remote_write` configuration element in the `prometheus.yml` file:

```
remote_write:
  url: ...
```

Your first impulse may be to put the remote write URL of your workspace (this can be found in the Summary section
of your AMP workspace). This URL is something like `https://aps-workspaces.us-east-1.amazonaws.com/workspaces/ws-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX/api/v1/remote_write`.

The problem here is that the remote write URL is a public endpoint authenticated with IAM via AWSv4 signatures, and Prometheus 
doesn't know how to sign remote write requests.

To get Prometheus to write to the AMP service we will make it think it is talking to an unauthenticated remote write service, that
will just sign the requests, and send them to AMP. We will use the AWS sigv4-proxy:

Run the sigv4-proxy on your Prometheus machine:

```
docker run -d --network=host --name promsigner public.ecr.aws/aws-observability/aws-sigv4-proxy:1.0 --name aps --region us-east-1 --host aps-workspaces.us-east-1.amazonaws.com --port :8005
```

Note: The AMP service is refered to as `aps` in IAM and uses a `aps-workspaces` name in its endpoint. This can be a bit confusing at times, and it's not a typo.

The signing proxy need some type of IAM credentials to work. Configuring an IAM Role for the instance is the best solution in this case (maybe you already have an
instance role configure for Prometheus to call the EC2 API, for example).

```
PolicyName: prometheusdatastore
PolicyDocument:
 Version: "2012-10-17"
 Statement:
 - Sid: AllowRemote
   Effect: Allow
   Action: [ "aps:GetLabels", "aps:GetMetricdata", "aps:GetSeries", "aps:QueryMetrics", "aps:RemoteWrite" ]
   Resource: "*"
```

Now we will configure the `remote_write` section of the Prometheus configuration:

```
remote_write:
- url: 'http://localhost:8005/workspaces/AMP_WORKSPACE_ID/api/v1/remote_write'
```
Remember to substitute the `AMP_WORKSPACE_ID` for the Id of your workspace.

After that, Prometheus will start writing to AMP.

## Getting data out of Prometheus

A very typical setup is to have a Grafana server for building our dashboards (since the Prometheus
web interface is very limited).

Lets see how to get Grafana to read from AMP:

In this case, Grafana knows how to authenticate requests to the AMP endpoints, but you have to
activate this option (it won't appear in the Grafana datasource configuration interface):

In `grafana.ini` add:

```
[aws]
allowed_auth_providers=default,ec2_iam_role
[auth]
sigv4_auth_enabled=true
```

This will make a new option appear in the Prometheus datasource configuration for Grafana: 
`SigV4 auth`. Activate this authentication option, and configure the default region in the 
`SigV4 Auth Details` section.

In `URL`, introduce the endpoint of your AMP Workspace: `https://aps-workspaces.us-east-1.amazonaws.com/workspaces/ws-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX/`
 (note that the URL doesn't have the trailing `api/` part that the write and the query endpoints have).

Note that optionally we could not configure the AWSv4 auth in Grafana, pointing the datasource URL to 
`http://localhost:8005/workspaces/AMP_WORKSPACE_ID/` (note the http at the start), so Grafana would also 
use the sigv4 proxy, but I think it's better that Grafana handles everything natively.

## Is it working?

By now your Prometheus should be pushing its metrics to AMP, and Grafana should be reading those metrics. 

## Going the extra mile

By now you should be capable, with a bit more work to adapt your monitoring instance to immutability. On instance
startup, you can pull config files and start Prometheus and Grafana. You will need to investigate:

 - Grafana datasource provisioning
 - Grafana dashboard provisioning

That is left as an excercise to the reader. After doing this excercise, you will find that you can simply
throw away instances with old configurations, starting new instances with new configuration. Your monitoring
infrastructure will also be more robust thanks to the fact that Autoscaling can substitute your failed 
monitoring instance.

## No AMP Resources in CloudFormation

As of this writing, there was no CloudFormation support for creating AMP Workspaces via CloudFormation. I've built a
[CloudFormation Custom Resource](amp.yaml) to fill that gap.

# Conclusions

Although it is not explicitly documented, we can get AMP to work outside of a Kubernetes environment and get
the benefits of not having to manage the metrics storage of Prometheus.

There are, though, a couple of things to remark:

 - sigv4-proxy is only distributed as a Docker container. Being a stand-alone Go program, It would be useful
   that AWS also distribute it in non-docker format also.
 - I hope AWS gets v4 signatures support into Prometheus to make things simpler (no need for v4sig-proxy).
 - No native CloudFormation support for AMP: services should have CloudFormation support at Launch. The 
   community can "fill in gaps", but it is increasingly frustrating to work on things that AWS should do.
 - You can look at the Managed Grafana Enterprise AWS Offering, but it will probably be more expensive
   than hosting your own server with Grafana Community and a small instance.

# Author, Copyright and License

This article was authored by Jose Luis Martinez Torres.

This article is (c) 2021 Jose Luis Martinez Torres, Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

The canonical, up-to-date source is [GitHub](https://github.com/pplu/aws-managed-prometheus). Feel free to contribute back.
