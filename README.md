# heroku-skeleton-buildpack

Skeleton buildpack; experimental.

* Next step: source snippets in the .heroku.rc.d folder to control the detect / compile / release process on a per-app basis
* Next step: Look at integrating support for shellfire dependency extraction (essentially, get a list of packages)
* Next step: domains
* Replace configuration refspec with code that detects current git branch / commitish? That way when branching we don't need to modify code for heroku pushes...
* Consider using maintenance mode during deployment to allow time for migration?
* Heroku pipelines
* How to delete unused configuration variables?
