# MBTA Performance API

## Origins

Anyone who has ever commuted by subway (NY, DC, Boston, etc.) will have run into
mystery delays. Whether on the platform, or in a train, the delays are sudden,
and inevitably infuriating as the transit authority almost never gives riders
full information about the situation. The MBTA of Boston may indicate the cause
of the delay over their alert feed, and may describe the delays in vague terms
like "minor" or "moderate", but what do those terms actually mean? 

When I began this project, I had the ambitious goal of quantifying these
qualitative terms to their actual affects on riders' commutes. Or perhaps
modeling expected delays due to different weather conditions. After exploring
the data available from different transit authorities, I found that the MBTA of
Boston had a web API that yields historical train performance data. Nice!
