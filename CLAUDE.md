# Claude Code Notes

This file provides important context for AI assistants working on this portfolio website.

## Project Overview

Single-page portfolio website for Brad Coles, a Senior Data Engineering Consultant based in Adelaide, South Australia. The site showcases professional experience, technical skills, projects, certifications, and contact information.

**Language:** Australian English (en-AU) - Use Australian spelling conventions throughout (e.g., "optimisation" not "optimization", "utilisation" not "utilization", "colour" not "color" in text content).

## Architecture Decisions

### Single-Page Design
The entire website is contained in a single `index.html` file with embedded CSS and JavaScript. This was chosen for:
- Simplicity - no build process required
- Fast loading - everything loads at once
- Easy maintenance - one file to manage
- GitHub Pages compatibility - works out of the box

### Styling Approach
All CSS is embedded in the `<style>` tag rather than external stylesheet because:
- Reduces HTTP requests
- Keeps everything self-contained
- Simplifies deployment
- Single file makes it easier to share/backup

### No Framework
Built with vanilla HTML, CSS, and JavaScript (no React, Vue, etc.) because:
- Lightweight - faster load times
- No dependencies to manage
- Simpler for a portfolio site
- Better for SEO

## Key Implementation Details

### Responsive Design
- Mobile-first approach
- Breakpoint at 768px for tablet/desktop
- Flexbox and CSS Grid for layouts
- Images scale appropriately

### Navigation
- Sticky navigation appears on scroll (after 100px)
- Smooth scroll behavior to anchor sections
- Active section highlighting
- Mobile hamburger menu

### Animations
- CSS animations for background grid movement
- Fade-in animations on scroll for sections
- Hover effects on cards and buttons
- Scanline effect on header border

### Sections Structure
1. **Hero** (header) - Name, title, profile image, social links
2. **Skills** - 4-column grid of skill categories
3. **Experience** - Timeline layout with 3 positions
4. **Projects** - Featured project cards (3 major projects)
5. **Certifications** - Grid of 18 professional certifications
6. **Education** - Academic qualifications and awards
7. **About** - Personal bio and background
8. **Contact** - Contact form with Netlify integration

### Color Scheme
Tech-focused dark theme with blue accents:
- Background: Dark navy (#0a0e1a)
- Cards: Slightly lighter (#111827)
- Accent: Blue (#3b82f6)
- Success: Green (#10b981)
- Text: Light gray (#e5e7eb)

## Important Constraints

### DO NOT:
- Split into multiple HTML files (maintain single-page pattern)
- Remove or change Jekyll configuration (needed for GitHub Pages)
- Break responsive design (always test mobile/tablet/desktop)
- Remove accessibility features (ARIA labels, semantic HTML)
- Change the color scheme dramatically (stay consistent with current theme)
- Remove structured data (JSON-LD schema)

### ALWAYS:
- Use Australian English spelling (optimisation, utilisation, colour, etc.)
- Update README.md when adding features
- Maintain line number references in README when editing
- Test on multiple screen sizes
- Keep CSS variables for theming consistency
- Update meta tags and Open Graph data when changing content
- Preserve the professional, tech-focused aesthetic
- Include credential IDs for certifications when available

## Common Update Patterns

### Adding a Certification
Add a new `cert-card` div in the certifications section:
```html
<div class="cert-card">
    <h3>Certification Name</h3>
    <div class="issuer">Issuing Organization</div>
    <div class="date">Issued Month Year | Credential ID: XXXXX</div>
</div>
```

### Adding a Project
Add a new `project-card` div in the projects section:
```html
<div class="project-card">
    <h3>Project Name</h3>
    <p>Project description...</p>
    <div class="project-tech">
        <span class="tech-badge">Technology 1</span>
        <span class="tech-badge">Technology 2</span>
    </div>
</div>
```

### Adding Experience
Add a new `timeline-item` div in the experience section:
```html
<div class="timeline-item">
    <h3>Job Title</h3>
    <div class="role">Company Name</div>
    <div class="period">Start Date - End Date | Location</div>
    <p>Job description and achievements...</p>
</div>
```

## SEO and Metadata

The site includes:
- Meta description and keywords
- Open Graph tags for social sharing
- Twitter Card metadata
- Structured data (JSON-LD) with Person schema
- Semantic HTML5 elements
- Descriptive page title

When updating content, remember to update:
1. Page title and meta description
2. Open Graph title/description
3. Structured data (JSON-LD at bottom of file)
4. Keywords if adding new technical skills

## Contact Form

Uses Netlify Forms (`data-netlify="true"` attribute). If not deployed on Netlify, falls back to JavaScript alert on submission. The form fields are:
- Name (required)
- Email (required)
- Message (required)
- Hidden honeypot field for spam protection

## Performance Considerations

- All assets are minimal (single HTML, single image)
- Fonts loaded from Google Fonts with preconnect
- No JavaScript frameworks or libraries
- CSS is optimized and embedded
- Images should be optimized before uploading

## Future Enhancement Ideas

Documented in README.md but not yet implemented:
- Dark/light mode toggle
- Blog section
- Detailed project case studies
- Analytics integration
- Downloadable resume/CV
- Separate CSS/JS files for better code organization

## Testing Checklist

When making changes:
- [ ] Test on Chrome/Edge desktop
- [ ] Test on Firefox desktop
- [ ] Test on Safari desktop
- [ ] Test on mobile (iOS Safari)
- [ ] Test on mobile (Android Chrome)
- [ ] Verify navigation links work
- [ ] Test contact form submission
- [ ] Check responsive breakpoints
- [ ] Validate HTML (W3C validator)
- [ ] Check accessibility (WAVE tool)
- [ ] Test Open Graph preview (LinkedIn, Facebook)

## Brand Voice and Tone

Brad's professional brand is:
- Technical and competent
- Results-driven
- Clear and concise
- Professional but approachable
- Focused on business impact
- Data-driven decision making

When updating copy, maintain:
- Quantifiable achievements (percentages, dollar amounts)
- Technical accuracy
- Business outcome focus
- Professional language
- Active voice
