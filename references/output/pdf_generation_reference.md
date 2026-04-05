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

## HTML Template for PDF Generation

**Primary Approach:** Generate HTML with inline CSS, then convert to PDF using pandoc + weasyprint.

### Complete HTML Template

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Google Ads Performance Report</title>
  <style>
    @page {
      size: letter;
      margin: 1in 0.75in;

      @top-center {
        content: "[Account Name] | Google Ads Performance Report | [Start Date] to [End Date]";
        font-family: "Helvetica Neue", "Arial", sans-serif;
        font-size: 9pt;
        color: #666666;
        border-bottom: 1pt solid #e0e0e0;
        padding-bottom: 5mm;
      }

      @bottom-left {
        content: "Report generated: [Current Date Time]";
        font-family: "Helvetica Neue", "Arial", sans-serif;
        font-size: 9pt;
        color: #666666;
      }

      @bottom-right {
        content: "Page " counter(page) " of " counter(pages);
        font-family: "Helvetica Neue", "Arial", sans-serif;
        font-size: 9pt;
        color: #666666;
      }
    }

    body {
      font-family: "Source Sans Pro", "Open Sans", "Arial", sans-serif;
      font-size: 11pt;
      line-height: 1.6;
      color: #1a1a2e;
    }

    h1 {
      font-size: 24pt;
      font-weight: bold;
      color: #1a1a2e;
      margin-top: 0;
      margin-bottom: 16pt;
    }

    h2 {
      font-size: 18pt;
      font-weight: bold;
      color: #1a1a2e;
      margin-top: 20pt;
      margin-bottom: 12pt;
      page-break-after: avoid;
    }

    h3 {
      font-size: 14pt;
      font-weight: bold;
      color: #1a1a2e;
      margin-top: 16pt;
      margin-bottom: 10pt;
      page-break-after: avoid;
    }

    p {
      margin-bottom: 12pt;
      text-align: left;
    }

    ul, ol {
      margin-bottom: 12pt;
      padding-left: 20pt;
    }

    li {
      margin-bottom: 6pt;
      text-align: left;
    }

    table {
      width: 100%;
      border-collapse: collapse;
      font-size: 10pt;
      margin-bottom: 16pt;
      page-break-inside: auto;
    }

    thead {
      display: table-header-group;
    }

    tr {
      page-break-inside: avoid;
      page-break-after: auto;
    }

    th {
      background-color: #f5f5f5;
      font-weight: bold;
      text-align: left;
      padding: 8px;
      border: 1px solid #e0e0e0;
    }

    th.number {
      text-align: right;
    }

    td {
      padding: 8px;
      border: 1px solid #e0e0e0;
      text-align: left;
    }

    td.number {
      text-align: right;
      font-family: "Roboto Mono", "Courier New", monospace;
    }

    tr:nth-child(even) {
      background-color: #f9f9f9;
    }

    .status-good { color: #28a745; font-weight: bold; }
    .status-warning { color: #ffc107; font-weight: bold; }
    .status-critical { color: #dc3545; font-weight: bold; }

    .positive-change { color: #28a745; }
    .negative-change { color: #dc3545; }

    .section {
      margin-bottom: 24pt;
    }

    .priority-high {
      background-color: #fee;
      padding: 12pt;
      margin-bottom: 12pt;
      border-left: 4pt solid #dc3545;
    }

    .priority-medium {
      background-color: #fffbf0;
      padding: 12pt;
      margin-bottom: 12pt;
      border-left: 4pt solid #ffc107;
    }

    .priority-low {
      background-color: #f0f8ff;
      padding: 12pt;
      margin-bottom: 12pt;
      border-left: 4pt solid #0066cc;
    }
  </style>
</head>
<body>
  <h1>Google Ads Performance Report</h1>

  <div class="section">
    <p><strong>Account:</strong> [Account Name] ([Account ID])</p>
    <p><strong>Report Period:</strong> [Start Date] to [End Date]</p>
    <p><strong>Comparison Period:</strong> [Prior Start Date] to [Prior End Date]</p>
    <p><strong>Generated:</strong> [Current Date and Time]</p>
  </div>

  <div class="section">
    <h2>Executive Summary</h2>
    <p>[2-3 sentence executive summary with key findings]</p>
  </div>

  <div class="section">
    <h2>Key Findings</h2>
    <ul>
      <li><span class="status-good">🟢</span> [Positive finding]</li>
      <li><span class="status-warning">🟡</span> [Warning finding]</li>
      <li><span class="status-critical">🔴</span> [Critical finding]</li>
    </ul>
  </div>

  <div class="section">
    <h2>Detailed Campaign Analysis</h2>
    <table>
      <thead>
        <tr>
          <th>Campaign</th>
          <th class="number">Impressions</th>
          <th class="number">Clicks</th>
          <th class="number">CTR</th>
          <th class="number">Cost</th>
          <th class="number">Conv.</th>
          <th class="number">CPA</th>
          <th>Status</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>[Campaign Name]</td>
          <td class="number">123,456</td>
          <td class="number">1,234</td>
          <td class="number">1.00%</td>
          <td class="number">$1,234.56</td>
          <td class="number">45</td>
          <td class="number">$27.43</td>
          <td><span class="status-good">🟢 Good</span></td>
        </tr>
      </tbody>
    </table>
  </div>

  <div class="section">
    <h2>🚨 High Priority Issues</h2>
    <div class="priority-high">
      <h3>[Campaign Name]</h3>
      <p><strong>Issue:</strong> [Description with supporting data]</p>
      <p><strong>Recommended Actions:</strong></p>
      <ul>
        <li>[Action 1]</li>
        <li>[Action 2]</li>
      </ul>
    </div>
  </div>

  <div class="section">
    <h2>⚠️ Medium Priority Issues</h2>
    <div class="priority-medium">
      <h3>[Campaign Name]</h3>
      <p>[Issue description and recommendations]</p>
    </div>
  </div>

  <div class="section">
    <h2>📊 Account-Level Summary</h2>
    <table>
      <thead>
        <tr>
          <th>Metric</th>
          <th class="number">Current Period</th>
          <th class="number">Previous Period</th>
          <th class="number">Change</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>Total Cost</td>
          <td class="number">$12,345.67</td>
          <td class="number">$10,234.56</td>
          <td class="number"><span class="positive-change">+20.6%</span></td>
        </tr>
      </tbody>
    </table>
  </div>

  <div class="section">
    <h2>🎯 Next Steps</h2>
    <h3>Immediate Actions (Today/Tomorrow)</h3>
    <ul>
      <li>[High priority action]</li>
    </ul>

    <h3>This Week</h3>
    <ul>
      <li>[Medium priority action]</li>
    </ul>

    <h3>Ongoing Monitoring</h3>
    <ul>
      <li>[Low priority item]</li>
    </ul>
  </div>
</body>
</html>
```

## PDF Generation Methods

### Option 1: Using pandoc with weasyprint (Recommended)

**Step 1: Generate HTML file with the template above**

Save your report content using the HTML template provided above to `/tmp/report.html`.

**Step 2: Convert to PDF**

```bash
# Convert HTML to PDF using pandoc with weasyprint engine
pandoc /tmp/report.html \
  -o google_ads_report_{account_name}_{YYYY-MM-DD}.pdf \
  --pdf-engine=weasyprint \
  --metadata title="Google Ads Performance Report" \
  --metadata author="[Account Name]"
```

**Why weasyprint?**
- Better CSS3 support for advanced styling
- Excellent rendering of alternating table row shading
- Reliable page headers/footers through CSS `@page` rules
- Superior font rendering and typography control
- Proper handling of text alignment (left for text, right for numbers)
- Support for page-break-inside controls

**Simplified One-Liner (if HTML is in a string variable):**

```bash
# Generate HTML and convert to PDF in one step
cat > /tmp/report.html << 'EOF'
[HTML content here]
EOF

pandoc /tmp/report.html \
  -o google_ads_report_{account_name}_{YYYY-MM-DD}.pdf \
  --pdf-engine=weasyprint
```

**Alternative: Using pandoc with xelatex (Not Recommended)**

```bash
# Only use if weasyprint is not available
# Note: LaTeX engines have limited support for HTML/CSS styling
pandoc /tmp/report.html \
  -o google_ads_report_{account_name}_{YYYY-MM-DD}.pdf \
  --pdf-engine=xelatex \
  -V geometry:margin=1in \
  -V fontsize=11pt
```

**⚠️ Note:** xelatex has limited CSS support and may not render table shading, proper alignment, or page headers/footers correctly. Always prefer weasyprint.

### Option 2: Using wkhtmltopdf (Alternative)

```bash
# Convert HTML directly to PDF with header/footer using wkhtmltopdf
# Note: wkhtmltopdf uses command-line headers/footers instead of CSS @page rules
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

**⚠️ Note:** wkhtmltopdf requires headers/footers to be specified via command-line flags (shown above) rather than CSS `@page` rules. You'll need to remove the `@page` section from the CSS if using this method.

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

**HTML Generation:**
- [ ] HTML file created using the template above
- [ ] All placeholder values replaced with actual data
- [ ] Tables use `<th class="number">` for numeric column headers
- [ ] Tables use `<td class="number">` for numeric data cells
- [ ] Currency values formatted with $ and commas (e.g., $1,234.56)
- [ ] Percentage values formatted with % symbol (e.g., 12.5%)
- [ ] Status indicators use correct CSS classes (.status-good, .status-warning, .status-critical)
- [ ] Priority sections use correct CSS classes (.priority-high, .priority-medium, .priority-low)
- [ ] Line breaks added between major sections for readability
- [ ] Bullet points used for action items and recommendations

**PDF Conversion:**
- [ ] **PDF saved with proper filename**: `google_ads_report_{account_name}_{YYYY-MM-DD}.pdf`
- [ ] Used pandoc with weasyprint engine (not xelatex or wkhtmltopdf)
- [ ] **Header on every page** includes:
  - [ ] Account name
  - [ ] "Google Ads Performance Report" title
  - [ ] Report date range (Start Date to End Date)
- [ ] **Footer on every page** includes:
  - [ ] Report generation timestamp
  - [ ] Page numbers (Page X of Y)
- [ ] Professional fonts render correctly (sans-serif for headings/body)
- [ ] **Tables properly formatted**:
  - [ ] Alternating row colors (white/#f9f9f9) visible
  - [ ] Column headers have gray background (#f5f5f5)
  - [ ] Text columns left-aligned
  - [ ] Numeric columns right-aligned
  - [ ] Adequate cell padding (8px minimum)
  - [ ] Borders visible and clean
- [ ] Status indicators (🟢 🟡 🔴) render correctly with colors
- [ ] Page breaks occur at logical section boundaries (not mid-table)
- [ ] PDF is client-ready and professional looking
