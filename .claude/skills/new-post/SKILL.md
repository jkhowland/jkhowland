---
name: new-post
description: Create a new blog post page with proper markup and styling
user-invocable: true
argument-hint: [title]
---

# Create a New Blog Post

Create a new blog post titled "$ARGUMENTS".

## Steps

1. Generate a slug from the title (lowercase, hyphens, no special characters)

2. Create `posts/{slug}.html` using the same page structure as the other pages:
   - Same `<head>` with title set to "$ARGUMENTS - Joshua Howland"
   - Same `#topbar`, `#header` with logo, menu toggle, and nav
   - Same `#footer` with social links and copyright
   - Content area with the post title, date, and article body

3. The article markup should be:
   ```html
   <div class="content">
     <article class="article">
       <h2>$ARGUMENTS</h2>
       <div class="article-meta">DATE in "Mon DD, YYYY" format</div>
       <div class="article-body">
         <p>Post content goes here.</p>
       </div>
     </article>
   </div>
   ```

4. Add an entry to `index.html` inside `.content`, replacing or alongside the empty state:
   ```html
   <div class="article">
     <h2><a href="/posts/{slug}.html">$ARGUMENTS</a></h2>
     <div class="article-meta">DATE</div>
   </div>
   ```

5. Confirm the file path and remind the user to add their content.
