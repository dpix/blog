---
layout: post
title:  "Hosting images an easier way"
date:   2017-02-14 18:10:59 +0800
categories: jekyll update
---
# Hosting images an easier way

*This article describes how to set up an 
image service on AWS that can handle simple 
image hosting for SASS or other websites*

When it comes to user submitted images on websites, you can pretty quickly run into heaps of design issues.
* The user has uploaded a wierdly shaped image
* The images are massive and slow down the site
* How do I handle uploads of large images?
* Where do I store and serve all these images efficiently?

These are exactly the issues and questions I faced when building [Yabble](https://www.yabble.com.au). 
Yabble lists activities for kids holiday programs, and all those activities have images attached.
They then need to be displayed efficiently on in different sizes and shapes all over the website.

## Serving Images

The easiest solution I came up with for this was to store the images in an S3 bucket 
on AWS and host a separate image service on a subdomain.

I managed to find [this handy repo on github](https://github.com/jimmynicol/image-resizer) 
which serves up images direct from an S3 bucket and then can do image manipuations 
using the [sharp](https://github.com/lovell/sharp) library.

I then set up a simple Elastic Beanstalk instance running node on the cheapest option available,
set the appropriate environment variables: 
```javascript
//AWS keys
AWS_ACCESS_KEY_ID: <your_key>,
AWS_SECRET_ACCESS_KEY: <your_secret>,
AWS_REGION: <your_region>,
S3_BUCKET: <your_bucket>
```

and the image service was up and running. Now serving images at any width or height, configurable on the url.

eg: original image: https://images.yabble.com.au/activities/Jam-Sessions-Young-ones_41130

and image scaled to 200px wide: https://images.yabble.com.au/**w200**/activities/Jam-Sessions-Young-ones_41130

Perfect, this lets us scale any uploaded image to handle whatever is necessary for different parts of the site. Pop this behind a CDN like cloudflare and cache like crazy!

## Uploading images

This part is a little trickier, we now need to somehow send the images users upload to the site to our S3 bucket.

You could sift throught the terrible documentation for S3 multi part uploads and programmatically creating permissions and signing requests.
Or you could use another handy little library [EvaporateJS](https://github.com/TTLabs/EvaporateJS) that does most of this for you.

The code you will need for the front end will look something like this (ES6):

```javascript
import Evaporate from 'evaporate';
uploadImage = (file) => {
	return new Promise(resolve => {
		const evaporate = new Evaporate({
			signerUrl: '<Url to sign the response using your AWS secret key>',
			aws_key: '<AWS Key>',
			bucket: '<Your Bucket name>',
			aws_url: '<Region Url>'
		});

		const folder = `some_folder`
		evaporate.add({
				name: folder,
				file: file,
				progress: (progressValue) => console.log('Progress', progressValue),
				complete: (_xhr, awsKey) => { 
					console.log('Complete!'); 
					resolve([name])
				}
			});
	})
}
```

Evaporate puts together all the details for a request that can be signed with your AWS Secret key, and uploaded to your S3 bucket.
This requires a little bit of code on your backend to sign the request. In express on node this looks something like:

```javascript
var crypto = require('crypto');
var secret = 'your_secret_key';

module.exports = {
    get: (req, res) => {
        res.send(
            crypto
                .createHmac('sha1', secret)
                .update(req.query.to_sign)
                .digest('base64')
        );
    }
};
```

Evaporate can use that signed request to upload the file directly from the users browser to your S3 bucket without the image ever touching your backend. It also avoids having open write permissions on your bucket.
Obviously you would want to authenticate your users in your signing request so that this can't be abused.

From here you can use the folder and filename as the identifier for images in your database, and they can be retrieved by appending them to whatever url your image service is hosted on.

eg: **host:** https://images.yabble.com/, **id:** activities/Jam-Sessions-Young-ones_41130

## Summary
Now we have a solution that:
* cheaply and efficiently stores images in an S3 bucket (allows for simple backups also)
* hosts them through a simple node server on Elastic Beanstalk (it will spend very little cpu time too!)
* can be easily cache requests through a CDN like cloudflare
* uploads images easily without having to store or serve them locally
* can resize images to whatever dimensions for different parts of the site

I've had this setup running for the last 6 months and haven't had to touch it once. It uses barely any CPU on the EC2 instance and is super responsive.


