As threat hunters, analysts and even engineers, we are tasked with finding threats and ensuring that we have all the data that we need in order to maintain a level security with the data that we use every day.

Sometimes we lose that data. It stopped showing up and we are unware that there is an issue. As SIEM engineers, one of our jobs is and should be, ensuring that the logs are coming in as expected, but how do we do that? Especially in Sentinel which doesn't have a simple detection switch?

In my own environment and clients that I manage, I like to use many different ways to get the whole picture and have the ability to quickly drill down and find issues. (if you are not currently, be sure to checkout and use my Sentinel Dashboards at https://github.com/truvis/Sentinel which makes this easy)

One of the newest ideas that came to my mind was how could I use a time chart and display data week over week and see if the trend was the same?

![](https://media.licdn.com/dms/image/D4E12AQFYXJXgZaHMCg/article-inline_image-shrink_1500_2232/0/1680978954512?e=1690416000&v=beta&t=OAUG_J31sGqFQ22iYx19A25Q01gzq09hb4GFzoG4y8Y)

One of the issues with this is that if you have ever used or worked with a timechart graph, they work with time and you can't exactly over lay different times as shown below:
![](https://media.licdn.com/dms/image/D4E12AQE_QPHjzmb78Q/article-inline_image-shrink_1500_2232/0/1680978243378?e=1690416000&v=beta&t=OizCmCRWqdTIVXzS9Gea68Pms754lkqpvwbYUByTskI)

The next question is, how does one move one of the lines over? After all, isn't that all we really need to do? If we think about it, we really just need a static index point and align two sets of events to that fixed number array and increment.

NOTE: Now I am pretty sure there is a way to do actual indexing in KQL and assign and create a dynamic array using make_set() or along those lines. For the sake of time, I never could find an example that I saw which that type of logic was done, so I went another route looking through all the Sentinel KQL prebuild functions.

Looking over the functions trying to find something date wise that would allow a date shift, I saw datetime_part(). What does this function do? It allows us to take a date and time and extracts the requested date part as an integer value. Why does this matter? Think arrays. How do they work? Remember those silly programming classes we took in college? They actually matter!

Previously, we talked about how we wanted to build an index and use that as a way to control the data.

![](https://media.licdn.com/dms/image/D4E12AQEKh__xUomP3A/article-inline_image-shrink_1500_2232/0/1680978781560?e=1690416000&v=beta&t=ffRCVpoYhIwQiZIh3akq8y498FHy1BvsBNxAQGzd7PI)

Looking at the the options that we have, the one that stands out the most is the "dayOfYear". Why? Years have 365 days in them, each with a unique number which would allow us to shift the graph points now that everything is an integer.

Now that we have a cheap type of array that is integeer based, we can do some fancy logic and shift the previous data to align with our current data by 7 to account for the data being 7 days behind current:

> | summarize Count = count() by Day=datetime_part("dayOfYear", TimeGenerated)+7)

And with that we have the following:

![](https://media.licdn.com/dms/image/D4E12AQF7plqsBVYCCQ/article-inline_image-shrink_1500_2232/0/1680979375204?e=1690416000&v=beta&t=DQwAx4UVA67oGRy3WoR4-uwqqtV8wBtR3EVM5hRDIOk)

![](https://media.licdn.com/dms/image/D4E12AQH4aJWphaVRxQ/article-inline_image-shrink_1500_2232/0/1680979416817?e=1690416000&v=beta&t=TXiw1d5t4zu6eksYSKso3MU0IzNU0pVuAFZL1OFenzo)

And that is exactly what we want to see. We should see a tight line showing that the data ingestion has not changed.

To answer challenge question 2, the issue we will have is that when we hit the end of the year, the shift will make the days 366, 267, ect.. when the current will start at 1, 2, 3 ect... and that's why the ideal approach would be to build an array and start at 0 and then assign the count events.

NOTE: These charts are built into my dashboards located at: https://github.com/Truvis/Sentinel

If you are new to my content, be sure to follow/connect with me on LinkedIn and other social media for new ideas and solutions to complicated real world problems.
