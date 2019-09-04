---
layout: post
title:  "Presenting ActiveStorage Resumable or what defines a full stack developer?"
date:   2019-08-18 12:36:57 -0300
categories: rails programming full-stack
---
The sadness on his eyes, for sure :)

This post is not about definitions, internet is full of definitions about full stack developers, it's about how sad the
life of a full stack developer is!

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

tus.io and Shrine are very interesting, but they need an upload server, a thing that I really want to avoid. I find
these solutions too much work for a simple feature. There are some
[tutorials](https://github.com/janko/tus-ruby-server#streaming-web-server) using the
[Falcon](https://github.com/socketry/falcon) webserver, but this path has a very bad smell. No, thanks!

I didn't mention about [gcs-resumable-upload](https://github.com/googleapis/gcs-resumable-upload), but it's appear to be
projected to be used in the back-end, or is just my lack of experience with front-end tools? I just asked for help to
[include](https://stackoverflow.com/q/57532710/977201) gcs-browser-upload in my project (it's the third item of my list,
I'm not talking about gcs-resumable-upload anymore :-). Webpacker is really a life saver!

gcs-browser-upload appear to be the perfect tool for the job, take a look at it's workflow:

1. User selects a file
2. File + a [Google Cloud Storage resumable upload URL](https://cloud.google.com/storage/docs/json_api/v1/how-tos/upload#resumable) are given to gcs-browser-upload
3. File is read in chunks
4. A checksum of each chunk is stored in localStorage once succesfully uploaded
5. If the page is closed and re-opened for some reason, the upload can be resumed by passing the same file and URL back to gcs-browser-upload. The file will be validated against the stored chunk checksums to work out if the file is the same and where to resume from.
6. Once the resume index has been found, gcs-browser-upload will continue uploading from where it left off.
7. At any time, the pause method can be called to delay uploading the remaining chunks. The current chunk will be finished. unpause can then be used to continue uploading the remaining chunks.

This is quite some work, so I decided to sticky with it and look at it's code. For my surprise it's small and relative
simple to uderstand. I just copy and paste it inside my project and get it running!

Ok, how can I generate a resumable upload URL? Looking to Google documentation there is only an example with OAuth
token. I'm using [google-cloud-storage gem](https://rubygems.org/gems/google-cloud-storage), so after googling a bit
I find a [very useful issue comment](https://github.com/googleapis/google-cloud-ruby/issues/2024#issuecomment-379402604)
and a [PR that adds V4 signed URLs](https://github.com/googleapis/google-cloud-ruby/pull/3057). In this PR you can view
some [tests that uses v4 signed URLs for resumable uploads](https://github.com/googleapis/google-cloud-ruby/pull/3057/files#diff-56ad5b547c2baf68910ecdebce23288bR117).

Fantastic, I just have to upload now! Everything going well, my bits all flowing to GCS, part by part

Nice, after a long time to figure it out, I forgot about the last item, of item 5, of topic
[Initiating a resumable upload session](https://cloud.google.com/storage/docs/json_api/v1/how-tos/resumable-upload#start-resumable):

* Origin, if you have enabled Cross-Origin Resource Sharing. You must also use this header in subsequent upload requests.

I just can't find a way to describe how painful it was to discover that CORS where the problem, I know that I'm in a
browser, but Google really doesn't helped here. Everything worked pretty well, from the creation of the URL, to the
parts submission, to fail with the last chunk, yeah the last chunk! 

Since we are talking about browsers, never forget about CORS! The coolest part is that everything works, only the last
chunk fails, thanks Google to blame me only in the end of the road!

So, here is the code to generate a resumable URI (pay attention to the 'Origin' header):

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

Are you still here? Did you still remember I talk about ActiveStorage? It was only to get an improved page hank, just
kidding!

ActiveStorage already implements direct upload for Amazon S3, Google Cloud Storage, and Microsoft Azure Storage. Direct
upload is not that different from resumable ones. The implementation of the last is a bit more complex for obvious
reasons: you need to know what you need to resume!

How ActiveStorage works for direct uploads? In it's simplest form, when the form is submitted it's request the back-end
for a direct upload URL, this process creates a new ActiveStorage::Blob, and one of the attributes returned to the
front-end is the [signed_id](https://api.rubyonrails.org/classes/ActiveStorage/Blob.html#method-i-signed_id).

When the direct upload finishes, the signed_id is submitted with the form, ActiveStorage uses it's valeu to find the
Blob and creates an ActiveStorage::Attachments that keeps the link between your model and the blob.

The idea with resumable uploads is that you can upload BIIIIG files, using an unreliable connection, or just resuming
later because you want to close your notebook. Google give us an URL that is valid for 7 days, so we must ignore
resumable blobs older than 7 days. Good, we are approaching a solution!

One of the things from ActiveStorage that I don't want to use for resumable uploads is checksum, at least not in the
same way. It calculates in response for the submit, for big files, it's a loooong wait. gcs-browser-upload already
solved this problem (partially to be honest, but the tools for the proper solution are all there), so we can be sure
that the file uploaded successfully.

What gcs-browser-upload misses is really a small, but very important thing: compare the calculated checksum with the
checksum GCS return for the part received. If they differ we must re-send the part.

When the user decide to resume an upload, we must find the proper blob and continue our work of submitting parts. We
must avoid user errors, so we need some kind of checking the file input and the blob we already have, there isn't any
sense in continue a Manjaro ISO upload using a Ubuntu ISO.

For resumable uploads I decided to checksum the file using the following data:

* Content type
* File size
* The checksum of the first 10mb of the file

Why 10mb? I don't know, I just need a value, right? It's fast to calculate and I think that is a good amount of data to
be sure we are dealing with the same file. Change my mind!

I also decided to add exponential backoff for when a transmission fail. [p-retry](https://www.npmjs.com/package/p-retry)
was the lib that appeals more for me, and after using it a bit in [RunKit](https://npm.runkit.com/p-retry) integrates it
with webpacker can't be that hard, right?

* `yarn add p-retry`
* adds `import * as pRetry from 'p-retry'` to my JS
* `TypeError: "exports" is read-only` in browser console

After more headbanging I find solution that worked: https://medium.com/@po_thiago/cannot-assign-to-read-only-property-exports-of-object-object-1d908589e135. Nice, now that it worked, I will try to
understand why it worked, can't be that hard, right? I find the
[sourceType documentation](https://babeljs.io/docs/en/options#sourcetype) good, but I just don't have so much background
to understand all that. All this fragmentation is a virtue or a defect? It's dauting, the more deep I go with JS, the
more I appreciate this ["old" article](https://hackernoon.com/how-it-feels-to-learn-javascript-in-2016-d3a717dd577f)!

The only thing I added to ActiveStorage::Blob was a `resumable_url:string` attribute. For last, before searching for a
resumable URL, I first delete all blobs with expired URLs (older than 7 days).

In the short term I don't intend to support Amazon S3 nor Azure Storage, but would be glad to receive PRs.

I hope that ActiveStorage Resumable is useful for you as it's being useful form me.
