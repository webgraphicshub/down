# Cloudflare Workers Guide: Redirect Error 1033 to Down Page

This guide will help you set up Cloudflare Workers (Free Tier) to redirect visitors to a custom "server down" page when your server is offline (Error 1033).

## What is Error 1033?

Error 1033 occurs when Cloudflare's Tunnel cannot reach your origin server (when your PC is shut down). Instead of showing this error, we'll redirect users to a custom downtime page.

---

## Prerequisites

- A Cloudflare account (free tier works)
- A domain managed by Cloudflare
- A subdomain for your down page (e.g., `down.webgraphicshub.com`)

---

## Step 1: Set Up Your Down Page Subdomain

### 1.1 Upload Your Down Page
First, upload your "Server is Resting" page to a reliable hosting service that's always online:
- **GitHub Pages** (recommended for static sites)
- **Cloudflare Pages** (also free)
- **Netlify** (free tier available)
- **Vercel** (free tier available)

### 1.2 Configure DNS for Down Page
1. Log in to your Cloudflare dashboard
2. Select your domain (e.g., `webgraphicshub.com`)
3. Go to **DNS** → **Records**
4. Add a CNAME record:
   - **Type**: CNAME
   - **Name**: `down` (this creates `down.webgraphicshub.com`)
   - **Target**: Your hosting URL (e.g., `username.github.io`)
   - **Proxy status**: Proxied (orange cloud)
   - Click **Save**

---

## Step 2: Create a Cloudflare Worker

### 2.1 Navigate to Workers
1. In your Cloudflare dashboard, click **Workers & Pages** in the left sidebar
2. Click **Create application**
3. You'll see a page with "Ship something new" - on the **left sidebar**, click **Create a Worker**
   - Note: Don't click the options in the center (GitHub, GitLab, Hello World, etc.) - those are for Pages
   - The "Create a Worker" option is in the left sidebar
4. Give it a name like `error-1033-redirect`
5. Click **Deploy**

### 2.2 Edit the Worker Code
1. After deployment, click **Edit code**
2. Replace the default code with the following:

```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const url = new URL(request.url)
  
  // List of domains that should redirect on error
  const protectedDomains = [
    'server.webgraphicshub.com',
    'dev.webgraphicshub.com'
    // Add more domains as needed
  ]
  
  // Down page URL
  const downPageUrl = 'https://serverdown.pages.dev/'
  
  // Check if this is one of your protected domains
  if (protectedDomains.includes(url.hostname)) {
    try {
      // Try to fetch from origin
      const response = await fetch(request)
      
      // Check if we got an error response
      if (response.status === 502 || response.status === 503 || response.status === 504) {
        // Redirect to down page
        return Response.redirect(downPageUrl, 302)
      }
      
      // Check for Cloudflare error page (Error 1033)
      const contentType = response.headers.get('content-type')
      if (contentType && contentType.includes('text/html')) {
        const text = await response.text()
        if (text.includes('Error 1033') || text.includes('Argo Tunnel error')) {
          return Response.redirect(downPageUrl, 302)
        }
        // Return the original response if no error detected
        return new Response(text, {
          status: response.status,
          statusText: response.statusText,
          headers: response.headers
        })
      }
      
      return response
    } catch (error) {
      // If fetch fails completely, redirect to down page
      return Response.redirect(downPageUrl, 302)
    }
  }
  
  // For non-protected domains, just pass through
  return fetch(request)
}
}
```

3. Click **Save and Deploy**

---

## Step 3: Add Worker Route

### 3.1 Create a Route
1. Go back to **Workers & Pages**
2. Click on your worker (`error-1033-redirect`)
3. Click the **Triggers** tab
4. Under **Routes**, click **Add route**
5. Configure the route:
   - **Route**: `*webgraphicshub.com/*` (matches all paths on your domain)
   - **Zone**: Select your domain from the dropdown
   - Click **Save**

### 3.2 Add Routes for Subdomains (Optional)
If you want to protect specific subdomains, add additional routes:
- `*www.webgraphicshub.com/*`
- `*api.webgraphicshub.com/*`
- etc.

> **Note**: Do NOT add a route for `down.webgraphicshub.com` to avoid redirect loops!

---

## Step 4: Advanced Configuration (Optional)

