04/09/18 - Tuesday

Rebuilding
Ssh into same container as last night. Runserver is magically running, all three processes show the same start time, nothing is returned on curl
K8s only records those processes into stdout that starts from the entrypoint. Let’s try removing the command in the k8s? Also maybe we could change the explicit 8000 to 0.0.0.0:8000
Did all of the above, plus wrote rule to explicitly open firewall on 8000 (gcloud compute firewall-rules create my-rule --allow=tcp:8000) didn’t make a difference.
Seen a high increase in the restarts of the pods, could it be because drf is primarily on py2 on my local? Is something messing up there?
Set up yet another completely brand new project and make it run? Okay mom.
Have to record at this point that runserver again kept hanging, didn’t giev us more error like last time, so mysqlclient probably worked. Python shell works.
On trying to get all models, I (finally!!!) get a different error. _mysql_exceptions.OperationalError: (2005, "Unknown MySQL server host 'mysql.default' (-2)")
This means my ‘host’ variable in the config OR the settings.py file was wrong. Rpobably the latter. Need to figure out what this means.
Added ‘     tty: true’ into my container spec. Fingers crossed.
LOGS WORK YES YES YES YES
Generally host is set to local/db, but in a container does each container have it’s own local network? Iirc no.
Set host to localhost, “_mysql_exceptions.OperationalError: (2002, 'Can\'t connect to local MySQL server through socket \'/var/run/mysqld/mysqld.sock\' (2 "No such file or directory")')”
Set host to 127.0.0.1 (uses internal TCP/IP as opposed to unix sockets). Works.
Access denied for user sb. Does the mysql service have an sb or a root?




'HOST': os.getenv('POSTGRES_SERVICE_HOST', '127.0.0.1'),
   'HOST': '127.0.0.1', # Or an IP Address that your DB is hosted on






03/09/18 - Monday

Rebuilding
Decided to run a sample django app. 
Have drf tutorial project. Let’s reuse that + move it to sql
Lots of random things going on in current local mysql setup. Don’t have time to fix this but TODO.
Re-fixed the nginx config. Turns out this upstream is supposed to be within the http context (tested via nginx -t -c and it gives the same error)
Question: But then why did that DO article I refer to say it was okay?
Back to bad gateway errors. Yay.
Wait, my config is porbably running the server from 8002 but nginx is redirecting to 8000 (why did I redirect to 8002 anyway?) No 8002 was another experiment, I have the port right
Nginx logs say connection refused -> nothing running on the localhost 8000 server in that case. Points more and more to backend issue. Lines up with server not running, etc.
Yeah btw there’s always two things running runserver
First has PID 1, doesn’t die even if kill -9. Spawned from docker image thing.
Second has PID 5, upon killing will immediately exit shell… and upon re-entering process will still be there.
Pip also stalls on using, will add mysqlclient to requirements and see what happens. Argh.
Updated requirements to mysqlclient, took literally two hours to pushabout 600mb rip mobile data. Deployed okay. Ps aux shows literally nothing running. Why must god hurt me this way.
Logs empty as usual. 




31/08/18 - Friday

Rebuilding
Wrote the dockerfile for the django container, works great, can’t connect to mysql
TODO: change settings.py to reflect on the mysql instance we have running on the other thing, and rebuild. Pushed for now.
Spoke to Zishan about current progress. Recommends moving entrypoint.sh into a seperate file because the things it sets are not config related, and should be swappable.
TODO: refactor config and deployment to use entrypoint.sh as a file
TODO: write a mysql config that will initialize the database with the tables we want + sets mysql password isthisapassword
I was not supposed to do anything to entrypoint. The internet is working. All I had to do was use the env variables specified by the mysql docker image while building and it was great!
This was very easy to do once I got input from Zishan. Until then I was floundering in the dark.
Now the mysql server is up and it’s listening for connections! Yay!
Questions: How to test the internal IP? 
Questions: how to pass variables to settings.py
TODO: Set env variables in the settings.py. Set these variables in django’s yaml as well, referring to the config map. 
Question: For host, do I have to expose the IP to everyone? (probably yes)
Question: Why don’t secrets need to be mounted?
Added MYSQL_HOST to mysql’s config
Wrote django-deployment.yml using myswl-config, mounting, etc. All worked +1
Moved django as a container in mysql-deployment because easieer to host
Moved nginx to mysql-deployment because I didn’t want to change the config that I had uploaded it with. 
Still getting a bad gateway. Why?
Could be because the gateway is connected to the nginx-deployment and not the new one. Exposed the mysql deployment on port 80. IP pending.
(Also, sb can’t access thing warning has mysteriously dissappeared. And there are no logs for django or nginx. Why?)
With new expose, still bad gateway.
Decided to get regular django up and running on it’s own deployment first, if nothing else to get something in the log. 
Hypothesis 1: Dockerfile runs, runserver not executed. Add explicit run as command in yaml
Logs are dead no matter what. Explicitly set in django settings. Set Pythonunbuffered to 0 both on docker and in yaml. No logs.
On entering the pod and runserver, takes endless amount of time with no output.
How are two of the same command able to run without conflicting with each other on the same port?

Can’t. Sending to Zishan. Going home. And I thought I was so close.
Changed nginx config - upstream was in wrong context
TODO: The app itself possibly doesn’t work - try doing it with a basic app and see if it works.


 84         'USER': os.getenv('MYSQL_USER'),  <- mysql-config
 85         'PASSWORD': os.getenv('MYSQL_ROOT_PASSWORD'), <- mysql-secret
 86         'HOST': os.getenv('MYSQL_HOST'), <-define new




30/08/18 - Thursday 

