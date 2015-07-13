---
layout: post
title: iOS Metaprogramming with Python, Pt 2
date: 2015-07-06
tags:
- metaprogramming
- python
- xcode
- code
- json
---

so offended
------------

I recently became so mightily offended by the process of adding an App icon to my Xcode project, I couldn't bear to move forward without scripting it. One wonders if this is the impetus for all automation. 

To understand why I wanted to automate this, we should first take a look at the current process and see its pain points.
Consider FooApp, an an iPhone-only iOS application. 

![such icon](/assets/2015-07-06/oldicon.png)

brazen disregard
----------

Xcode wants us to provide 3 different sizes of icons, at 2 different scales, for a total of 6 assets that a designer (or in this case, me) needs to cut. An app that also supported iPad would require even more assets. 

In a manual process, I need to:

* open up my large (1024x1024) source image
* resize it appropriately for one of the size/scale combinations
* save it with a descriptive name for later identification

After repeating the above steps for each of the 6 possibilities, I need to:

* locate the new images in Finder
* drag them into the appropirate slot in Xcode

This process is tedious enough that it discourages me from iterating through different icon options. It occurred to me that I could probably come up with a way to automate all of this, so long as I have a large source image and the path to my Xcode project.

apply python
--------

First things first, how do we use python to resize an image? A bit of googling led me to [Pillow](https://python-pillow.github.io/), a wrapper/fork for Python Image Library (PIL). It looks like the Image.thumbnail() function from PIL will give us the result we want. After installing it, I wrote a small script to verify it works:

{% highlight python %}
from PIL import Image

source_img = Image.open('path/to/image')
try:
	size = (120,120)
	source_img.thumbnail(size)
	source_img.save('new_img', "PNG")
	successStr = 'created image for size ' + str(size)
	print successStr
except IOError:
	errStr = 'cannot create image for size ' + str(size)
	print errStr
	return

{% endhighlight %}

Sure enough, this produced a png named 'new_img'. I opened it up and verified the correct dimensions (120x120px in this case) and that the image quality had not degraded too drastically. Everything looks good, so the "image" portion of our task is essentially done. 

what is an XCAsset?
--------------

When Xcode deals with the images in your project, it prefers to think of them as XCAssets. From XCode's perspective, our "App Icon" is a single object, with 6 separate representations. The appropriate representation can be determined at runtime, based on the OS and Device you need to draw the icon for.

XCode stores the metadata for these asset files in a file called Contents.json. The JSON for the doge icon in the screenshot above looks like:

{% highlight json %}

{
    "images": [
        {
            "filename": "doge_60_3x.png",
            "idiom": "iphone",
            "scale": "3x",
            "size": "60x60"
        },
        {
            "filename": "doge_60_2x.png",
            "idiom": "iphone",
            "scale": "2x",
            "size": "60x60"
        },
        {
            "filename": "doge_40_3x.png",
            "idiom": "iphone",
            "scale": "3x",
            "size": "40x40"
        },
        {
            "filename": "doge_40_2x.png",
            "idiom": "iphone",
            "scale": "2x",
            "size": "40x40"
        },
        {
            "filename": "doge_29_3x.png",
            "idiom": "iphone",
            "scale": "3x",
            "size": "29x29"
        },
        {
            "filename": "doge_29_2x.png",
            "idiom": "iphone",
            "scale": "2x",
            "size": "29x29"
        }
    ],
    "info": {
        "author": "xcode",
        "version": 1
    }
}

{% endhighlight %}

If we're going to automate the insertion of our images into Xcode and eliminate the "drag and drop" step of the process, we're going to need to be able to generate a compatible JSON file. A nice object-oriented way of doing this is to create a data model for the "slice" objects in python. Then we just need to give those slices a way to output a JSON representation of themselves. The data model may look like this:

{% highlight python %}

class XCAssetImage:
	def __init__(self,size,scale,idiom,filename):
		self.size = size
		self.scale = scale
		self.idiom = idiom
		self.filename = filename

{% endhighlight %}

The initializer expects 5 arguments, although we are only going to pass it 4. 
The first argument, 'self' is passed implicitly. The remaining ones are size, scale, idiom and filename. These correspond to the properties in the above JSON. 

Before we go off and code the to_json() method, it might be helpful to figure out how we want to initialize our Slice objects. A simple way to test our model is to initialize 6 Slices, with the same size/scale/idiom attributes that Xcode expects. We'll know the filename at runtime because we'll be creating the images, so we do not need to mimic that property.

Initializing those objects programatically may look like:

{% highlight python %}

def generate_assets(filename):
	iphone = 'iphone'
	scale2x = 2
	scale3x = 3
	assets = [
			XCAssetImage(60,scale3x,iphone,filename),
			XCAssetImage(60,scale2x,iphone,filename),

			XCAssetImage(40,scale3x,iphone,filename),
			XCAssetImage(40,scale2x,iphone,filename),
			
			XCAssetImage(29,scale3x,iphone,filename),
			XCAssetImage(29,scale2x,iphone,filename)
			]
	return assets

{% endhighlight %}

<!-- {% highlight python %}

import os, sys, fnmatch, json, shutil
from PIL import Image

class XCAsset:
	def __init__(self,size,scale,idiom,filename):
		self.size = size
		self.scale = scale
		self.idiom = idiom
		self.filename = filename+'_'+str(self.size)+'_'+str(self.scale)+'x.png'

	def to_json(self):
		jsonDict = {}
		jsonDict['size'] = str(self.size)+'x'+str(self.size)
		jsonDict['idiom'] = self.idiom
		jsonDict['filename'] = self.filename
		jsonDict['scale'] = str(self.scale)+'x'
		# print jsonDict
		return jsonDict

def emptyDir(directory):
	for root, dirs, files in os.walk(directory):
		for f in files:
			os.unlink(os.path.join(root, f))
		for d in dirs:
			shutil.rmtree(os.path.join(root, d))

def findFiles(projectPath):
	
	pattern = 'AppIcon.appiconset'

	for root, dirnames, filenames in os.walk(projectPath):

		for directory in dirnames:
			if pattern in directory:
				print 'found file'
				fullPath = os.path.join(root, directory)
				print fullPath
				return fullPath

def makeImage(source_img,asset):
	try:
		pixels = asset.size*asset.scale
		size = (pixels,pixels)
		source_img.thumbnail(size)
		source_img.save(asset.filename, "PNG")
		successStr = 'created image for size ' + str(size)
		print successStr
	except IOError:
		errStr = 'cannot create image for size ' + str(size)
		print errStr
		return

def generate_assets(filename):
	iphone = 'iphone'
	scale2x = 2
	scale3x = 3
	assets = [
			XCAsset(60,scale3x,iphone,filename),
			XCAsset(60,scale2x,iphone,filename),

			XCAsset(40,scale3x,iphone,filename),
			XCAsset(40,scale2x,iphone,filename),
			
			XCAsset(29,scale3x,iphone,filename),
			XCAsset(29,scale2x,iphone,filename)
			]
	return assets

def writeContents(assets):

	jsonDict = {}

	jsonDict['info'] = {
					    'version' : 1,
					    'author' : 'xcode'
						}

	images = []
	for a in assets:
		images.append(a.to_json())

	jsonDict['images'] = images

	#pretty print
	print json.dumps(jsonDict,sort_keys=True,indent=4,separators=(',', ': '))

	with open('Contents.json', 'w') as outfile:
		json.dump(jsonDict, outfile)

def main():

	#intended invocation:
	#python make_images.py <imagename (relative to script location)> <project path>
	#python make_images.py FooAppIcon2.png /Users/niko/Documents/FooApp
	argsList = sys.argv

	if len(argsList) < 3:
		exit('not enough args')

	path = argsList.pop()
	imgName = argsList.pop()
	fileName = os.path.splitext(imgName)[0]

	source_img = Image.open(imgName)

	newDir = findFiles(path)
	emptyDir(newDir)
	assets = generate_assets(fileName)
	os.chdir(newDir)

	for a in assets:
		img = source_img.copy()
		makeImage(img,a)

	writeContents(assets)

if __name__ == '__main__':
	main()

{% endhighlight %} -->


___full gist___

{% gist a56ffdd77145905fe8fb %}
