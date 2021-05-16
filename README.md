#

## Local Setup

jekyll gh-pages is currently not working with Ruby 3.0 and requires Ruby 2.7 instead.
Crossed-out lines are using Ruby 3.0 which won't work, so just skip them.

* ~~install `ruby` and `rubygems`: `pacman -S ruby rubygems`~~
* ~~setup rubygems: <https://wiki.archlinux.org/title/ruby#Setup>~~
* install `rvm`: `curl -L get.rvm.io | bash`
    * follow all instructions printed to the terminal
* install `ruby` 2.7: `rvm install 2.7`
* from now on execute all ruby commands prefixed with `rvm 2.7 do ...`
* setup jekyll environment
    ```
    rvm 2.7 do gem install jekyll bundle
    #bundle config set --local path $GEM_HOME #not needed with rvm
    rvm 2.7 do bundle install
    ```
* run jekyll locally: `rvm 2.7 do bundle exec jekyll serve`