Rebuilding
The config files are doing a bad job of initialising the database. 
Initialised them, deployed files - getting inane warnings Network is not ready: Kubenet does not have netConfig. This is most likely due to lack of PodCIDR]
Quick google shows that this happens across providers when there is not enough resources (RAM/CPU). GCP Dashboard shows that our CPU Usage is about 2%. 
Reverted configmap commit, on redeploy,  Unable to mount volumes for pod timeout expired waiting for volumes to attach/mount for pod "default". 
All these error reportees had it come and go intermittently. Re-instatiated volumes, and removed new stuff from the config file.Goes back to old ‘create not found’ error. This is good.
Repasted the old config. Shows an InnoDB: Unable to lock ./ibdata1 error: 11    Changed the order of working of the file, tried gain (removed mysql_install_db)
Right now I’m placing all my configs in entrypoint.sh, How about I place it somewheer else and run it AFTER I run entrypoint.sh?
Frustrating and I’m not getting anywhere. Stashing for now.

Wrote half a docker file for the python container


set -e
    set -x

    mysql_install_db

    # Start the MySQL daemon in the background.
    /usr/sbin/mysqld &
    mysql_pid=$!

    until mysqladmin ping >/dev/null 2>&1; do
    echo -n "."; sleep 0.2
    done


29/08/18 - Wednesday

Rebuilding
Decided to leave the mysql image alone for now, will bundle django and update nginx config
Deleted nginx one-click install because I realised it was costing me too much money
Have a deployment up. Have a service up. They are talking to each other (via same-label selectors). No welcome page. Why?
Question: Does creating a service not automatically expose the port? 
Question: Does deploying an image not automatically run the image?
(from other browsing, will have to create a new image with my config if I want a custom config)
I did kubectl expose deployment nginx-deployment --port=80 --type=LoadBalancer and this now works. Why? 
TODO: re-write this command in yml format for better understanding. This file can be seen using kl edit
Hypothesis: some selector gadbad
Can create a configmap object using the nginx.conf file via kubectl create configmap nginx-config --from-file=./nginx.conf
add these volumes to deployment, patch. Should work. Doesn’t work.
Syntax errors. Fixed. Arm build fails, unable to pull. Known issue with some libraries, revert to old version to make it work. Says file is empty. On testing says syntax is okay, text is failing.
Was missing an events {} block, and now it works!! The IP gives me a 502, which is exactly what I want. Now for fixing and bundling my django app (which I think will be the hard part, but doable now with all my context. :)
TODO: move from using a docker structure to using configMap to set env variables
Using configmap. Same issues as before.
Use mariadb as the image. It works brilliantly. Next error.


24/08/18 - Friday

Rebuilding
Created new mysql image (setting env DATABASE to polls)
Built image, tagged, pushed to hub https://hub.docker.com/r/tanvibhaktasb/mysql-polls/
Rewrote mysql k8s file to use this image instead of defaults
Bunch of errors; SQL doesn’t recognise the database we want with creation, so add line to Dockerfile
MySQL doesn’t play well with the lost+found directory that is automatically present in every volume file that is provisioned - keeps aborting because data directory has files in it. See threads https://github.com/docker-library/mysql/issues/69#issuecomment-98678411  and https://github.com/docker-library/mysql/issues/186 for larger context.
Tried mounting it to a different path. Not solved.
Tried adding --ignore-db-dir flag. Nt solved.
Shifted versions (currently running 8+, shifted to 5.6 and 5.7). Not solved.
Moved to using a different image completely (mariadb over mysql). Server logs still say mysql is used. What is up? Are there caches that I’m missing flushing? (no.)


23/08/18 - Thursday

Rebuilding
Instantiate a disk, install mysql on the instance associated with the disk
Clone sample django project for kicks, set it up to work with mysql locally
Released I was working on compute engine as opposed to container engine - delete everything, start over. (Good job, Tanvi)
Deployed nginx via 1-click install on the same cluster
Deployed mysql on it’s own containers (and they’re running!)
TIL If you create just a pv-claim and attach it to the image you want the volume to belong to gcloud will instantiate a persistent disk with ext4 and attach it to a container for you. This is by default a StatefulSet.
TODO: set up custom mysql image with our databases
TODO: set up nginx with config t opoint directly to django’s app server (see below pastd config)
TODO: Package current django setting into the old docker image, push, figure out how to connect.

TODO: a mysql-persistant-disk, a mysql-password-secret


    location / {

        proxy_pass http://django;

        proxy_set_header Host $host;

        proxy_set_header X-Real-IP $remote_addr;

        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    }

upstream django {

    server 127.0.0.1:8000;

}



22/08/18 - Wednesday

Wanted to start over, because 
I felt like i was being mowed down by all the mixed concepts I had previously
I felt like I wasn’t actually learning anything (hence this doc was born)

In a new folder, I pulled image I had made two weeks ago via docker
Couldn’t see anything in my directory, proceeded to figure out what happens when an image is pulled
You need to not only pull and image but also run it. ._.
Upon running, the image would immediately exit.
The image contains both django and mysql, as opposed to django only
Does that make sense, if the image is supposed to work with my nginx-django-mysql set up in k8s? Probably not.
The command shown is ‘python3’. Need to modify docker-compose file to fix the issue.
On discussions with Karthik Nayak, decided to use k8s config file to build the django/mysql image from python. (thereby not touching the mess of thread that is docker-compose)
Deep spoke about a type of resource called a StatefulSet that is used to handle mysql in k8s. 


Rebuilding:
Wrote a file that creates a mysql service, a mysql-pv-claim and a mysql-deployment
Created a new project on GCP


