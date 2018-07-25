---
layout: post
title: "Async Download with Celery"
comments: true
---
Recently my coworkers in Beijing changed our front end server from squid to varnish. I don't get to see the new configurations but one function on our site has broken since the change. Nginx log reports 499 status code, which means the client has closed the connection before the response is sent. So I guess probably the timeout settings was changed so this time consuming function got no chance to finish and return the response.

Anyway this function is a pain in the ass for a long. What it does is basically exporting a large query set to a csv file. It usually takes 3 to 5 minutes to finish. The legacy solution was set a very large timeout in uwsgi, nginx and front end servers. Obviously it's a very ugly approach. You got the client waiting and other requests blocked.

Here I'm gonna refactor this function with the all mighty Celery.

This is the pseudo code of the current django view:

``` python
def export(request, **kwargs):
    qs = get_queryset(**kwargs)
 
    response = HttpResponse(mimetype='text/csv')
    response['Content-Disposition'] = 'attachment; filename=foo.csv'
    writer = csv.writer(response, dialect=csv.excel)
 
    for i in qs:
        writer.writerow([i.foo, i.bar, i.baz, ...])
 
    return response
```
The forloop is the bottleneck here. If we can do it outside the request-response life circle, ie asynchronously, the problem will be solved. And that's where Celery comes in:
``` python
from tasks import generate_file
 
 
def export(request, **kwargs):
    task = generate_file.delay(**kwargs)
    return render_to_response("poll_for_download.html",
                              {"task_id": task.task_id }
```
So we simply dump the whole process into this generate_file task and pass it the arguments
it needs. The workers will start processing in the background. Meanwhile the response is sent to client side. The request is fulfilled, the resource is freed.
``` python
@task
def generate_file(**kwargs):
    filename = "%s.csv" % generate_file.request.id
    qs = get_queryset(**kwargs)
 
    with open("%s%s" % ("/path/to/export/", filename), "w+") as f:
        writer = csv.writer(f, dialect=csv.excel)
 
        for i in qs:
            writer.writerow([i.foo, i.bar, i.baz, ...])
 
    return filename
```
The client will poll for result every 5 seconds(we can also leverage socket.io to build a long connection here which is more elegent) with the task_id we sent on last request.
``` html
<!DOCTYPE html>
<html>
<head>
    <title></title>
</head>
<body>
<div id="content">Please wait. <span id="loading"></span></div>
<script type="text/javascript" src="/path/to/jquery.js"></script>
<script>
    $(function(){
        $.ajaxSetup({ cache: false, timeout: 360000 });
        var url = "/poll_for_download/";
        var i = 0;
        (function worker() {
            $.getJSON(url+"?task_id={{ task_id }}", function(data){
                if(data.filename) {
                    var file_url = url+"?filename="+data.filename;
                    $("#content").html("If your download doesn't start automatically, please click <a href='"+file_url+"'>here</a>.");
                    window.location.href = file_url;
                } else {
                    setTimeout(worker, 5000);
                }
            });
        })();
        setInterval(function() {
            i = ++i % 4;
            $("#loading").html("loading"+Array(i+1).join("."));
        }, 1000);
    });
</script>
</body>
</html>
```
The is the view to tell the client if the task is done and file is ready for download. If it's ajax request we'll check if the task status via task.ready() call. If it's ready we get the filename returned by the task and inform the user to download.
``` python
def poll_for_download(request):
    task_id = request.GET.get("task_id")
    filename = request.GET.get("filename")
    
    if request.is_ajax():
        result = generate_file.AsyncResult(task_id)
        if result.ready():
            return HttpResponse(json.dumps({"filename": result.get()}))
        return HttpResponse(json.dumps({"filename": None}))
    
    try:
        f = open("/path/to/export/"+filename)
    except:
        return HttpResponseForbidden()
    else:
        response = HttpResponse(file, mimetype='text/csv')
        response['Content-Disposition'] = 'attachment; filename=%s' % filename
    return response
```
And make sure you set CELERY_IGNORE_RESULT to False and get result backend setting right. Otherwise tasks may be stuck in pending state.

That's it. Happy hacking!
