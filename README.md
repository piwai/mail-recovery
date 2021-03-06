Mail recovery tutorial
======================

This repository describes the procedure I used (with success!) to recover deleted emails, in the hope that it can help somebody else.

Note: Don't use this if you're using Microsoft Outlook or another mail agent which uses a global binary file per folder (.pst for Outlook). There are dedicated tools for that.

Here we will try to recover text files on a linux server with an ext4 partition. The files were emails stored in an IMAP folder accessed through a webmail, and have been deleted by mistake.
Details are here: https://serverfault.com/questions/942480/recovering-deleted-plain-text-emails-files-from-raw-disk-image

I will assume that you've already did what I did first:

 * Dump the whole server partition using "dd" and transfer it elsewhere
 * Try recovery tools on the dd image (ext4undelete, ext4magic)
 * Find that the tools don't see anything to restore.

I will also assume that you have basic Python knowledge to make the scripts and snippets work.


Step 0: Do a proper backup
--------------------------

If you're reading this, chances are that, just like me, you didn't have a working backup of your data. So, stop reading this, and do a backup of your other data which you care about. Now.


You're back? OK, let's continue.


Step 1: Configure foremost to locate emails
-------------------------------------------

Foremost (http://foremost.sourceforge.net/) is a data recovery tool.
Its purpose is to recover binary files like pictures, videos, compressed archives which were deleted from a hard drive... These files generally have a specific signature in their header, and they also include their own size in their metadata, so they are generally easy to restore when they have not been overwritten by other files. In my case, the mails were just plain text, with SMTP headers, body and base64 attachments, and sadly the SMTP standard doesn't define a content size header, like the "Content-Length" for HTTP. So foremost cannot know how much data it needs to extract when it detects a SMTP signature. 

So, we will use foremost only to locate where the emails could be on the partition, and we will delegate the extraction to a custom Python script, which will be able to extract just the correct amount of data.

To do this, you will need to define a specific signature based on a SMTP header encoded in hex, so that foremost can look for this in the partition and tell you at which offset it saw the signature. After a quick analysis of my own remaining emails I chose the header "Received:", which is a fairly common SMTP header that each MTA adds to the email when routing it to its destination.
I could have used something else like "Message-ID" which is unique, but this one is generally in the middle of the headers, after "From:" and "To:", so by using it I would have lost the mail sender and recipients.

To encode the header in Hex format used by foremost, use the snippet below:

```
$ python3 -q
>>> header = "Received:"
>>> print(''.join(str(hex(ord(c))) for c in header).replace('0x','\\x'))
\x52\x65\x63\x65\x69\x76\x65\x64\x3a
```

Then put this result into a foremost.conf file, like this:
```
$ echo "txt y 1000 \x52\x65\x63\x65\x69\x76\x65\x64\x3a" > foremost.conf
```

You may also use the foremost.conf from this repository.

Step 2: Run foremost to get the potential location of files
-----------------------------------------------------------

Now that we have set our custom config to locate emails, we can run foremost to tell us at which offset it saw the signature.
On Debian-based systems, foremost can be installed with:

```
$ sudo apt-get install foremost
```
If it's not available for your distribution, you may need to compile it from source.

Once installed, use the following command to run it:

```
$ foremost -w -c ./foremost.conf -i disk.img
```
 * -w is "dry-run" mode, it means don't try to extract the data, just locate the header and write it in the audit file.
 * -c tells to use the config in the current folder
 * -i means use "disk.img" as the raw image extracted by "dd" to look for deleted mails.

Depending on the image size, this might take several hours to complete.
When it's done, you should get a file named "audit.txt" in a "output" folder.
There is a "audit.txt.sample" file in the repo so you can see what it looks like.
In my case, this "audit.txt" file was nearly 45MB with more than 800k matches.

Step 3: Extract file chunks based on audit.txt file
---------------------------------------------------

Now starts the tricky part. Like we've seen, foremost doesn't know how much data it should extract when it finds a match.
So, what we will do is parse the audit.txt from step 2, and extract the chunks using the extract-chunks.py script.
The script will re-create one file per chunk of data which looks like an email, using some simple heuristics to merge "Received:" entries which belongs to the same email (because an email generally includes this header multiple times, one per MTA)

```
$ python3 extract-chunks.py audit.txt raw_image.bin
```

The chunks will be written in the "chunks" folder by default. After this stage I got around ~100k chunks using 15GB space, representing for the most part every single email which has ever existed on the server, deleted or not, spam or not, plus also some garbage. It's far less than the original 900GB raw image, but there's still work!

Step 4: Filter chunks to keep only the deleted emails
----------------------------------------------------

In my case the deleted mails were in my father's inbox, so the filtering was quite simple: Just keep the mails which have his address as the recipient.
The "filter-recipient.py" takes care of that, using the python "email" module from the standard library.

```
$ python3 filter-recipient.py chunks dad@domain.com
```
This will look at each chunk, and move it to the "emails" folder if the recipient is "dad@mydomain.com".

After that step, I had 7k mails using 1,5GB space.

Step 5: Remove duplicates
-------------------------

At this stage, I tried to import back the recovered emails into the IMAP folder, and it worked! I was able to open them using a mail client, the attachements were opening correctly...
But I noticed several of them were duplicated, so I added one last script to remove them.

```
$ python3 filter-duplicates.py emails
```

Using the "Message-Id" seen earlier, I put all the extra copies in a "duplicates" folder, to keep only a single copy of each message. I'm not familiar enough with ext4 and postfix internals to explain these duplicates, I assume the filesystem sometimes move things to optimize space, or maybe postfix needs to re-write them during anti-spam processing.
Anyway, I finally got around 5k non-duplicated emails, which I could sucessfully restore in the IMAP folder, like nothing ever happened. It was a relief for me and my father.

Conclusion
----------

Using this procedure, I was able to successfully recover almost all the emails that were lost.
I could have avoided all of this hassle with a proper backup (which I now have).
If you ever find yourself in a similar situation, I just hope that these scripts will be useful to you. If yes, i'll be glad to hear about it!

