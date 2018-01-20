title: Developing on github
published_date: "2017-12-30 17:23:30 -0500"
data:
    tags: ["programming"]
---
Setting up a GitHub Project
Setup appveyeor and travis accounts
- https://github.com/japaric/trust
- https://llogiq.github.io/2016/07/05/travis.html

Setup Bash for Windows
sudo apt-get install ruby
sudo apt-get install ruby-dev (latter to avoid `require': no such file to load -- mkmf (LoadError)')
sudo apt-get install build-essential (latter to avoid “make: not found”)
sudo gem install travis
travis encrypt -r epage/relint GH_TOKEN=KEY

Badges
AppVeyor: "settings"->"badges"
Travis: right on the top of the screen
Line count: https://github.com/Aaronepower/tokei#badges


Fork
clone your fork
git remote add upstream <project you forked>
git checkout -b <name>
...
git push --set-upstream origin <name>
git push

Update from master
git checkout master
git rebase upstream/master
git checkout <name>
git rebase master
