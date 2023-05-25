If you have ever worked with Linux, then you know the struggle of properly configuring and securing machines. Each one is different and generally requires it's own set of configurations. Going further, once you get the logs into a SIEM, the adventure continues still as you have to find ways to parse and make the logs useful as they come in from many different ways, and in many different structures.

While this topic is a huge topic and we could spend months going over all the ways to secure, and work with linux logs and parse them(follow and connect with me on LinkedIn much more to come), this article is designed to focus more on a quick way to log command line activity to give you more visibility insight on your machines.

What makes this way ideal? This concept uses an open source application that loads a dynamic library to capture activity from users and running processes on TTY. It also works on virtualized kernels or where you can't properly load AuditD, which is the ideal way to do Linux system auditing, but requires much more configuration and a learning curve to configure. It works by loading a kernel module allowing capture of much more system activities such as syscalls(), which can be used to even catch buffer overflows and exploits, but that's for another series of articles. I mention that as this concept isn't fool proof, and there are ways for an attack to circumvent this logging.

With that out of the way, lets start logging our system activity. For that, we will use this tool for logging: https://github.com/a2o/snoopy - the readme has a quick copy and paste one liner that can be ran as root.

The nice thing about this tool is how easy it is to configure. You can easily exclude items from being logged, structure the log output and define where the logs are sent. You can find this under /etc/snoopy.ini

The examples are very straight forward and easy to follow so I won't spend any time on them, but in regards to what I picked, I used this as my logging format:
![](https://media.licdn.com/dms/image/D4E12AQHIUJgIh0er0w/article-inline_image-shrink_1500_2232/0/1681263519495?e=1690416000&v=beta&t=1DU_KPteupMb3760wGjskdFK1Rk_k8Nd9KIBrd8CiwA)

Why? With Sentinel there are many ways you can parse. You can use the parse() function or even the split() function and extract() if you like regex. So many options. It really comes down to preference, but the thing to keep in mind is that you need to have characters that allow you to pick where to start parsing from and where to end. Because these can be very dynamic, I skipped using common delimiters due to the chance that if someone used a , or another character that so commonly used in a commandline for example, it would break my parsing based on characters.

With Snoop installed, and our config file in place(all you have to do is save the file. no need to reload a service but you do need a reboot after install) we start seeing our logs come in to sentinel following our format:
![](https://media.licdn.com/dms/image/D4E12AQFtLSEU9kfvHA/article-inline_image-shrink_1500_2232/0/1681263147938?e=1690416000&v=beta&t=kiqdzUxb3qLLJjByYknbd5h2PvEpomtFJJykDu3WW5Q)

Now we still are not finished yet. We need to parse out our variables so we can search for them.

This is what I used. I set my regex in a way that would capture everything and anything up to a space:
![](https://media.licdn.com/dms/image/D4E12AQGrTYb5gpVSAw/article-inline_image-shrink_1500_2232/0/1681263390001?e=1690416000&v=beta&t=zZgDzTuOjFsW3EgklqOz5zen68XTO1PBTDorX8ZHVhg)

By running this we can test and see that this works:
![](https://media.licdn.com/dms/image/D4E12AQEd609EBHjRkA/article-inline_image-shrink_1500_2232/0/1681263635013?e=1690416000&v=beta&t=ojB956nItHC2YqZu2mzSZb4wnCEpCvfSVzwFdXuCKzk)

So now what? We don't want to run this all the time so lets make this into a function!

![](https://media.licdn.com/dms/image/D4E12AQFvha-puApcgg/article-inline_image-shrink_1500_2232/0/1681263880866?e=1690416000&v=beta&t=_0YL2LH_-DOd2SFQirdke-850ItfQia7d6FD9K1Rbdk)

There we go. We can now call this table and all our data is parsed out ready to use.

So we have all our command line data. What can we do with this? Well one we could look for certain commands or CWDs and alert on activity but this could be noisy.

Another idea is we could also baseline this activity per machine two different ways. One being by total commands run and then look for a spike indicating user activity that's not normal on a machine that should generally always do the same actions while not touched.

Another is per machine, look back at all the commands ran and alert on commands that have not been seen in x amount of time or if certain commands are used in a specified time period, which can happen when recon is going on.

Hopefully this provides you a way to start logging your linux machines and learning how to parse out some data for searching.

If you are new to my content, be sure to follow/connect with me on LinkedIn and other social media for new ideas and solutions to complicated real world problems.
