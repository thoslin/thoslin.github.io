---
layout: post
title: "A hack to load openx asynchronously"
date: 2013-07-20 22:39
comments: true
---
I failed to load openx asynchronously follwing their documentation. It's very disappointing especially when you read the fucking manual and it just doesn't work. So I found a way to work around it. A bit dirty but works.

The normal openx setup is to reference the openx script in the head section. The script will issue a request to openx server to pull the ads content for the ad scripts in the body section to document.write. As you know this script will block the page load. But if you move the script to the bottom of the body, the document.write will fail.

``` html
<html>
    <head>
        <script type="text/javascript">
            var OA_zones = {
                'top_banner': 120
            }
        </script>
        <!-- the script will issue a request to fetch the ads -->
        <script type='text/javascript' src='http://ads.example.com/www/delivery/spcjs.php?id=7'></script>
    </head>
    <body>
        <script type='text/javascript'>
            // openx document.write the ad content
            OA_show('top_banner', true);
        </script>
    </body>
</html>
```

What I did is basically override the OA_show function of openx and queue the function calls and release them when the time is right.
``` html
<html>
    <head>
        <script type="text/javascript">
            var adQueue = [];
            var OA_show = function(position, output) {
                // queue the function call and create a place holder
                adQueue.push([position, output]);
                document.write("<div id='_"+position+"'></div>");
            }
        </script>
    </head>
    <body>
        <div id="top_banner">
            <script type='text/javascript'>
                OA_show('top_banner', true);
            </script>
        </div>
        <script type="text/javascript">
            var OA_zones = {
                'top_banner': 120
            }
        </script>
        <script type='text/javascript' src='http://ads.example.com/www/delivery/spcjs.php?id=7'></script>
        <script type='text/javascript'>
            // excute function calls and display ads
            while(adQueue.length > 0) {
                var args = adQueue.shift();
                var ad = OA_show(args[0], false);
                if(ad){
                    $("#_"+args[0]).html(ad);
                }
            }
        </script>
    </body>
</html>
```
PS: Openx sucks!
