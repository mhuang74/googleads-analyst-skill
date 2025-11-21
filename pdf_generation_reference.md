# PDF Report Generation Reference

This document provides detailed instructions for generating professionally formatted PDF reports from Google Ads performance analysis.

## PDF Filename Format

```
google_ads_report_{account_name}_{YYYY-MM-DD}.pdf
```

## Styling Requirements

### Fonts (Professional & Modern)

- **Headings**: Inter, Helvetica Neue, or system sans-serif (bold weight)
- **Body text**: Source Sans Pro, Open Sans, or system sans-serif
- **Data tables**: Roboto Mono or system monospace for numerical alignment
- **Font sizes**:
  - H1: 24pt
  - H2: 18pt
  - H3: 14pt
  - Body: 11pt
  - Table content: 10pt
  - Header/Footer: 9pt

### Page Layout

- **Page size**: Letter (8.5" × 11") or A4
- **Margins**: 1 inch (top/bottom), 0.75 inch (left/right)
- **Line spacing**: 1.4 for body text
- **Color scheme**:
  - Primary headings: #1a1a2e (dark navy)
  - Secondary text: #4a4a4a (dark gray)
  - Accent/links: #0066cc (blue)
  - Table headers: #f5f5f5 background with #1a1a2e text
  - Status colors: 🟢 #28a745, 🟡 #ffc107, 🔴 #dc3545

### Header (appears on every page)

```
┌─────────────────────────────────────────────────────────────────┐
│  [Account Name]                                                 │
│  Google Ads Performance Report                                  │
│  Report Period: [Start Date] to [End Date]                      │
└─────────────────────────────────────────────────────────────────┘
```

- Account name: Bold, 14pt
- "Google Ads Performance Report": Regular, 12pt
- Report period: Regular, 10pt, gray (#666666)
- Separator line below header: 1pt, light gray (#e0e0e0)

### Footer (appears on every page)

```
┌─────────────────────────────────────────────────────────────────┐
│  Report generated: [Full Date with Time]              Page X/Y  │
└─────────────────────────────────────────────────────────────────┘
```

- Format: "Report generated: November 21, 2025 at 2:30 PM"
- Page numbers: Right-aligned, "Page 1 of 5" format
- Separator line above footer: 1pt, light gray (#e0e0e0)

### Table Styling

- Alternating row colors: white (#ffffff) and light gray (#f9f9f9)
- Header row: Bold text, light background (#f5f5f5)
- Cell padding: 8px
- Border: 1px solid #e0e0e0
- Numeric columns: Right-aligned
- Text columns: Left-aligned

## PDF Generation Methods

### Option 1: Using pandoc with custom styling (Recommended)

```bash
# Create a PDF with professional styling
pandoc google_ads_report_{account_name}_{YYYY-MM-DD}.md \
  -o google_ads_report_{account_name}_{YYYY-MM-DD}.pdf \
  --pdf-engine=xelatex \
  -V geometry:margin=1in \
  -V mainfont="Source Sans Pro" \
  -V monofont="Roboto Mono" \
  -V fontsize=11pt \
  -V colorlinks=true \
  -V linkcolor=blue \
  --metadata title="Google Ads Performance Report" \
  --metadata author="[Account Name]" \
  --metadata date="Report Period: [Start Date] to [End Date]"
```

### Option 2: Using wkhtmltopdf with HTML intermediate

```bash
# First convert markdown to HTML with styling
pandoc google_ads_report_{account_name}_{YYYY-MM-DD}.md \
  -o /tmp/report.html \
  --standalone \
  --metadata title="Google Ads Performance Report"

# Then convert to PDF with header/footer
wkhtmltopdf \
  --page-size Letter \
  --margin-top 25mm \
  --margin-bottom 20mm \
  --margin-left 20mm \
  --margin-right 20mm \
  --header-spacing 5 \
  --header-font-name "Helvetica Neue" \
  --header-font-size 9 \
  --header-left "[Account Name]" \
  --header-center "Google Ads Performance Report" \
  --header-right "Period: [Start Date] - [End Date]" \
  --header-line \
  --footer-spacing 5 \
  --footer-font-name "Helvetica Neue" \
  --footer-font-size 9 \
  --footer-left "Report generated: [Current Date Time]" \
  --footer-right "Page [page] of [topage]" \
  --footer-line \
  /tmp/report.html \
  google_ads_report_{account_name}_{YYYY-MM-DD}.pdf
```

### Option 3: Using Python with weasyprint

```python
# Example with weasyprint
from weasyprint import HTML, CSS

# Define custom CSS for professional styling
custom_css = CSS(string='''
  @page {
    size: letter;
    margin: 1in 0.75in;
    @top-center {
      content: "[Account Name] | Google Ads Performance Report | [Start Date] to [End Date]";
      font-family: "Helvetica Neue", sans-serif;
      font-size: 9pt;
      color: #666666;
      border-bottom: 1pt solid #e0e0e0;
      padding-bottom: 5mm;
    }
    @bottom-left {
      content: "Report generated: [Current Date Time]";
      font-family: "Helvetica Neue", sans-serif;
      font-size: 9pt;
      color: #666666;
    }
    @bottom-right {
      content: "Page " counter(page) " of " counter(pages);
      font-family: "Helvetica Neue", sans-serif;
      font-size: 9pt;
      color: #666666;
    }
  }
  body {
    font-family: "Source Sans Pro", "Open Sans", sans-serif;
    font-size: 11pt;
    line-height: 1.4;
    color: #1a1a2e;
  }
  h1 { font-size: 24pt; font-weight: bold; }
  h2 { font-size: 18pt; font-weight: bold; }
  h3 { font-size: 14pt; font-weight: bold; }
  table {
    width: 100%;
    border-collapse: collapse;
    font-size: 10pt;
  }
  th {
    background-color: #f5f5f5;
    font-weight: bold;
    text-align: left;
    padding: 8px;
    border: 1px solid #e0e0e0;
  }
  td {
    padding: 8px;
    border: 1px solid #e0e0e0;
  }
  tr:nth-child(even) { background-color: #f9f9f9; }
''')

HTML('report.html').write_pdf('report.pdf', stylesheets=[custom_css])
```

## PDF Generation Checklist

- [ ] **PDF saved with proper filename**: `google_ads_report_{account_name}_{YYYY-MM-DD}.pdf`
- [ ] **Header on every page** includes:
  - [ ] Account name
  - [ ] "Google Ads Performance Report" title
  - [ ] Report date range (Start Date to End Date)
- [ ] **Footer on every page** includes:
  - [ ] Report generation timestamp
  - [ ] Page numbers (Page X of Y)
- [ ] Professional fonts applied (sans-serif for headings/body, monospace for tables)
- [ ] Tables are properly formatted with alternating row colors
- [ ] Status indicators (🟢 🟡 🔴) render correctly
- [ ] Links are clickable (if applicable)
- [ ] Page breaks occur at logical section boundaries
