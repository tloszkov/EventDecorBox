# Event Decor Box - Project Specification

## Project Overview
A modern, responsive website for an event decoration business with an admin interface for content management.

**Domains:**
- Frontend: `eventdecorbox.ro` (future) / `eventdecor.ospc.ro` (current development)
- Backend: `api.eventdecorbox.ro` (future) / `api.ospc.ro` (current development)
- Server: Currently hosted on `ospc.ro` server

---

## Tech Stack

### Frontend
- **Framework:** Astro (static site generation)
- **Styling:** Tailwind CSS
- **Language:** JavaScript/TypeScript
- **Deployment:** Cloudflare Pages (auto-deploy from GitHub main branch)
- **Data Source:** PocketBase API

### Backend
- **CMS/Database:** PocketBase
- **Hosting:** Self-hosted on ospc.ro server
- **Connection:** Cloudflare Tunnel (no CORS issues, no port forwarding needed)
- **Admin Access:** PocketBase built-in admin UI

---

## Architecture

```
┌─────────────────────────────────────┐
│  GitHub Repository (main branch)    │
│  - Push triggers auto-deploy        │
└──────────────┬──────────────────────┘
               │
               ↓
┌─────────────────────────────────────┐
│  Cloudflare Pages                   │
│  - eventdecorbox.ro (future)        │
│  - eventdecor.ospc.ro (current)     │
│  - Static Astro site                │
└──────────────┬──────────────────────┘
               │ API calls
               ↓
┌─────────────────────────────────────┐
│  Cloudflare Tunnel                  │
│  - api.eventdecorbox.ro → server    │
└──────────────┬──────────────────────┘
               │
               ↓
┌─────────────────────────────────────┐
│  PocketBase (ospc.ro server)        │
│  - Port: 8090 (internal only)       │
│  - Admin UI: /admin/                │
│  - REST API + File Storage          │
└─────────────────────────────────────┘
```

---

## Frontend Requirements

### Pages Structure
1. **Homepage** (`/`)
   - Hero section with main image
   - Brief introduction
   - Featured services/decorations
   - Call-to-action buttons

2. **Gallery** (`/gallery`)
   - Responsive image grid (masonry or grid layout)
   - Images loaded from PocketBase
   - Lightbox/modal for full-size viewing
   - Filter by category (optional)

3. **Services** (`/services`)
   - List of decoration services
   - Descriptions and pricing (if applicable)
   - Images for each service

4. **About** (`/about`)
   - Company story
   - Team information
   - Mission/values

5. **Contact** (`/contact`)
   - Contact form
   - Business details (email, phone, address)
   - Social media links
   - Optional: embedded map

### Design Requirements
- **Responsive:** Mobile-first design
- **Clean & Modern:** Minimalist aesthetic
- **Color Scheme:** Elegant, event-appropriate colors (soft pastels, gold accents suggested)
- **Typography:** Professional, readable fonts
- **Navigation:** Sticky header with mobile hamburger menu
- **Footer:** Contact info, social links, copyright

### Components Needed
- Header/Navigation (responsive)
- Footer
- Image Gallery Component
- Service Card Component
- Contact Form Component
- Hero Section
- Loading states
- Error handling UI

---

## Backend Setup (PocketBase)

### Collections Schema

#### 1. **gallery**
```javascript
{
  title: "text",          // Image title
  description: "text",    // Optional description
  image: "file",          // Image file (single)
  category: "text",       // e.g., "wedding", "birthday", "corporate"
  featured: "bool",       // Display on homepage?
  order: "number",        // Display order
  created: "date",
  updated: "date"
}
```

#### 2. **services**
```javascript
{
  name: "text",           // Service name
  description: "editor",  // Rich text description
  image: "file",          // Service image
  price: "text",          // Optional pricing info
  features: "json",       // Array of features/bullets
  order: "number",
  active: "bool",         // Is service currently offered?
  created: "date",
  updated: "date"
}
```

