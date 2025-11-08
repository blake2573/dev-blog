## Liquid templating

[Liquid templates](https://shopify.dev/docs/api/liquid) define how Shopify App Theme extensions interact with the storefront, including how to render content, make configurable options available within the Shopify Theme Editor for merchants, and more. They are a **must** for any theme extension as at time of writing, and so were a key hurdle to overcome in the process of moving towards a Shopify embedded implementation. If you’re building from scratch, with intentions to only live within Shopify, then maybe you would build out your app components entirely within Liquid templates, rendering dynamic blocks directly instead of building a standard web application. For us, however, we already *had* a functioning web application - one that still needed to be maintained for purposes of providing a standalone experience outside of Shopify - and so migrating entirely to Liquid componentisation wasn’t an option.

So what to do?

The best way to explain it, is with an example. In your `index.tsx` file, for a standard React application, you might see a render function similar to the below.

```tsx
render(<App />, document.getElementById('root')!)
```

This same render function forms the foundation of setting up a Liquid template so that your application can be both a web app and a Shopify App, at the same time, without overwhelming ongoing maintenance to keep the template functional.

In order to get this working, you should create a custom app render function, essentially the same as above, but without a different target element instead of `'root'` . This element will be a wrapper element, defined within your Liquid template, so that wherever your app is installed within a merchant’s storefront, your app will be able to render with the appropriate target placement. You would simply place a snippet, similar to the below, inside your Liquid template (For a full sample snippet, see [code snippet 1](https://www.notion.so/Part-I-Liquid-templates-and-authentication-29f120157dac80b99ecbdd91a201a69a?pvs=21) at the end of this post)

```tsx
{{ 'your-app-bundle-name.css' | asset_url | stylesheet_tag }} // Load your app's css

// Preload not strictly necessary, but much more performant for TTL
{{ 'your-app-bundle-name.js' | asset_url | preload_tag: as: 'script' }}
<script defer src="{{ 'your-app-bundle-name.js' | asset_url }}"></script>

<div id="your-app-container" />

<script>
  document.addEventListener('DOMContentLoaded', function() {
    if (window.MyShopifyApp && typeof window.MyShopifyApp.init === 'function') {
	  // MyShopifyApp will be replaced with whatever your build output config names the default export
      window.MyShopifyApp.init({
        container: '#your-app-container', // Must match the id of the above div
        shop: '{{ shop_domain }}',
        authEndpoint: '{{ auth_endpoint }}',
        timestamp: {{ current_timestamp }}
      });
    }
  });
</script>
```

Congratulations! You’re web app can now be deployed and render as a theme extension within Shopify. You’re a long way from done, but this first step was a struggle to figure out initially. It’s unclear whether this is considered a hack or not, as I haven’t really been able to find other posts about this, but I haven’t had any issues with handling it this way, and it keeps things much simpler for me in the long run.

## Authentication

Our application is secured with OAuth tokens in both the frontend and admin sites, where clients authenticate with standard login practices. The intent being that clients would be the sole users of the platform, working alongside their customers, and so login requirements made the most sense at the time. However, for a Shopify-compatible customer-facing application, there can’t be the friction of a login experience and instead it should all be handled behind the scenes. Thus, we come quickly to confusion the first: authentication. More specifically, the confusing fact that there are multiple different requirements on *how* to authenticate, depending on where the app will live within Shopify. 

> For both of the below-listed authentication methods, it is important to note that the authentication with Shopify should happen *only once* at the start of a session. If you use any form of internal auth token to validate API requests, you will need to ensure that your internal token is generated once you have validated Shopify’s initial authentication request, and use only your internal token for subsequent requests.
> 

### Shopify Admin - AppBridge

The Shopify Admin section can be best described as the admin panel for merchants within Shopify. This is where they set up their themes, configuration, app installations, view metrics, etc. When a merchant installs a new app from the App Store, the first place they will see this app is within the Shopify Admin section. Apps within the admin panel are typically rendered within an iframe, and so exist as a standard website in their own right.

For authentication in the Shopify Admin panel, the requirement is to use the [AppBridge](https://shopify.dev/docs/api/app-bridge) service. Simply put, this service is required to initialise and generate session tokens on behalf of the Shopify merchant (See [code snippets 2](https://www.notion.so/Part-I-Liquid-templates-and-authentication-29f120157dac80b99ecbdd91a201a69a?pvs=21) and [3](https://www.notion.so/Part-I-Liquid-templates-and-authentication-29f120157dac80b99ecbdd91a201a69a?pvs=21) for sample React setups). Once you have your hands on the session token, you can validate the token you’ve got is a real token (AppBridge uses hmac tokens based on your app’s API client id + secret, and the merchant’s shop url, to generate signatures), and continue from there to either generate an internal auth token response, as you normally would, or proceed without further authentication if you’re playing in hardcore mode.

For fresh installs of an app, you might have a requirement to perform some additional OAuth handshakes with Shopify as well. At time of writing, there exists an endpoint at `https://{shop}/admin/oauth/authorize` that you will redirect to, through the AppBridge client in your frontend, to perform this handshake. This will require some query params, including your app details, and a `redirect_uri` so that Shopify can verify and respond appropriately. Once you’ve received back a successful authentication request to your callback endpoint, you’ll need to perform some secondary validation to confirm success, and then redirect to the Shopify admin manually from your callback endpoint. Don’t ask me why successful authentication can’t provide this redirection URL for you, but it’s straightforward enough to parse out using the shop name and app handle details `https://{shop}/apps/{app_handle}` . If Shopify change the format of this in the future, good luck, I guess.

And happy days - your admin app can now authenticate successfully! 🎉🎉 

But can you reuse that same flow for storefront authentication? Of course not. Obviously you’ve got to use the completely different App Proxy instead for that.

### Shopify Storefront - App Proxy

The [Shopify App Proxy](https://shopify.dev/docs/apps/build/online-store/app-proxies) for storefront authentication  has some decent documentation, but I didn’t feel the examples helped as much as they could have.  Below are the instructions as at time of writing, but obviously always check the latest information from the link above in case this has changed.

![Shopify App Proxy docs](image.png)

In essence, this all makes enough sense - your app sends an API request to the Shopify app proxy, then it receives and redirects through to the appropriate endpoint, as defined in your app’s configured `.toml` file. In doing so, it also provides session tokens and shop data so that you can validate that requests to your API are valid Shopify customers in the storefront, as well as identify *which* storefront they have accessed from.

What this means for authentication is fairly simple - instead of requiring a customer to sign into the app to get an internal auth token, you can instead hit the app proxy as the very first thing that happens when your app loads in a Shopify context, validate it’s a real Shopify customer from the forwarded app proxy data, and generate the auth token based on this handshake. Everything else within your app can then just work off your internal auth, without having to touch the app proxy again. I’ve left a sample of an authentication function in [code snippet 4](https://www.notion.so/Part-I-Liquid-templates-and-authentication-29f120157dac80b99ecbdd91a201a69a?pvs=21) that should be usable for this process. My real-world function is built out a bit more with other pieces, but this snippet forms the bones of how the storefront auth flow works, so expand upon it how you wish!

…and that’s a wrap for this part. There’s probably plenty of things that I have forgotten to cover here, but I’ll look to capture anything else important in Part Two, where I’ll go over more detail on interacting with Shopify from your app.

### Code snippets

The below code snippets are customised for this blog, but represent essentially the same logic flow as is required, and should be able to be incorporated to your site easily enough.

1. A liquid template sample for rendering your standard web application within Shopify Storefront

```tsx
{%- liquid
  assign shop_domain = shop.permanent_domain
-%}

{{ 'your-app-bundle-name.css' | asset_url | stylesheet_tag }} // Load your app's css

// Preload not strictly necessary, but much more performant for TTL
{{ 'your-app-bundle-name.js' | asset_url | preload_tag: as: 'script' }}
<script defer src="{{ 'your-app-bundle-name.js' | asset_url }}"></script>

<div id="your-app-container" />

<script>
  document.addEventListener('DOMContentLoaded', function() {
    if (window.MyShopifyApp && typeof window.MyShopifyApp.init === 'function') {
	  // MyShopifyApp will be replaced with whatever your build output config names the default export
      window.MyShopifyApp.init({
        container: '#your-app-container', // Must match the id of the above div
        shop: '{{ shop_domain }}',
        authEndpoint: '{{ auth_endpoint }}',
        timestamp: {{ current_timestamp }}
      });
    }
  });
</script>
```

```tsx
import { ApplicationContextProvider } from '@shared/providers/ApplicationContextProvider'
import { App } from './app'

type ShopifyConfig = {
  container: string
  shop: string
  timestamp?: number
}

const initShopifyApp = (config: ShopifyConfig) => {
  const container = document.querySelector(
    config.container,
  )
  if (!container) {
    console.error(
      `Container ${config.container} not found`,
    )
    return
  }
  
  // Context provider optional.
  // I found it useful to keep track of some context data later, but not strictly necessary
  render(
    <ApplicationContextProvider
      shopifyConfig={config}
    >
      <App />
    </ApplicationContextProvider>,
    container,
  )
}

export default initShopifyApp
```

2. A wrapper service for App Bridge, to handle initialisation and redirection functionality and keep App Bridge contained

```tsx
import { createApp } from '@shopify/app-bridge';
import { Redirect } from '@shopify/app-bridge/actions';
import { getSessionToken } from '@shopify/app-bridge/utilities';

export interface ShopifyServiceConfig {
  apiKey: string;
  host: string;
}

export class ShopifyService {
  private appBridge: ReturnType<typeof createApp> | null = null;
  private isShopifyIframe = false;
  private config: ShopifyServiceConfig;

  constructor(config: ShopifyServiceConfig) {
    this.config = config;
  }

  async initialize(): Promise<void> {
    // Detect if we're running inside a Shopify iframe
    const host = this.config.host;
    const shopDomain = this.getShopFromHost();

    // Check if we're in an iframe and have Shopify parameters
    const inIframe = window !== window.parent || window !== window.top;
    const hasShopifyParams = Boolean(shopDomain && host);

    if (inIframe && hasShopifyParams && host) {
      this.isShopifyIframe = true;

      // Initialize Shopify App Bridge
      try {
        this.appBridge = createApp({
          apiKey: this.config.apiKey,
          host,
        });
      } catch (error) {
        console.error('Failed to initialize Shopify App Bridge:', error);
      }
    }
  }

  public async getSessionToken(): Promise<string | null> {
    if (!this.isShopifyIframe || !this.appBridge) {
      return null;
    }

    try {
      // Fetch a new session token. You could handle caching of this if/as required
      const token = await getSessionToken(this.appBridge);
      return token;
    } catch (error) {
      console.error('Failed to get Shopify session token:', error);
      return null;
    }
  }

  /**
   * Decode the host parameter to get the shop domain
   */
  getShopFromHost(): string {
    try {
      // Host parameter is base64 encoded shop domain
      const decodedHost = atob(this.config.host);
      return decodedHost.replace('.myshopify.com', '');
    } catch (error) {
      console.error('Failed to decode host:', error);
      return '';
    }
  }

  public redirectToAuth(url: string): void {
    if (!this.appBridge) return;

    // Use Shopify App Bridge Redirect action for OAuth
    const redirect = Redirect.create(this.appBridge);
    redirect.dispatch(Redirect.Action.REMOTE, url);
  }
}

```

3. An Admin Context provider for React, to help conditionally run this authentication flow. Used in tandem with the service defined above

```tsx
import { ShopifyService } from '~/services/shopify';
import { createContext, ReactNode, useEffect, useState } from 'react';

interface IShopifyAdminContext {
  shop: string;
  isAuthenticated: boolean;
  appBridge: ShopifyService | null;
  isLoading: boolean;
}

export const ShopifyAdminContext = createContext<IShopifyAdminContext | null>(null);

interface ShopifyAdminProviderProps {
  children: ReactNode;
}

export const ShopifyAdminProvider = ({ children }: ShopifyAdminProviderProps) => {
  const context = useShopifyAdminProvider();

  return (
    <ShopifyAdminContext.Provider value={context}>
      {context.isAuthenticated ? (
        children
      ) : (
        <LoadingIndicator />
      )}
    </ShopifyAdminContext.Provider>
  );
};

const useShopifyAdminProvider = () => {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [isLoading, setIsLoading] = useState(true);
  const [shopifyService, setShopifyService] = useState<ShopifyService | null>(null);

  // Retrieve shop config from parent context, or however you wish
  const { shop, isEmbedded, host } = useApplicationContext();

  if (!shop || !host) {
    throw new Error(
      'ShopifyAdminProvider requires shop and hmac and timestamp from ApplicationContext',
    );
  }

  // Initialize App Bridge when component mounts
  useEffect(() => {
    const initializeAppBridge = async () => {
      if (!host) return;

      try {
        const service = new ShopifyService({
          apiKey: SHOPIFY_API_KEY, // Your app's Client ID from Shopify developer dashboard
          host,
        });

        await service.initialize();
        setShopifyService(service);
      } catch (error) {
        console.error('Failed to initialize App Bridge service:', error);
      }
    };

    if (isEmbedded) initializeAppBridge();
  }, [host, isEmbedded]);

  // Authenticate when App Bridge is ready
  useEffect(() => {
    const authenticateMerchant = async () => {
      if (!shopifyService) return;

      setIsLoading(true);

      try {
        // Get session token from App Bridge
        const sessionToken = await shopifyService.getSessionToken();

        if (!sessionToken) {
          console.warn('No session token retrieved from Shopify App Bridge');
          return;
        }

        // Send to your backend for validation and get your JWT token back
        const response = await shopifyAuth({
          sessionToken,
          shop: shop || undefined,
        });

        // Handle response from your API

        setIsAuthenticated(true);
      } catch (error: any) {
        if (error?.data) {
          const errorData = error.data;

          // State 0: 403 Forbidden - Store not installed, requires Shopify OAuth
          if (error?.status === 403 && errorData.authenticated === false && errorData.authUrl) {
            shopifyService.redirectToAuth(errorData.authUrl);
            return;
          }
        }
      } finally {
        setIsLoading(false);
      }
    };

    authenticateMerchant();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [shopifyService, shop, host]);

  return {
    shop,
    isAuthenticated,
    isLoading,
    shopifyService,
  };
};

// Usage example - conditionally wrap the entire app in ShopifyAdminProvider
// based on detection of Shopify Admin context 
export const App = () => {
  const appContext = detectAppContext();
  const appContent = <>...</>

  switch (appContext.mode) {
    case ApplicationMode.SHOPIFY_ADMIN:
      return <ShopifyAdminProvider>{appContent}</ShopifyAdminProvider>;
    case ApplicationMode.STANDALONE:
    default:
      return appContent;
  }
}
```

4. A sample authentication function that uses Shopify App Proxy to authenticate customers when accessing the theme extension in the storefront

```tsx
const authenticateCustomer = async () => {
  setLoading(true)

  try {
    // Use the app proxy URL - Shopify will automatically add HMAC validation
    // IMPORTANT: Anything after `/apps` in the below route should match your
    // backend API's auth endpoint, as this is how Shopify proxy works.
    // ie. below would map to your internal `/shopify/storefront-auth` endpoint
    const proxyUrl = '/apps/shopify/storefront-auth'

    // Build query parameters
    const params = new URLSearchParams({
      shop,
      timestamp: timestamp?.toString() || '',
    })

    // Make request through Shopify's app proxy (automatically HMAC signed)
    const response = await fetch(
      `${proxyUrl}?${params}`,
      {
        method: 'GET',
        headers: {
          'Content-Type': 'application/json',
          Accept: 'application/json',
        },
      },
    )

    // This response will be _your_ API's response
    // The app proxy just proxies through Shopify to your backend and returns the response
    const data = await response.json()

    if (response.ok && data.success) {
      // Handle success path
      setIsAuthenticated(true)
    } else {
      // Handle failure path
      setIsAuthenticated(false)
    }
  } catch (error) {
    console.error(
      'Storefront authentication error:',
      error,
    )
  } finally {
    setLoading(false)
  }
}
```