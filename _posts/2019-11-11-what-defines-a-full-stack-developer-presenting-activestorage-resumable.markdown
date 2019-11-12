---
layout: post
title:  What defines a full-stack developer (Presenting ActiveStorage Resumable)?
date:   2019-11-11 22:25:00 -0300
categories: programming
tags: gcs google-cloud-storage resumable upload rails activestorage full-stack
---
The sadness on his eyes, for sure :)

This post is not about definitions, internet is full of definitions about full-stack developers, it's about how sad the
life of a full-stack developer is! If you don't want to read the blah blah blah, just see how easy it's to use
[ActiveStorage Resumable](https://github.com/fnix/activestorage-resumable)

Let's talk about resumable uploads from browser?

Take a look at the [Google Cloud Storage documentation](https://cloud.google.com/storage/docs/json_api/v1/how-tos/resumable-upload),
don't appear to be very scary, right? Give a try and implement resumable uploads, you will feel the pain! If you
think I'm being pessimistic, just stay with me.

Is there any front-end solution for resumable uploads? I will present one for ActiveStorage, but I will also say a word
about other solutions.

* [tus.io](https://tus.io/)
* [Shrine](https://github.com/shrinerb/shrine/wiki/Adding-Resumable-Uploads)
* [gcs-browser-upload](https://www.npmjs.com/package/gcs-browser-upload)
* [ActiveStorage](https://guides.rubyonrails.org/active_storage_overview.html)

tus.io and Shrine are very interesting, but they need an upload server, a thing that I really want to avoid. Why
maintain an upload server when all the major cloud storage solutions support direct uploads? I think it's too much work
for a simple feature. There are some
[tutorials](https://github.com/janko/tus-ruby-server#streaming-web-server) using the
[Falcon](https://github.com/socketry/falcon) webserver, but this path has a very bad smell. Are you going this way only
because you are lazy enough to come up with a solution to send it directly to the cloud and take the risk of overloading
your server with upload requests? No, thanks!

I didn't mention about [gcs-resumable-upload](https://github.com/googleapis/gcs-resumable-upload), but it's appear to be
projected to be used in the back-end, or is it just my lack of experience with front-end tools?

gcs-browser-upload appear to be the perfect tool for the job, take a look at it's workflow:

1. User selects a file
2. File + a [Google Cloud Storage resumable upload URL](https://cloud.google.com/storage/docs/json_api/v1/how-tos/upload#resumable)
   are given to gcs-browser-upload
3. File is read in chunks
4. A checksum of each chunk is stored in localStorage once succesfully uploaded
5. If the page is closed and re-opened for some reason, the upload can be resumed by passing the same file and URL back
   to gcs-browser-upload. The file will be validated against the stored chunk checksums to work out if the file is the
   same and where to resume from.
6. Once the resume index has been found, gcs-browser-upload will continue uploading from where it left off.
7. At any time, the pause method can be called to delay uploading the remaining chunks. The current chunk will be
   finished. unpause can then be used to continue uploading the remaining chunks.

This is a pretty solution for uploads that will last for a long time, so I decided to sticky with it and look at it's
code. For my surprise it's small and relative simple to uderstand. I just copy and paste it inside my project and get it
running, because I was [unable to add it from the npm package](https://stackoverflow.com/questions/57532710/why-import-upload-from-gcs-browser-upload-is-failing)!

Ok, how can I generate a resumable upload URL? Looking to Google documentation there is only an example with OAuth
token. I'm using [google-cloud-storage gem](https://rubygems.org/gems/google-cloud-storage), so after googling a bit
I find a [very useful issue comment](https://github.com/googleapis/google-cloud-ruby/issues/2024#issuecomment-379402604)
and a [PR that adds V4 signed URLs](https://github.com/googleapis/google-cloud-ruby/pull/3057). In this PR you can view
some [tests that uses v4 signed URLs for resumable uploads](https://github.com/googleapis/google-cloud-ruby/pull/3057/files#diff-56ad5b547c2baf68910ecdebce23288bR117).

Fantastic, I just have to upload now! Everything going well, my bits all flowing to GCS, part by part and in the last
chunk, boom! A CORS error, but why only in the last chunk? What the hell I did wrong? After a long time validating all
the steps, I find it: I need to put the origin while creating the resumable session URI. This is the last point of item
5 of [Initiating a resumable upload session](https://cloud.google.com/storage/docs/json_api/v1/how-tos/resumable-upload#start-resumable)

I just can't find a way to describe how painful it is to discover that the problem is with the start of the road when
you are almost at the end. So much context changed that it's really difficult to realize that the problem was with an
URL you generated a long time ago. Please, Google, fix it!

*Since we are talking about browsers, remember that CORS can bite at any point, just breath and everything will work!*

Sorry, I was upset, here is the code to generate a resumable URI (pay attention to the 'Origin' header):

{% highlight ruby %}
require "google/cloud/storage"

storage = Google::Cloud::Storage.new(
  project_id: "my-project",
  credentials: "/path/to/keyfile.json"
)

bucket = storage.bucket "task-attachments"
signed_url = bucket.signed_url(key, method: 'PUT', version: :v4)

uri = URI.parse(signed_url)
https = Net::HTTP.new(uri.host, uri.port)
https.use_ssl = true

headers = {
  'Origin': 'https://example.com',
  'x-goog-resumable': 'start'
}
request = Net::HTTP::Put.new(uri.request_uri, headers)
response = https.request(request)
response['location'] # this is the resumable session URI, you can use it to start or resume an upload
{% endhighlight %}

It's appear so simple, right? And indeed it is! I hate these kind of headbanging!

Are you still here? Did you still remember I talk about ActiveStorage? It was only to get an improved page hank! Just
kidding!

ActiveStorage already implements direct upload for Amazon S3, Google Cloud Storage, and Microsoft Azure Storage. Direct
upload is not that different from resumable ones. The implementation of the last is a bit more complex for obvious
reasons: you need a way to track progress!

How ActiveStorage works for direct uploads? In a simple way, when the form is submitted a request is made to your Rails
application for a direct upload URL, this process creates a new ActiveStorage::Blob that have a uniq
[signed_id](https://api.rubyonrails.org/classes/ActiveStorage/Blob.html#method-i-signed_id) which is returned along with
the URL to make the upload.

When the direct upload finishes, the signed_id is submitted along with the form. ActiveStorage uses this information to
find the Blob and creates an ActiveStorage::Attachment that keeps the link between your model and the blob. Now you have
a persistent blob in your application! :tada:

The idea with resumable uploads is that you can upload BIIIIG files, using an unreliable connection or just resuming
later because you want to close your notebook. The difference in this process is that Google give us an URL that is
valid for 7 days.

One of the things from ActiveStorage that isn't suitable for resumable uploads is how it deals with checksum. Let's
detail a little more the behavior of ActiveStorage:

1. The checksum of the ENTIRE file is calculated
2. A request is made to the server with checksum, and some other metadata (filesize, mime, etc)
   1. A (ActiveStorage::)Blob is generated with these info
   2. Returns a blob identifier and URL of the cloud service to make a direct upload
3. Uploads the file to the returned URL
4. Submits the form to the application containing the blob identification
5. ActiveStorage creates the relationship of the blob and your model through an (ActiveStorage::)Attachment

For big files, step 1 means a loooong wait. gcs-browser-upload already solved this problem (partially to be honest,
I created a [fork](https://github.com/fnix/gcs-browser-upload/blob/master/CHANGELOG.md) to deal with some issues I find
in the way, because the original one became
[unmaintained](https://github.com/QubitProducts/gcs-browser-upload/issues/16#issuecomment-483160021)).

This is how we tracked checksum and keep the behavior as near to ActiveStorage as possible, this outlines the most
significant differences:

* We only calculate the checksum for the first 10mb of the file in step 1
* Steps 2 and 3 uses a resumable URL
* In step 3, the upload is done by [@fnix/gcs-browser-upload](https://github.com/fnix/gcs-browser-upload), that
  calculates the rest of the checksum on the fly (comparing with the checksum returned by GCS).
* Before executing step 4, we make a call to the application to update the blob checksum. Yeah, we could send the
  checksum with the blob identifcation of step 4, but this would require modifications in core parts of ActiveStorage
  models, a thing I want to avoid.

Why 10mb for the checksum? It's fast to calculate and I think that it's a good amount of data to be sure we are dealing
with the same file. Change my mind!

Another aspect of ActiveStorage that bothers me for resumable uploads is the analyzer. It's download the entire file to
extract some metadata, like width, height, duration, angle and aspect ratio. When dealing with big files the cost to
transfer tons of gigabytes to your application can be
[pretty high](https://cloud.google.com/storage/pricing#network-egress). You can
[manipulate the analyzers](https://api.rubyonrails.org/classes/ActiveStorage/Blob/Analyzable.html) or even create an
initializer with `Rails.application.config.active_storage.analyzers = []` if you don't need any analyzer.

I decided to keep the same JS module bundler of ActiveStorage, [rollupjs](https://rollupjs.org), to re-use the same
configuration, but @fnix/gcs-browser-upload uses async/await and the current configuration don't generate proper code
for the browser. I really get some bad hours of trial-and-error tweaking plugins and its configurations to generate a
functional browser JavaScript. In @fnix/gcs-browser-upload I use babel, the tool from the original codebase. In my Rails
applications I use webpacker. The JavaScript fragmentation is a virtue or a defect? For me it's dauting, the more deep I
go with JS, the more I appreciate this
["old" article](https://hackernoon.com/how-it-feels-to-learn-javascript-in-2016-d3a717dd577f)!

In the short term I don't intend to support Amazon S3 nor Azure Storage, but would be glad to receive PRs.

I hope that [ActiveStorage Resumable](https://github.com/fnix/activestorage-resumable) is useful for you as it's being
useful for me.
