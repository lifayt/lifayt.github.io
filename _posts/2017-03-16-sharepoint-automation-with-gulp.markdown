---
layout: post
title: "Sharepoint Automation with Gulp"
date: 2017-03-16 09:50:50 -0400
categories: blog
---

### Sharepoint Workflows
Typically small scriptlet or widget development requiring only one or two js/css files is done in Sharepoint on the fly. The files are uploaded to a document library in Sharepoint, and then development is easily accomplished by opening the directory directly from a text editor or ide. Alternately, you can keep the relevant document library open in Windows Explorer or on the web, and files can be dragged and dropped whenever changes are made. 

For projects only requiring a few files, this is an acceptable (although occasionally annoying) process. A widget that only needs a little bit of handwritten CSS/JS bolted onto a small HTML file doesn't require any sort of industrial strength deployment pipeline. That changes once we start getting into medium sized projects that come with their own dependencies and complexities. If you want to use a css pre-processor, or bundle your javascript files together, or deploy a whole directory structure at once (or worse, parts of a directory structure) - the manual process or save-as-you-go process can become cumbersome. 

### Adding Automation
I wanted to automate the entire process involved with moving my files to Sharepoint whenever I make development changes to the documents. To that end, I started looking at the task automation ecosystem. 

The two most common choices are [Gulp](http://gulpjs.com) and [Grunt](https://gruntjs.com). I settled on Gulp for a couple of reasons: One, the task file structures in Gulp more closely mirrored my own thinking and two, I found some great Sharepoint specific libraries that had Gulp integration. 

Setting up gulp is easy enough - you just install gulp cli tooling via npm, and then add gulp and a gulpfile to your local directory. Gulpfiles are straightforward to program, and I'll have some good examples later on in this post. 

### Sharepoint Deployment

The hardest piece of the puzzle was figuring out how to automatically deploy my files to a Sharepoint Document Library. There are some good resources on MSDN for how to automate file movement, but at a cursory glance it seemed like most of it was aimed at users who want to do document automation from within Sharepoint. Thankfully, I stumbled across the following three libraries which make the next part a breeze. 

All three libraries are developed by GitHub user [s-KaiNet](https://github.com/s-KaiNet), and I was just blown away by how quick and easy they were to setup. I encourage you to read through his repositories and get a feel for the code yourself. 

#### node-sp-auth

[node-sp-auth](https://github.com/s-KaiNet/node-sp-auth) is a library that allows unattended http authentication (yes, there are security implications) via nodejs. Critically, `node-sp-auth` allows you to select one of a variety of different authentication methods, including oddities like Forms Based Authentication - which I use and therefore require. Sample usage is quite straightforward, as the following demo code shows. 

{% highlight javascript %} 
var spauth = require('node-sp-auth');
var request = require('request-promise');

spauth
	.getAuth('[siteUrl]', {
		username: '[username]',
		password: '[password]',
		fba: true 			//Only required in situations where Forms Based Authentication applies. 
							//In situations where Forms Based Authentication is not used, please refer to the node-sp-auth documentation on github.
	})
	.then(function(data){
		var headers = data.headers;
		headers['Accept'] = 'application/json;odata=verbose';

		request.get({
			url: '[siteUrl]',
			headers: headers,
			json: true,
			rejectUnauthorized: false
		}).then(function (response){
			console.log(response.d.Title);
		});
	});
{% endhighlight %}

This sample code will return the title of the requested page. I recommend going through the process and attempting to use `node-sp-auth` prior to tinkering with `spsave` and `gulp-spsave` as it is easier to understand & configure the code without additional functionality built around it. 

#### spsave

[spsave](https://github.com/s-KaiNet/spsave) lets you save files in SharePoint - very straightforward. Once you've figured out how to authenticate using `node-sp-auth` saving files to SharePoint is a breeze. Example Code: 

{% highlight javascript %}
var spsave = require('spsave').spsave;

var coreOptions = {
	siteUrl: '[siteUrl]',
	notification: true,
	checkin: true, // If you're saving a file exists, you're required to specify that it's a checkin, otherwise the save will fail.
	checkinType: 1 // Major Version Checkin - see the github repository for more.
};
var creds = {
	username: '[username]',
	password: '[password]',
	fba: true // Only required in situations where Forms Based Authentication applies. 
 		  // In situations where Forms Based Authentication is not used, please refer to the node-sp-auth documentation on github.
};

var fileOptions = {
	folder: 'SiteAssets', // Folder you would like the file to be saved to.
	fileName: 'file.txt', // Filename you're creating
	fileContent: 'Hello World!' // Contents of the file
}
spsave(coreOptions, creds, fileOptions)
.then(function() {
	console.log('saved');
})
.catch(function(err){
	console.log(err);
});
{% endhighlight %}

#### gulp-spsave

Now we get to the real fun! [gulp-spsave](https://github.com/s-KaiNet/gulp-spsave) is the culmination of the previous two libraries, and combines them to offer easy integration with gulp. Anything you normally do with gulp (sass, browserify, etc.) can be integrated with Sharepoint deployment. Setting a watch on your css directory and then automatically deploying changes becomes as easy as this: 

{% highlight javascript %}

var spsave = require('gulp-spsave');
var gulp = require('gulp');
var creds = require("./settings.js"); // I specified my login credentials in another file - more on how to do this at the gulp-spsave 
				      // documentation.
var siteUrl = "[siteUrl]";

gulp.task("deploy_css", function() {
	return gulp.src('css/*.css')
		.pipe(spsave({ // Note that you no longer need to provide filename/contents since it's piped in from gulp.
			siteUrl: siteUrl,				
			folder: "[documentLibrary]",
			checkin: true,
			checkinType: 0
		}, creds));
});

gulp.task("watch", function(){
	gulp.watch(["./css/*.css"], ["deploy_css"]);
});

gulp.task("default", ["watch"]);
{% endhighlight %}

And that's it! Now, when you run `gulp` in your terminal, it will keep an automatic watch on the `css` folder in your local directory, and any time there's a change to a css file in that directory (or a new one is added!) it will detect it and push it to Sharepoint. Pretty nifty! 

From here, there's very little you can't do. Combine it with as many different gulp tasks as you want and see how quickly you can leverage the power of automation in your workflow.