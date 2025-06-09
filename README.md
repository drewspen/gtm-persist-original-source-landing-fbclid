# gtm-persist-original-source-landing-fbclid
GTM objects to persist original source, landing and fbclid as 1st party cookies 

JSON file that can import into a scratch/sandbox GTM Google Tag Manager workspace. The GTM objects use the GTM tag template Cookie Creator to address CSP Content Security Policy SHA-256 key issues. 

1) A folder for persisting the site landing referral source as a first party cookie. Both the first value associated with the user and the current / last value. Why? Because often marketers put 'garbage' into &utm_source= and the site landing referral source helps figure out what happened. 

2) A folder for capturing the site landing URL  as a first party cookie. Both the first value associated with the user and the current / last value. Why? Identifying the start of user journey across domains when there is a multi domain site and many of the subdomains have a '/' path). 

3) A folder for persisting &fbclid= as a first party cookie. Both the first value associated with the user and the current / last value. This is illustrative for how you can set up any incoming URL query string parameter.

4) An Analytics folder that has the shared event settings. You'll want to turn the shared event settings into event and user custom definitions in GA4.
