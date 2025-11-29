---
title: "Turning old unused iPads into photo frames"
date: 2025-05-18
---

I've always liked having a dynamic photo frame at home which shows off a selection of recent photos.

We take a lot of photos these days and it feels like showing them off in a nice photo frame is a nice way to enjoy them.

I've had various flavours of this over the years. Sadly the pace at which technology changes is often the reason why a perfectly good working solution stops working! And that's the reason for this post.

I suspect that most people who want to do this buy a custom made photo frame. And either feed it with a USB stick of photos and are happy to manually update them every now and again. Or are happy to pay a subscription for a service to integrate with their online photos.

My requirements are:

- Recent photos are automatically displayed on the photo frame shortly after taking

- Photos could be a mixture of "old special photos" and more recently taken ones

- No subscription fees

- Ideally using old devices that wouldn't otherwise be usable

- Attractive and HD images with no other visual clutter (or adverts!)

First some context, I use Dropbox and Google Photos and Amazon Photos to store my photos and those of my family. These days we generally do this through direct integrations from our phones with these three providers.

Why do I have 3 providers to do basically the same thing? Well... I like to have at least 2 for redundancy - I have had photos accidentally lost before in Dropbox and so I like to reduce this risk by having two providers.

I already pay for Dropbox for file backup and I have 2 Tb of space so there's no reason not to store photos on Dropbox and keep an archive there. The Dropbox photo client is abominable (it never ceases to amaze me how bad it is!). I also have Amazon Prime which gives me free storage of photos and so I integrate with it. And I find the interface for finding photos a bit better than Dropbox. Not much. And then a recent experiment saw me sync all photos into Google Photos where theoretically the interface to find and manage photos is a lot better.

Photo frames are perfect for older devices which might not have a use otherwise. And feels like a responsible and sustainable thing to do with them - rather than just throwing them away.