### 4.1 Custom Error Detection
You can customize the error detection logic. Here's an enhanced version:

```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const url = new URL(request.url)
  const downPageUrl = 'https://down.webgraphicshub.com'
  
  // Don't redirect if already on down page
  if (url.hostname === 'down.webgraphicshub.com') {
    return fetch(request)
  }
  
  try {
    const response = await fetch(request, {
      // Add timeout to fail faster
      cf: {
        timeout: 10000 // 10 seconds
      }
    })
    
    // Check for various error conditions
    const isError = 
      response.status >= 500 || // Server errors
      response.status === 0 ||   // Network failure
      !response.ok               // Any non-2xx status
    
    if (isError) {
      // Check if it's specifically a Cloudflare Tunnel error
      const text = await response.clone().text()
      if (text.includes('Error 1033') || 
          text.includes('Argo Tunnel') ||
          text.includes('cloudflared')) {
        return Response.redirect(downPageUrl, 302)
      }
    }
    
    return response
  } catch (error) {
    // Network error - redirect to down page
    return Response.redirect(downPageUrl, 302)
  }
}
```

### 4.2 Add Custom Headers
You can add custom headers to track redirects:

```javascript
return Response.redirect(downPageUrl, 302, {
  headers: {
    'X-Redirect-Reason': 'Server Offline',
    'X-Original-Host': url.hostname
  }
})
```

---

## Step 5: Testing

### 5.1 Test When Server is Down
1. Shut down your PC (or stop the Cloudflare Tunnel)
2. Visit your main domain (e.g., `webgraphicshub.com`)
3. You should be redirected to `down.webgraphicshub.com`
4. Verify the countdown timer and archery game work

### 5.2 Test When Server is Up
1. Start your PC and Cloudflare Tunnel
2. Visit your main domain
3. You should see your normal website (no redirect)

---

## Cloudflare Workers Free Tier Limits

The free tier includes:
- ✅ **100,000 requests per day**
- ✅ **10ms CPU time per request**
- ✅ **Unlimited workers** (but only 1 can be active at a time on free tier)
- ✅ **No bandwidth charges**

For most personal websites, this is more than enough!

---

## Troubleshooting

### Issue: Redirect Loop
**Solution**: Make sure you haven't added a route for `down.webgraphicshub.com`. The down page should NOT have the worker applied to it.

### Issue: Worker Not Triggering
**Solution**: 
1. Check that your route is correctly configured
2. Verify the route pattern matches your domain
3. Make sure the worker is deployed (not in draft mode)

### Issue: Still Seeing Error 1033
**Solution**:
1. Check the worker logs in the Cloudflare dashboard
2. Verify the error detection logic is working
3. Try adding more specific error checks

### Issue: Slow Redirects
**Solution**: Reduce the timeout value in the worker code (default is 10 seconds)

---

## Alternative Approach: Using Cloudflare Pages

If you prefer to host your down page directly on Cloudflare:

### 1. Create a Cloudflare Pages Project
1. Go to **Workers & Pages** → **Create application** → **Pages**
2. Connect your GitHub repository or upload files directly
3. Deploy your "Server is Resting" page
4. You'll get a URL like `error-page.pages.dev`

### 2. Add Custom Domain
1. In your Pages project, go to **Custom domains**
2. Add `down.webgraphicshub.com`
3. Cloudflare will automatically configure DNS

### 3. Use the Worker
Follow the same worker setup as above, but use your Pages URL as the `downPageUrl`

---

## Best Practices

1. **Keep the down page simple**: Avoid heavy dependencies that might fail
2. **Test regularly**: Periodically test the redirect when your server is down
3. **Monitor worker usage**: Check your Cloudflare dashboard to ensure you're within free tier limits
4. **Cache the down page**: Use Cloudflare's cache to ensure fast redirects
5. **Add monitoring**: Consider using a service like UptimeRobot to monitor your main site

---

## Summary

You've now set up:
✅ A custom "Server is Resting" page with countdown timer and archery game  
✅ Cloudflare Worker to detect Error 1033 and redirect users  
✅ Automatic handling of server downtime  
✅ All within Cloudflare's free tier!

When your PC is shut down, visitors will be seamlessly redirected to your down page where they can play archery while waiting for the server to come back online at 11 AM IST.
