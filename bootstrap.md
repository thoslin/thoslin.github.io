# set up octopress

sudo apt-get install ruby-dev

http://octopress.org/docs/setup/

# clone repository

mkdir _deploy
cd _deploy
git clone master

# new post

rake new_post["title"]
rake generate
rake preview
rake deploy

# save changes to source

git add .
git commit -m "update post"
git push origin source
