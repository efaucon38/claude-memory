
Objet : quelques informations concernant la gestion du temps en trading

Source : https://www.kbasm.com/blog/forex-time-knowledge

Knowledge on time and time zone in Forex trading
Forex trading time is one of the most confusing area in currency trading market. Not only do we have London session, New York session, Asia session, but also we have New York close candlestick bar, London close candlestick bar. And even worse, after DST creeps in, the problem becomes more complicated. In this article I will try to explain all I know about Forex trading time to make it simpler to understand and remember.

Please note, in this article I always use GMT, but it's safe to replace all GMT with UTC, they are same here.

DST -- Daylight Saving Time
For more details on Daylight Saving Time, please read here and here. DST is a very confusing thing around the globe. Fortunately China stopped DST in 1980s, but unfortunately the most important Forex trading countries such as the UK and the US are still using DST.

In standard time period, London time zone is GMT+0, and New York time zone is GMT-5. During Daylight Saving Time, London time zone is GMT+1, and New York time zone is GMT-4. Note that the time zone difference between London and New York is always 5 hours, despite of DST or not. This is because that both place enters and quits DST at the almost same time (not exactly, but the period that one is in DST while another is not is very short so we can ignore it). The stable time zone difference helps us to remember the DST. We can always remember the time zone of London and subtract 5 for New York time zone.

New York candlestick chart close time
Some few traders love London close chart, but New York close chart is the most important because most trades use it. Using proper timed bars is important because different timed bars can give different price action patterns on H4 and above time frames.

New York session ends at 17:00, NY time. Since Forex is 24 hours trading, 17:00 is both the open and close time for a daily bar. 17:00 is GMT 22:00 in standard time, and 21:00 in DST.

MetaTrader server time zone for NY close chart
You may think setting time zone to -5 on MT4 server will provide NY close chart. That's completely wrong. MT4 server time zone for New York close chart is GMT+2 in standard time, and GMT+3 in DST. Sound weird? Difficult to remember? Let's understand the mechanism.

A bar on daily chart should open at 0:00 and close at 24:00 (or 0:00, they are the same thing). That means to provide NY close chart, at the time of 17:00 NY time, the MT4 server should be at 24:00 in its time zone. As I have said, 17:00 NY time is 22:00 GMT in standard time and 21:00 GMT in DST. To be at 24:00 on the MT4 server side, it must be 2 hours ahead in standard time (22+2=24) or 3 hours ahead in DST (21+3=24). Then we can get the conclusion, MT4 server that provides NY close chart should be in time zone GMT+2 in standard time and GMT+3 in DST time.

There are several well know brokers that provide NY close chart on practice account. One is Oanda, another is FXDD.

Market time of London and New York
Both London and New York trade time is from 8:00 to 17:00, local time. So in standard time, London market is GMT 8:00 to 17:00, New York is GMT 13:00 to 22:00. In Daylight Saving Time, London market is GMT 7:00 to 16:00, New York is GMT 12:00 to 21:00. If you trading on open break out, you must understand the time.

Matching your offline bar data to MetaTrader
I'm always in this situation: I download hourly bar data from Oanda or Dukascopy, which time is GMT+0. Then I use my ftap system to back test some strategies. Then I need to verify the result visually by checking the chart in MT4. The problem is MT4 server time is not GMT+0. It can be GMT+2, or GMT-5, but not GMT+0. So I need to adjust the time. The adjust is simple, just add the time zone difference.

For example, if my bar data is in GMT+0, and MT4 time zone is GMT+2, for a trade happens at 19:00 in my back test data, it's 21:00 on MT4. If MT4 time zone is GMT-5, then it's 14:00.
