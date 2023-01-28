# Get a domain with less than $2

Ok, you've created that awesome website. It's looking all good and you feel impressed, and you want it hosted with a custom domain. Some popular hosting platforms like Netlify, Surge and Heroku offer free custom domains, only that you are assigned a subdomain, not an apex domain. 
But what's a subdomain and what's an apex domain.

## Subdomain versus Apex domain

Whenever you deploy your applications with some of these platforms offering free basic plans you are automatically given a domain for your newly hosted site. Usually looks like
[random_name].netlify.app. This kind of domain is called a subdomain while the netlify.app is the apex domain, well that could be enough and we call it a day, you've done a great job😁.

You now have an awesome looking website, but you'd still love an awesome looking domain or your application will serve as a backend API for another frontend app, and you are required to store the same session cookies for both the backend and the frontend for authentication to avoid CSRF(Cross-Site-Request-Forgery) errors while doing stuff like authentication or serving protected resources. 

## Getting a domain 

Not too long ago I was developing an API backend that needed a hosting platform where I could quickly deploy and test out the authentication feature of the app, After some Googling, I found Namecheap offers TLDs(Top level domain) of ".xyz" for as low as $1, but retail price after a year is about $12, after one year you can wait till it expires and buy it again, but in most cases, you only need it for testing and a year is more than enough for whatever testing you might need. 

However, you could have both the frontend and the backend running in a local environment but on different ports, but you could be working in a team and need others testing it or you explicitly want it to be served over an HTTPS protocol which a local development won't provide. Unlike Cloudflare, Namecheap doesn't offer free SSL for their domain, you'd have to buy.
So you can have the domain transferred from Namecheap to Cloudflare.

Sometimes running your application in a local development might not be enough for testing or you could want others also testing for you this way you can fix those silly bugs that won't show up until in production👋.
