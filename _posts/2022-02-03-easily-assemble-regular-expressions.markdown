---
layout: "post"
title: "Easily Assemble Regular Expressions"
date: "2022-02-03 11:02"
---
<style>
  code {
    white-space : pre-wrap !important;
    word-break: break-word;
  }
</style>

# List to regular expression

Have you ever encountered a list like this and thought "I wish I could easily create a regex to cover all these possible values"?

![@ymzkei5](/images/2022/02/03/regex.1.png)
[source](https://twitter.com/ymzkei5/status/1469765165348704256)

I have a linux based method to share which makes this task easy by leveraging an incredible Perl module called `Regexp::Assemble`

## Step 1 - create a list
In our example, We'd like to capture all the possible values behind the `${` characters (we'll deal with the leading characters later), so our first step is to list the values in a text document:

```
k8s
main
sys
lower
web
env
upper
date
```

## Step 2 - assemble

Copy the following to a file (in this example it is named `regex_assemble.pl`:
```
#!/usr/bin/perl
use strict;
use Regexp::Assemble;
 
 my $ra = Regexp::Assemble->new;
 while (<>)
 {
   $ra->add($_);
   }
   print $ra->as_string() . "\n";
```

Note: you may need to [install the Perl module](https://www.howtoinstall.me/ubuntu/18-04/libregexp-assemble-perl/)

Then, we can copy our list to the clipboard and paste it into the STDIN of the invocation of our Perl script (note single quote):
```
$ echo 'k8s
> main
> sys
> lower
> web
> env
> upper
> date
> :
> ' | perl ~/scripts/regex_assemble.pl
(?:(?:low|upp)er|(?:k8|sy)s|date|main|env|web|:)
```
Notice the output:
`(?:(?:low|upp)er|(?:k8|sy)s|date|main|env|web|:)`

# Step 3 - refinement
Now we can add logic to account for preceeding characters

`j[ndi]*` will match the preceeding characters, note that this uses `0 or more of this character class` logic and not a string match, a stylistic choice for readablity to cover partial strings: `jn${..`, `jnd${`, ...

Also, it seems every string following `${` also follows with a `:` character, so we add:

`j[ndi]*${(?:(?:low|upp)er|(?:k8|sy)s|date|main|env|web|:):`

# Step 4 - use
In my case, I'm writing a [hunting rule](https://github.com/travisbgreen/hunting-rules) for use in investigating URL payloads, so a suricata rule using this regex would look like this:

`alert http $HOME_NET any -> any any (msg:"TGI HUNT Possible Log4shell Obfuscation Technique"; flow:established; http.uri; content:"${"; fast_pattern; pcre:"/j[ndi]*${(?:(?:low|upp)er|(?:k8|sy)s|date|main|env|web|:):/i"; reference:url,twitter.com/ymzkei5/status/1469765165348704256; classtype:bad-unknown; sid:2610828; rev:1;)`

And a quick check shows that it is working:
```
$ ~/scripts/suri.hunting.sh
3/2/2022 -- 11:58:01 - <Notice> - This is Suricata version 6.0.2 RELEASE running in USER mode
<...>
3/2/2022 -- 11:58:01 - <Info> - Alerts: 1
3/2/2022 -- 11:58:01 - <Info> - cleaning up signature grouping structure... complete
02/02/2022-15:10:56.628683  [**] [1:2610826:1] TGI HUNT WAF MITM of HTTPS [**] [Classification: Misc activity] [Priority: 3] {TCP} 10.96.175.18:39376 -> 111.111.111.111:80
```

Hope that helps, and feel free to [@](https://twitter.com/travisbgreen) me with feedback.