#### 3. **pages**
```javascript
{
  slug: "text",           // e.g., "about", "homepage"
  title: "text",
  content: "editor",      // Rich text content
  images: "file",         // Multiple images
  meta_description: "text", // SEO
  published: "bool",
  created: "date",
  updated: "date"
}
```

#### 4. **contact_submissions** (optional)
```javascript
{
  name: "text",
  email: "email",
  phone: "text",
  message: "text",
  status: "select",       // "new", "read", "replied"
  created: "date"
}
```

### PocketBase Configuration
- Enable CORS for Cloudflare Pages domain
- Set up file storage rules
- Configure admin user
- API rules: public read, admin write

---

## Implementation Steps

### Phase 1: Backend Setup
1. **Install PocketBase on ospc.ro server**
   ```bash
   wget https://github.com/pocketbase/pocketbase/releases/download/v0.22.0/pocketbase_0.22.0_linux_amd64.zip
   unzip pocketbase_0.22.0_linux_amd64.zip
   chmod +x pocketbase
   ```

2. **Create systemd service for PocketBase**
   ```bash
   sudo nano /etc/systemd/system/pocketbase.service
   ```
   ```ini
   [Unit]
   Description=PocketBase
   After=network.target

   [Service]
   Type=simple
   User=your-user
   WorkingDirectory=/path/to/pocketbase
   ExecStart=/path/to/pocketbase/pocketbase serve --http=127.0.0.1:8090
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```

3. **Set up Cloudflare Tunnel**
   ```bash
   # Install cloudflared
   wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64
   sudo mv cloudflared-linux-amd64 /usr/local/bin/cloudflared
   sudo chmod +x /usr/local/bin/cloudflared
   
   # Authenticate
   cloudflared tunnel login
   
   # Create tunnel
   cloudflared tunnel create eventdecorbox-api
   
   # Configure DNS
   cloudflared tunnel route dns eventdecorbox-api api.ospc.ro
   ```

4. **Create Cloudflare Tunnel config**
   ```yaml
   # ~/.cloudflared/config.yml
   tunnel: <tunnel-id>
   credentials-file: /root/.cloudflared/<tunnel-id>.json
   
   ingress:
     - hostname: api.ospc.ro
       service: http://127.0.0.1:8090
     - service: http_status:404
   ```

5. **Run tunnel as service**
   ```bash
   sudo cloudflared service install
   sudo systemctl start cloudflared
   sudo systemctl enable cloudflared
   ```

6. **Access PocketBase admin**
   - Visit `https://api.ospc.ro/_/`
   - Create admin account
   - Create collections as per schema above

---

### Phase 2: Frontend Development

1. **Initialize Astro Project**
   ```bash
   npm create astro@latest eventdecorbox
   cd eventdecorbox
   npx astro add tailwind
   npm install pocketbase
   ```

2. **Project Structure**
   ```
   src/
   ├── components/
   │   ├── Header.astro
   │   ├── Footer.astro
   │   ├── Hero.astro
   │   ├── GalleryGrid.astro
   │   ├── ServiceCard.astro
   │   └── ContactForm.astro
   ├── layouts/
   │   └── Layout.astro
   ├── pages/
   │   ├── index.astro
   │   ├── gallery.astro
   │   ├── services.astro
   │   ├── about.astro
   │   └── contact.astro
   ├── lib/
   │   └── pocketbase.js
   └── styles/
       └── global.css
   ```

3. **Environment Variables**
   ```bash
   # .env
   PUBLIC_POCKETBASE_URL=https://api.ospc.ro
   ```

4. **PocketBase Client Setup**
   ```javascript
   // src/lib/pocketbase.js
   import PocketBase from 'pocketbase';
   
   export const pb = new PocketBase(import.meta.env.PUBLIC_POCKETBASE_URL);
   
   // Disable auto-cancellation for SSG
   pb.autoCancellation(false);
   
   export async function getGalleryImages() {
     const records = await pb.collection('gallery').getFullList({
       sort: '-created',
       filter: 'featured = true'
     });
     return records;
   }
   
   export function getImageUrl(record, filename, thumb = false) {
     if (thumb) {
       return pb.files.getUrl(record, filename, { thumb: '300x300' });
     }
     return pb.files.getUrl(record, filename);
   }
   ```

