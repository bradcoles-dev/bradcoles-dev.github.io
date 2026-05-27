# Brad Coles - Personal Portfolio Website

Senior Data Engineering Consultant portfolio website showcasing experience, projects, certifications, technical expertise, and blog articles.

## Overview

A portfolio website and technical blog built with vanilla HTML, CSS, and JavaScript. It features a modern, tech-focused design with animated backgrounds, responsive layouts, and smooth scrolling navigation.

**Live Site:** [https://bradcoles-dev.github.io](https://bradcoles-dev.github.io)

## Project Structure

```
bradcoles-dev.github.io/
├── index.html              # Main portfolio page
├── blog/                   # Technical blog articles
│   ├── fabric-mirroring.html
│   ├── fabric-delta-table-maintenance.html
│   ├── fabric-private-links.html
│   └── ai-augmented-engineering.html
├── images/                 # Image assets
│   └── Head_shot-removebg-preview.png  # Profile headshot image
├── _config.yml             # Jekyll configuration (GitHub Pages)
└── README.md               # This file
```

## Technology Stack

- **HTML5** - Semantic markup with accessibility features
- **CSS3** - Custom styling with CSS Grid, Flexbox, animations
- **JavaScript** - Vanilla JS for navigation, scroll effects, form handling
- **GitHub Pages** - Hosting platform
- **Jekyll** - Static site generator (minimal theme)

## Key Features

- **Responsive Design** - Mobile-first approach with breakpoints at 768px
- **Animated Background** - Moving grid pattern using CSS animations
- **Sticky Navigation** - Shows/hides on scroll with smooth transitions
- **Portfolio Sections:**
  - Hero with profile image and social links
  - Technical Stack grid
  - Experience timeline
  - Featured Projects
  - Certifications (18 professional certifications)
  - Education & Awards
  - About Me
  - Contact form (Netlify integration)
- **Technical Blog** - In-depth articles on data engineering and Microsoft Fabric
- **SEO Optimized** - Meta tags, Open Graph, Twitter Cards, structured data
- **Accessibility** - ARIA labels, semantic HTML, keyboard navigation

## Development

### Local Development

Simply open `index.html` in a web browser. No build process required.

### Making Changes

The portfolio is a single HTML file (`index.html`) with embedded CSS and JavaScript. Blog articles are individual HTML files in the `blog/` directory, each self-contained with embedded styles consistent with the main site theme.

### Styling

All CSS is embedded in each file's `<style>` tag. Key CSS variables are defined in `:root` for easy theming:

```css
--bg-dark: #0a0e1a;
--bg-card: #111827;
--accent: #3b82f6;
--success: #10b981;
```

### Adding a Blog Article

Create a new HTML file in `blog/` using an existing article as a template. Ensure the file includes:
- Consistent `<head>` metadata (Open Graph, Twitter Cards, GA tag)
- An accuracy notice callout if the content may date quickly
- A back link to `../index.html`
- The standard footer

### Deployment

Automatically deployed via GitHub Pages when changes are pushed to the `main` branch.

## Contact Form

The contact form uses Netlify Forms for submission handling. If not deployed on Netlify, submissions will trigger a browser alert instead.

## Browser Support

- Chrome/Edge (latest)
- Firefox (latest)
- Safari (latest)
- Mobile browsers (iOS Safari, Chrome Mobile)

## Future Enhancements

Potential improvements to consider:
- Add dark/light mode toggle
- Add project case studies with detailed pages
- Integrate analytics
- Add downloadable resume/CV
- Create separate CSS/JS files for better organisation

## License

Personal portfolio website. All rights reserved.

## Contact

**Brad Coles**
- Email: bradcoles@outlook.com.au
- LinkedIn: [linkedin.com/in/brad-coles](https://linkedin.com/in/brad-coles)
- GitHub: [github.com/bradcoles-dev](https://github.com/bradcoles-dev)
- Location: Adelaide, South Australia
