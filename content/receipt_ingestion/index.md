+++
title = "Trials and Tribulations of Automating Receipt Ingestion"
date = 2022-04-02
[extra]
header_image = "/receipt-ingestion/poster.jpg"
[taxonomies]
tags= ["computer vision", "OpenCV", "OCR", "hacking", "programming", "python"]
+++

You've just gone out for a meal with a friend.
You take a picture of your receipt with your phone.
Within moments, all pertinent information is extracted
and sent to [Splitwise](https://secure.splitwise.com), the expense-sharing service you use.
This is the dream I had recently when manually ingesting a large number of receipts.
Being a programmer, naturally I spent more time realizing this dream
than it would have taken me to manually digitize every receipt I will ever receive.

<!-- more -->

## The plan

The original plan was quite simple:
take a picture of the receipt,
detect the corners,
straighten the image and crop in on the receipt,
perform optical character recognition (OCR) to extract the receipt text,
and extract all desired information.

That last step involves searching through the output text
to find the total amount, the name of the issuer of the receipt, the date, and the person who paid.
This is mostly done through the use of regular expressions.
For example, the total amount can be found by looking for a line of text that includes the string `total` followed by some punctuation and a number.

I copy-pasted some code from a handful of blog posts
and had a demo running in an evening.

The image below shows the major steps of the procedure on a simple example.
I say that the example is simple because it is a short receipt with clean corners that sits flat on the table.
I took the picture on an angle to show the perspective transformation.
Normally, I would try to take the picture straight down and rely on the transformation only for small corrections.

| ![](/receipt-ingestion/plan.svg) |
| -------------------------------- |
| From left to right, top to bottom: the original image, a Canny edge detection, finding the receipt contour, four-point transformation based on the contour's corners, conversion to a binary black and white image, and the result of running [Tesseract OCR](https://tesseract-ocr.github.io/tessdoc/Home.html). The OCR is not perfect, but all desired fields (restaurant name, total, date, credit card last four digits) are present. |

## "Hacking" Splitwise

Now that I have a cropped image of a receipt and a bunch of information about its contents,
how do I get this into Splitwise so everyone knows how much they owe me?
This part was surprisingly easy.
I went to [https://secure.splitwise.com](https://secure.splitwise.com),
opened my devtools to the network monitor (filtering for XHR requests),
and submitted their form to add a receipt.
A new entry appeared instantly, revealing their API.
Here is what I saw in the request:

```json
{
    "cost": "123",
    "currency_code": "CAD",
    "group_id": "<redacted>",
    "users__0__user_id": "<redacted>",
    "users__0__owed_share": "62",
    "users__1__owed_share": "61",
    "users__1__user_id": "<redacted>",
    "users__0__paid_share": "123",
    "users__1__paid_share": "0",
    "category_id": "18",
    "date": "2022-04-02 18:38:51",
    "description": "My Very Favourite Store",
    "creation_method": "equal"
}
```

I right-clicked the request and clicked "Copy as cURL".
This copies not just the URL, but everything about the request in the format of [`curl`](https://curl.se/),
including headers, user agent, and referrer.
I pasted this `curl` command into my Python file and manipulated it into the [`requests`](https://docs.python-requests.org/en/latest/) format.
I ran the code, and to my amazement, it worked!
I could add any receipt I wanted to Splitwise programmatically.
I could even add a picture of the receipt if I specify the `files=` keyword argument.
I expected this step to be much more problematic.

## Receipts are awful

This is the part of the story where I wonder if I have bitten off more than I can chew.
My implementation only worked for the most trivial of cases.

Receipts are awful!
They crumple easily.
They refuse to lie flat for a clean image.
Sometimes, the corner gets torn off when ripping the receipt out of the machine, throwing off the rectangle detection and four-point transformation.
Other times, you get an extra little piece in the corner where somebody else's receipt tore.
Is this even viable?

| ![](/receipt-ingestion/ears.png) |
| -------------------------------- |
| An example of a receipt with an extra ear on the corner throwing off the corner detecting and four-point transformation. |

On top of that, OCR is still far from perfect, especially for cellphone images of crumpled receipts.

If I remember correctly, at this point,
my method had only successfully found all the desired data for one image out of the five I was using to test.
I needed a new plan!

## Scanners to the rescue??

If I place my receipt into my scanner,
it should press it flat, eliminating the receipt's wrinkles
and aforementioned refusal to sit flat on the table and pose for a photo.
In practise, it did just that.
The quality of the scans was exceptional.
The only issues were the overhead of manually putting receipts into the scanner
and the limitations of the length.

The "flatbed" (the part that is used by opening the lid and placing a piece of paper onto a sheet of glass)
is fiddly to use for a large number of receipts
and cannot scan very long receipts like those listing an entire week's groceries.

Therefore, I attempted to use the "feeder" part of the scanner, which is intended to scan a stack of pages
by sucking them into the printer, moving them across the scanner, and spitting them out.
Since it is the *paper* that is moving and not the scanner,
in theory I should be able to scan a receipt of any length.

The receipts cannot be fed through the scanner alone,
so I made a backing by cutting the sides off of a large envelope (the kind that can fit an entire sheet of paper without folding it)
and unfolded it.
I now had a piece of paper that was more than twice as long as a standard sheet of paper (if you count the little flap that gets folded back to seal the envelope).
I could stick the receipts to this and use the envelope to help guide receipts through the scanner.
Since I used a brown envelope, there was even good contrast between the backing and the receipt to do edge detection.

Of course, it couldn't be so simple.
My scanner seems to be limited to documents of 14 inches (355.6 mm for us civilised people) in the feeder.
I don't see any reason for this.
The feeder should support documents of any length.
It just has to feed them through with rollers and scan a row of pixels as they pass by.
What does it care about the length?
The scanner has no problem *feeding* through a long document, it just stops *scanning* at some point.

I was unable to find a solution to this issue.
I even put a bounty on [my Stack Exchange question](https://superuser.com/questions/1709265/scan-documents-longer-than-14-inches-with-brother-scanner-using-sane-in-linux).
I will have to find another way.

## Maybe a website??

Getting pictures off of my phone and into my computer to run my code against them
was quickly getting tiring.
I can only imagine how annoying it would be once this project is finished (as if any project is ever finished)
and I'm just trying to use it.

Not wanting to touch Android Studio with a 10-foot pole, I had the idea to set up a website to take a picture and send it to the server for processing.
I did some research and found the [`MediaDevices.getUserMedia` API](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia)
that can be used to stream video data from your device's camera.
I hacked something together,
but it was clear that this API was indented to be used for a webcam in a video call and not to take pictures.
To use it in a camera app, you need to first stream the camera feed to a `<video>` element.
When the shutter button is pressed,
draw the current frame of the `<video>` to a `<canvas>`.
To send the image to the server,
save convert the `<canvas>` to base-64 with `canvas.toDataURL` and sent that data in a network request.

Since I need to crop in on a small part of the image (the receipt) and do OCR,
I need a high-resolution image.
However, if I use a full-resolution video stream,
the preview is unusably laggy (seconds per frame not frames per second).
Furthermore, converting that large image to base-64 takes around 10 seconds on my phone,
and sending this data is very inefficient.
Finally, the flash is not accessible through this API,
and I have found that the flash is very helpful to get crisp images for OCR.

The [`ImageCapture` API](https://developer.mozilla.org/en-US/docs/Web/API/ImageCapture) promises to fix many of these issues,
but the browser support is still very limited.

I have given up on this app,
but the source is still [available in a branch](https://github.com/MarcelRobitaille/Receipt-Ingestion/tree/webapp).

## The silver bullet: QR codes

After giving up on the website, eureka!
Why am I limiting myself to trying to detect a receipt in the image?
I can add whatever I want to the image to make my life easier.

What to add?...
If I put a QR code in the image flat on the table,
I can figure out the angle of the camera with respect to the table
and transform the image to a top-down view.
The QR library will even tell me exactly how to transform the image!

Hey, I can use the QR code's data to my advantage too!
Instead of taking a picture in a website,
I only have to look for this QR code in every image I take.
All my pictures are already synchronized to my computer anyway.

I was really getting somewhere now.
I updated my Python program to watch for new files in my phone's "Camera Roll" folder.
If a new picture appears without the QR code,
the program skips that image.
If the QR code is present, I use this tag to help with the perspective transformation (shown in the image below)
and continue with the image processing (same steps as before from this point).
I am using the amazing library [`pyzbar`](https://github.com/NaturalHistoryMuseum/pyzbar/) because I have had bad luck
analyzing QR codes with OpenCV.

| ![Representation of the QR straightening on an examplary receipt](/receipt-ingestion/qr.png) |
| -------------------------------------------------------------------------------------------- |
| The results of the QR perspective transformation on an exemplary image |

The image above shows an extreme case for demonstration purposes.
By using the QR code, I can get a top-down perspective of the table by leveraging existing robust QR libraries.
At this point, I need only rotate the receipt,
which is much more reliable than finding its corners for a four-point transformation.
I cut the page from the QR code into this silly shape for two reasons: (1) it reminds me quickly which way is up (2) it does not trigger the rectangle detection when filtering contours.

I find that this is quite a nice UX.
I don't have to tape anything to an envelope and feed it through my scanner.
I don't have to hunt for an app or a website to take a picture and have it appear in my server.
I just take a picture with my normal camera app, and it will be analyzed.
If the image does not contain a magic QR code,
it won't be processed,
so the street sign in the background of a portrait won't tell my friends that they owe me some nonsensical amount of money.

## Conclusions

I am still using this software almost every day.
Sometimes, an image gets processed correctly.
Other times, my software gets better as a result of my expanding it to handle a new edge case.
I am very proud of this project.
I think it's extremely cool that I can take a picture and have a receipt show up in a website I don't control
and have push notifications get sent to my friends telling them they owe me money as a result.
At the same time, I am very aware that this project has taken a lot more of my precious time than I anticipated,
and that it would have been much faster for me to digitize those receipts by hand.
But where's the fun in that?!
Plus, if this blog post or the [open source code on GitHub](https://github.com/MarcelRobitaille/Receipt-Ingestion) is used by or inspires anybody,
then it was all worth it.