5. **Build Components**
   - Responsive header with mobile menu
   - Footer with contact info
   - Gallery with lazy loading
   - Service cards with hover effects
   - Contact form (POST to PocketBase)

6. **Implement Pages**
   - Fetch data from PocketBase in each page's frontmatter
   - Render components with data
   - Add meta tags for SEO
   - Implement loading and error states

---

### Phase 3: Deployment

1. **GitHub Repository Setup**
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   git branch -M main
   git remote add origin https://github.com/yourusername/eventdecorbox.git
   git push -u origin main
   ```

2. **Cloudflare Pages Setup**
   - Connect GitHub repository
   - Build settings:
     - Framework preset: `Astro`
     - Build command: `npm run build`
     - Build output directory: `dist`
   - Environment variables: `PUBLIC_POCKETBASE_URL`
   - Custom domain: `eventdecor.ospc.ro` (current) → `eventdecorbox.ro` (future)

3. **Auto-Deploy Configuration**
   - Push to `main` branch triggers automatic rebuild
   - Preview deployments for other branches (optional)

---

### Phase 4: Content Population

1. **Admin Access**
   - Login to `https://api.ospc.ro/_/`
   - Upload initial images to gallery
   - Create service entries
   - Add page content

2. **Testing**
   - Test all pages load correctly
   - Verify images display properly
   - Check responsive design on mobile/tablet
   - Test contact form submission
   - Verify SEO meta tags

---

## Migration Plan (ospc.ro → eventdecorbox.ro)

When `eventdecorbox.ro` access is available:

1. **Update Cloudflare Tunnel DNS**
   ```bash
   cloudflared tunnel route dns eventdecorbox-api api.eventdecorbox.ro
   ```

2. **Update Environment Variables**
   ```bash
   # Update in Cloudflare Pages settings
   PUBLIC_POCKETBASE_URL=https://api.eventdecorbox.ro
   ```

3. **Update Custom Domain**
   - In Cloudflare Pages, change custom domain
   - `eventdecor.ospc.ro` → `eventdecorbox.ro`

4. **Update PocketBase CORS** (if needed)
   - Add new domain to allowed origins

5. **Test Everything**
   - Verify all API calls work
   - Check SSL certificates
   - Test from multiple devices

---

## Security Considerations

- PocketBase admin UI: use strong password, consider IP whitelist
- API rules: ensure collections have proper read/write permissions
- File uploads: limit file types and sizes
- Contact form: implement rate limiting or captcha
- HTTPS: enforced via Cloudflare
- Cloudflare Tunnel: no exposed ports on server

---

## Performance Optimization

- Use PocketBase thumbnail API for gallery images
- Implement lazy loading for images
- Optimize image sizes before upload
- Use Astro's built-in optimizations
- Enable Cloudflare caching
- Minimize JavaScript bundle size

---

## Future Enhancements

- [ ] Multi-language support (Romanian/English)
- [ ] Blog/News section
- [ ] Customer testimonials
- [ ] Booking/inquiry system
- [ ] Social media feed integration
- [ ] Analytics integration (Cloudflare Analytics)
- [ ] Newsletter signup
- [ ] Portfolio/case studies section

---

## Support & Documentation

- **Astro Docs:** https://docs.astro.build
- **Tailwind CSS:** https://tailwindcss.com/docs
- **PocketBase Docs:** https://pocketbase.io/docs
- **Cloudflare Tunnel:** https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/
- **Cloudflare Pages:** https://developers.cloudflare.com/pages/

---

## Notes

- Admin has sole access to PocketBase admin UI
- Frontend is fully static (SSG), rebuilds on content changes via manual trigger or scheduled builds
- No authentication needed on frontend (public website)
- All images and content managed through PocketBase admin interface
- Design should be elegant and professional, suitable for event decoration business
