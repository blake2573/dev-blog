# Shopify-ifying your web application
### 2025-11-07

I’ve recently had need to throw myself headfirst into the land of Shopify, with the end goal of finding the path of least resistance to convert an existing web application into a Shopify-App-Store-compatible theme extension. As a SaaS platform, the key criteria for us to adhere to through this migration were: to ensure we could still use the web app as a web app - a standalone mode will be most compatible for some clients; handle full (read: as much as possible) configurability of themes and language and pricing so clients can match the layout to their hearts’ content; and to cater for full interactivity with the Shopify ecosystem of cart and product functionality so there’s a seamless transition between our app and their storefront.

Through this process, I’ve dug into documentation, forums, blog posts, and of course a multitude of AI models to figure out the best way forwards, but there were still so many things that didn’t seem clear - so many dead ends that didn’t have obvious answers - so I thought I would put together some information on how we went about it, the issues we came across, and the solutions we uncovered.

I’ve split this journey up into 3 key parts: Some first setup steps like authentication and liquid templating, configuring Shopify interactions such as product creation and add to cart, and some frustrating technical pain points/oversights. With any luck, this might save someone else the pain of figuring the process out themselves.

> P.S.: This is an AI-free zone; all slop uncovered within is organic human-generated slop
> 
