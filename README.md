# npm-register [![CircleCI](https://circleci.com/gh/dickeyxxx/npm-register/tree/master.svg?style=svg)](https://circleci.com/gh/dickeyxxx/npm-register/tree/master)

Your own private npm registry and backup server. Used in production and maintained by Heroku. Designed to be easy to set up and maintain, performant, and stable.

[![Code Climate](https://codeclimate.com/github/dickeyxxx/npm-register/badges/gpa.svg)](https://codeclimate.com/github/dickeyxxx/npm-register)
[![codecov](https://codecov.io/gh/dickeyxxx/npm-register/branch/master/graph/badge.svg)](https://codecov.io/gh/dickeyxxx/npm-register)

Overview
--------

This project allows you to have your own npm registry. This server works with the necessary `npm` commands just like the npmjs.org registry. You can use it to not worry about npm going down or to store your private packages. It performs much faster than npmjs.org and can even be matched with a CDN like Cloudfront to be fast globally.

Rather than trying to copy all the data in npm, this acts more like a proxy. While npm is up, it will cache package data in S3. If npm goes down, it will deliver whatever is available in the cache. This means it won't be a fully comprehensive backup of npm, but you will be able to access anything you accessed before. This makes it easy to set up since you don't need to mirror the entire registry. Any packages previously accessed will be available. Storing the data in S3 makes npm-register easy to maintain since any time a server acts up, you can simply blow it away and provision a new one with the same S3 credentials.

The inspiration for this project comes from [sinopia](https://github.com/rlidwka/sinopia). This came out of a need for better cache, CDN, and general performance as well as stability of being able to run multiple instances without depending on a local filesystem.

This is also a [12 Factor](http://12factor.net/) app to make it easy to host on a PaaS like Heroku or in a custom Ansible/Chef/Puppet cluster.

Setup
-----

The bulk of the data is stored in S3. You will need to set the `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_S3_BUCKET` environment variables.

If `REDIS_URL` is set (optional) redis will be used to cache package data.

The easiest way to set this up is with the Heroku button:

[![Deploy to Heroku](https://www.herokucdn.com/deploy/button.png)](https://heroku.com/deploy)

Alternatively, you can set it up from npm:

```
$ npm install -g npm-register
$ npm-register
```

Either way, your registry is now setup and you should be able to test it by updating the packages with it:

```
$ npm update --registry http://urltomyregistry
```

See below for how to enable authorization and `npm publish`.

How it works
------------

Essentially the goal of the project is to quickly deliver current npm data even when npm is offline.  In npm there are 2 main types of requests: package metadata and tarballs.

Package metadata mostly contains what versions of a package are available. These cannot be cached for very long since the package can be updated. By default, it is cached for 60 seconds. You can modify this with `CACHE_PACKAGE_TTL`. Etags are also supported and cached to further speed up access.

The tarballs are the actual code and never change once they are uploaded (though they can be removed via unpublishing). These are downloaded one time from npmjs.org per package and version, stored in S3 and in the local tmp folder for future requests. These have a very long max-age header.

In the event npmjs.org is offline, npm-register will use the most recent package metadata that was requested from npmjs.org until it comes back online.

npm commands supported
----------------------

* `npm install`
* `npm update`
* `npm login`
* `npm whoami`
* `npm publish`

Authentication
--------------

npm-register uses an htpasswd file in S3 for authentication and stores tokens in S3. To set this up, first create an htpasswd file. Then upload it to `/htpasswd` in your S3 bucket. Use aws-cli.

```
$ aws s3 cp s3://S3BUCKET/htpasswd ./htpasswd
$ htpasswd -nB YOURUSERNAME >> ./htpasswd
$ aws s3 cp ./htpasswd s3://S3BUCKET/htpasswd
```

Then you can login with npm. Note that the email is ignored by the server, but the CLI will force you to add one.

```
$ npm login --registry http://myregistry
Username: dickeyxxx
Password:
Email: (this IS public) jeff@heroku.com
$ npm whoami --registry http://myregistry
dickeyxxx
```

This stores the credentials in `~/.npmrc`. You can now use `npm publish` to publish packages.

**NOTE**: Because the original use-case for having private packages was a little strange, right now you need to be authenticated to upload a private package, but once they are in the registry anyone can install them (but they would have to know the name of it). Comment on https://github.com/dickeyxxx/npm-register/issues/1 if you'd like to see better functionality around this.