For a long time, I've used an old iPad (one which couldn't be upgraded beyond 2020) and a really good app called [LiveFrame](https://liveframeapp.com/) which is purposefully designed to keep working with old devices and can be installed right back to iPads with iOS versions 9.3.5/9.3.6. I used the integration with Google Photos which has been generally reliable and the way I set it up means photos from the last month are shown in random rotation on the iPad and fairly quickly after taking a photo and uploading it, it will appear on the frame.

It worked brilliantly for years and years - this combo met my needs perfectly.

But unfortunately only up until 31st March 2025 when [Google annoyingly changed it's APIs](https://developers.google.com/photos/support/updates#library_api_impact_on_common_use_cases) such that 3rd party apps cannot be granted access to ALL your photos - which means that LiveFrame can no longer discover your recently taken pictures and the whole thing stops working.

I presume Google is trying to constrain the things third party apps can do and probably has "privacy" as it's main goal. But in this case they really are taking away something substantial that feels like it's enforcing device obsolescence.

I was really annoyed (at Google) for doing this. It's such a simple thing but something I genuinely love to have - and shouldn't need us to be spending a lot more money every few years on a brand new device to achieve it.

As it happens I think LiveFrame have given up altogether because they haven't even bothered to make the changes to work with Google Photos. I've tried to contact them but got no response.

When this happened I was disappointed. It had worked perfectly for years and years. But I thought the LiveFrame integration with Dropbox might work instead. Unfortunately I've tried various things with that but it just won't display photos. Not sure if I have too many photos or what, but it doesn't work.

So last weekend I thought I was going to have to abandon my old iPad altogether - it's not much use for anything else because it can't be upgraded and most apps don't work on it or can't be installed.

Luckily whilst digging through some forums I came across [Pixette](https://pixette.app/) - an app written by someone with similar ambitions. But this time - and for good or bad - Pixette doesn't have any integrations with 3rd party tools. Instead it is a way to display a \*locally\* served image library (off a NAS or similar) onto an iPad. This turns out to be genius and actually means that my new solution can be much more resilient to future changes in tech.

So Pixette runs on my old iPad and expects the details of a local WebDav server which contains the images to display. This takes the responsibility for getting the images in sync away from the iPad and means it can just focus on displaying the images. Much better.

In order to create a working solution I then needed to create the missing local server and somehow connect it to a source of photos. Now Google Photos is out because of the same restrictions via the API - and even Google has stopped providing an automatic "syncing" tool for a desktop or laptop computer.

I happened to have a mini PC running as a local server used for other purposes (e.g. Home Assistant) that I could use for this. But the clincher was coming across [rclone](https://rclone.org/) which is a very capable synchronisation tool which happens to have a Dropbox integration AND can serve files over WebDav.

So below I'm going to take you through the steps I needed to do to get this working. And I can say it is now just as good as before - but even more resilient.

What you'll need:

- A local server device of some kind. I have an old Dell OptiPlex 3040 Micro Desktop (very cheap off eBay) which I had setup Proxmox running on it so I could spin up a Virtual Machine easily to run these jobs. Any device running linux would do nicely. It does need to be running 24/7 so a low power device is useful.  
    

- An old iPad that can run Pixette  
    

- Photos in Dropbox (or potentially other sources that integrate via rclone also work)  
    

The way it works is:

- You have an `rclone mount` job running against Dropbox with a "last month" filter so that it automatically synchronises only files from the last month (for your Camera Uploads folder) onto your linux server:  
```
    rclone mount dropbox: /mnt/pve/shared/dropbox --allow-non-empty --max-age 6w
``` 

- Then a similar `rclone serve` job which is running against that same synchronised folder and which then serves those images locally over a WebDav connection:  
    
```
    rclone serve webdav '/mnt/pve/shared/dropbox/Camera Uploads' --user USER --pass PASSWORD --addr :2121
```

- Then the iPad and Pixette configured to point at the same URL from the latter WebDav protocol so that it can see the images.  
      
    In my case the URL `http://\[LOCAL IP OF SERVER\]:2121/` now contained the images!  
    

For running this a little more robustly you can use systemctl to turn these into jobs.

I also found that I needed to turn on the caching mode on rclone so that when the Dropbox sync was updating Pixette didn't see images disappearing or throw errors.

So the final commands were:  

```sh
vi /etc/systemd/system/rclone-mount-dropbox.service
```

Enter:

```
\[Unit\]  
Description=Rclone mount  
After=network.target
 
\[Service\]  
User=root  
Group=root  
ExecStart=/usr/bin/rclone mount dropbox: /mnt/pve/shared/dropbox --vfs-cache-mode full --max-age 12w --allow-non-empty  
Restart=on-failure

\[Install\]  
WantedBy=multi-user.target
```

And then for the serving:  

```sh  
vi /etc/systemd/system/rclone-serve-photos.service
```
  
```
\[Unit\]  
Description=Rclone serve  
After=network.target

\[Service\]  
User=root  
Group=root  
ExecStart=/usr/bin/rclone serve webdav '/mnt/pve/shared/dropbox/Camera Uploads' --user USER --pass PASSWORD --addr :2121 --vfs-cache-mode full  
Restart=on-failure

\[Install\]  
WantedBy=multi-user.target
```

Now you can load those jobs and start and stop them:

```
systemctl daemon-reload
systemctl start rclone-mount-dropbox.service
systemctl status rclone-mount-dropbox.service
systemctl stop rclone-mount-dropbox.service
```
  
```
systemctl start rclone-serve-photos.service
systemctl status rclone-serve-photos.service
systemctl stop rclone-serve-photos`.service
```

I have to say it's really satisfying to have done this. And it's been running smoothly for months now.

There is one outstanding problem with the setup:

After a few days/weeks it seems to lose the dynamic updates.
I think this is a problem with Pixette not the server - but basically it throws lots of not found errors when re-indexing. The solution so far has been to restart the server and then trigger a manual reindex. Am hoping to find a fix for this.

Thanks Google for forcing me to have to learn a whole bunch of new stuff!
