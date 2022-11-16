---
layout: posts


title: "Jekyll: Build Locally and Push into Github"
categories: dev

date: 2022-11-15 19:26:00 +09:00

---

## Motivation

I wanted to use `jekyll-archives` in my blog to autogenerate index pages for each category.
This worked on my local machine, but when I published the changes, I soon realized that my expected category-wise index pages are not shown, and instead I got 404.

So I did some research, and soon knew that github pages only support [specific jekyll plugins](https://pages.github.com/versions/), and if you want to use more than that, you may **build locally and upload the compiled result as a static site** as explained in [this reference](http://ixti.net/software/2013/01/28/using-jekyll-plugins-on-github-pages.html).

## How to Build Jekyll Locally with rake(ruby make)

Provide this `rakefile` in our jekyll root directory:
```ruby
require "rubygems"
require "tmpdir"

require "bundler/setup"
require "jekyll"

GITHUB_REPONAME = # my repo goes here

desc "Generate blog files"
task :generate do
  Jekyll::Site.new(Jekyll.configuration({
    "source"      => ".",
    "destination" => "_site"
  })).process
end

desc "Generate and publish blog to gh-pages"
task :publish => [:generate] do
  Dir.mktmpdir do |tmp|
    cp_r "_site/.", tmp

    pwd = Dir.pwd
    Dir.chdir tmp

    system "cd .."
    system "git init -b built"
    system "git add ."
    message = "Site updated at #{Time.now.utc}"
    system "git commit -m #{message.inspect}"
    system "git remote add origin https://github.com/#{GITHUB_REPONAME}.git"
    system "git push origin built --force"

    Dir.chdir pwd
  end
end
```

I slightly modified the code so that I can separately upload on different branches - source files on `main` and build output on `built`[^1].

[^1]: I feel sorry for that name. Maybe `build` would be a better choice, but hey, I'm not an English native person.

Basically, this code enables `rake generate` to build the blog (into `/_site/`, if default settings), and `rake publish` to build then initialize `/_site/` as git repo and push it into the remote github repo, branch `built`. The commit message is generated with current time.

### Still Problem

This solution, while apparently correct, still did not work for me.
I failed to figure out why, but `rake generate` appears to be omitting `jekyll-archives` I wanted to apply, and the build result is different from that of `jekyll build`.

To get my desired behavior, I simply changed the task body into invocation of system command.
```ruby
task :generate do
  system "jekyll build"
end
```

I also tried to get the API docs of Jekyll package used inside Ruby, but I couldn't find one.
Should admit also that I am too lazy to dig into the Jekyll code (or even learn Ruby) just for publishing a blog,
I wanted to keep my environment setting step minimal and do the real works as fast I can - or at least for this moment.

## References

* [Dependency versions \| Github Pages](https://pages.github.com/versions/)
* [Using Jekyll plugins on GitHub Pages](http://ixti.net/software/2013/01/28/using-jekyll-plugins-on-github-pages.html)
* [Jekyll Github Page 미지원 Plugin 빌드 배포](https://jr-developers.github.io/blog/2020/12/08/blog1.html) (Korean)
