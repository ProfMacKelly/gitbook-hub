---
hidden: true
---

# sitemap

I can help convert that sitemap (and the pages it lists) into GitBook-optimized Markdown, but I can't fetch external URLs directly from here. If you can either (A) paste the sitemap XML or page HTML, or (B) allow me to fetch it by running a script on your machine and sending me results — I can produce the converted Markdown. Below are two practical options you can choose from.

Option A — You paste content here

* Paste the sitemap XML or the HTML of any pages you want converted.
* I’ll return GitBook-optimized Markdown following your rules (tabs for package managers, steppers for long numbered steps, FAQs as expandables, remove navigation, etc).

Option B — Run the provided script locally to fetch and convert the sitemap and pages

* Run the script below on your machine. It:
  * Downloads the sitemap XML,
  * Fetches each page listed,
  * Converts pages to Markdown,
  * Applies the GitBook enhancements (tabs for npm/yarn/pnpm blocks, stepper conversion for long numbered steps, FAQ -> expandables),
  * Saves per-page Markdown files you can import into GitBook.

Save this Python script as convert\_sitemap\_to\_gitbook.py and run it with Python 3.9+.

Python script (self-contained):

* Requires pip installs: requests, beautifulsoup4, markdownify, lxml
* The script is conservative (doesn’t add content). It tries to implement your rules automatically but outputs logs so you can review and tweak.

Script:

````python
#!/usr/bin/env python3
"""
convert_sitemap_to_gitbook.py

Usage:
    pip install requests beautifulsoup4 markdownify lxml
    python convert_sitemap_to_gitbook.py https://prof-mackelly.gitbook.io/business-law-i/sitemap-pages.xml output_dir

What it does:
- Downloads sitemap XML
- Fetches each URL in the sitemap
- Converts HTML to Markdown (using markdownify)
- Applies GitBook-specific transformations:
  - Removes common navigation blocks by pruning header/footer if found
  - Detects code blocks preceded by a line listing npm / yarn / pnpm and turns into {% tabs %} with one tab per package manager
  - Converts long numbered steps (1., 2., 3.) with large content into a {% stepper %} block
  - Turns "FAQ" sections (heading "FAQ", "Frequently Asked Questions", or headings with Q/A patterns) into <details> expandables
- Writes per-page markdown files to output_dir
"""
import os
import re
import sys
import requests
from bs4 import BeautifulSoup, Tag
from markdownify import markdownify as md

if len(sys.argv) != 3:
    print("Usage: python convert_sitemap_to_gitbook.py <sitemap_url> <output_dir>")
    sys.exit(1)

SITEMAP_URL = sys.argv[1]
OUT_DIR = sys.argv[2]
os.makedirs(OUT_DIR, exist_ok=True)

session = requests.Session()
session.headers.update({"User-Agent": "sitemap-to-gitbook/1.0"})

def fetch(url):
    print("Fetching:", url)
    r = session.get(url, timeout=30)
    r.raise_for_status()
    return r.text

def parse_sitemap(xml_text):
    soup = BeautifulSoup(xml_text, "lxml-xml")
    locs = [loc.get_text().strip() for loc in soup.find_all("loc")]
    return locs

def prune_navigation(soup):
    # Remove common nav/footer elements heuristically
    for sel in ['nav', 'header', 'footer', '.site-header', '.site-footer', '#header', '#footer', '.navigation', '.topbar']:
        for el in soup.select(sel):
            el.decompose()
    return soup

def html_to_markdown(html):
    # convert to markdown using markdownify with some control
    return md(html, heading_style="ATX")

def detect_package_manager_tabs(md_text):
    # find patterns: a small list like "npm, yarn, pnpm" followed by code block (fenced)
    # or lines that mention these names followed by code fences.
    # We'll create a {% tabs %}... {% endtabs %} block replacing the set.
    pm_names = ["npm", "yarn", "pnpm"]
    pattern = re.compile(
        r"(?P<prefix>(?:.*(?:npm|yarn|pnpm).*\n){1,2})\s*(```(?:[\s\S]*?)```)",
        flags=re.IGNORECASE
    )
    def repl(m):
        prefix = m.group("prefix").strip()
        codeblock = m.group(2)
        # find which managers are mentioned in prefix
        found = [pm for pm in pm_names if re.search(r'\b' + re.escape(pm) + r'\b', prefix, re.I)]
        if not found:
            return m.group(0)
        # extract fenced block content and language
        cb_match = re.match(r"```(\w+)?\n([\s\S]*?)\n```", codeblock)
        lang = cb_match.group(1) if cb_match else ""
        content = cb_match.group(2) if cb_match else codeblock
        # split content by lines if it's multiple commands separated by newlines; try to map to each manager by line index
        lines = [l for l in content.strip().splitlines() if l.strip() != ""]
        # If same number of lines as managers, map each. Otherwise duplicate content into each tab.
        tab_blocks = []
        if len(lines) == len(found):
            for pm, line in zip(found, lines):
                tab_blocks.append((pm, f"```{lang}\n{line}\n```"))
        else:
            for pm in found:
                tab_blocks.append((pm, f"```{lang}\n{content}\n```"))
        tabs = ["{% tabs %}"]
        for pm, block in tab_blocks:
            tabs.append(f"{{% tab title=\"{pm}\" %}}\n{block}\n{{% endtab %}}")
        tabs.append("{% endtabs %}")
        return "\n\n".join(tabs)
    new_md = pattern.sub(repl, md_text)
    return new_md

def detect_numbered_steps(md_text):
    # Detect long numbered lists where each item has at least a paragraph or multiple lines.
    # Convert to {% stepper %} with {% step %} sections.
    # Find blocks starting with "1. " followed by "2. " etc.
    list_pattern = re.compile(r"(^|\n)((?:\d+\.\s[^\n]+(?:\n(?:\s{2,}.*|\s*[^0-9]\S.*))+\n?)+)", flags=re.M)
    def to_stepper(m):
        block = m.group(2).strip('\n')
        # split items
        items = re.split(r"\n\d+\.\s+", "\n" + block)
        items = [it.strip() for it in items if it.strip()]
        # only convert if there are multiple items and at least one has multiple lines
        multi = any("\n" in it for it in items)
        if len(items) < 2 or not multi:
            return m.group(0)
        parts = ["{% stepper %}"]
        for it in items:
            # try to use first line as a step title if short
            first_line = it.splitlines()[0]
            title = first_line if len(first_line) < 80 else ""
            if title:
                body = it[len(first_line):].strip()
                parts.append("{% step %}\n## " + title + "\n\n" + body + "\n{% endstep %}")
            else:
                parts.append("{% step %}\n" + it + "\n{% endstep %}")
        parts.append("{% endstepper %}")
        return "\n\n".join(parts)
    new = list_pattern.sub(to_stepper, md_text)
    return new

def detect_faq_expandables(md_text):
    # Convert sections under headings named FAQ or Frequently Asked Questions into details blocks
    # Also convert Q: / A: patterns into <details>
    lines = md_text.splitlines()
    out = []
    i = 0
    while i < len(lines):
        line = lines[i]
        if re.match(r"^#+\s*(FAQ|Frequently Asked Questions)\s*$", line, re.I):
            # take until next same-or-higher heading
            level = line.count('#')
            i += 1
            block = []
            while i < len(lines) and not re.match(rf"^#{{1,{level}}}\s+", lines[i]):
                block.append(lines[i])
                i += 1
            # convert Q/A pairs in block
            faq_text = "\n".join(block).strip()
            # find Q/A pairs like "Q: ...\nA: ..." or headings for questions
            qa_pairs = []
            # method 1: headings as questions
            soup_like = faq_text.splitlines()
            j = 0
            while j < len(soup_like):
                l = soup_like[j]
                h = re.match(r"^#+\s*(.+)", l)
                if h:
                    q = h.group(1).strip()
                    j += 1
                    ans_lines = []
                    while j < len(soup_like) and not re.match(r"^#+\s*", soup_like[j]):
                        ans_lines.append(soup_like[j])
                        j += 1
                    qa_pairs.append((q, "\n".join(ans_lines).strip()))
                else:
                    # method 2: Q: / A:
                    mQ = re.match(r"^\s*(?:Q:|Question:)\s*(.+)", l, re.I)
                    if mQ:
                        q = mQ.group(1).strip()
                        j += 1
                        ans_lines = []
                        while j < len(soup_like) and not re.match(r"^\s*(?:Q:|Question:)", soup_like[j], re.I):
                            ma = re.match(r"^\s*(?:A:|Answer:)\s*(.+)", soup_like[j], re.I)
                            if ma:
                                ans_lines.append(ma.group(1))
                            else:
                                ans_lines.append(soup_like[j])
                            j += 1
                        qa_pairs.append((q, "\n".join(ans_lines).strip()))
                    else:
                        j += 1
            if qa_pairs:
                out.append(line)  # keep the FAQ heading
                for q,a in qa_pairs:
                    out.append(f"<details>\n<summary>{q}</summary>\n\n{a}\n\n</details>")
            else:
                out.append(line)
                out.extend(block)
        else:
            out.append(line)
            i += 1
    return "\n".join(out)

def sanitize_filename(url):
    # create a safe filename from URL
    from urllib.parse import urlparse, unquote
    p = urlparse(url)
    path = unquote(p.path).strip("/")
    if not path:
        path = "index"
    fname = path.replace("/", "_")
    # include netloc for uniqueness if needed
    fname = f"{p.netloc}__{fname}.md"
    fname = re.sub(r"[^A-Za-z0-9._-]", "_", fname)
    return fname

def process_page(url):
    html = fetch(url)
    soup = BeautifulSoup(html, "lxml")
    soup = prune_navigation(soup)

    # Optionally try to extract main article/content
    main = None
    for sel in ['main', 'article', '#content', '.content', '.main']:
        el = soup.select_one(sel)
        if el:
            main = el
            break
    if main is None:
        # fallback to body
        main = soup.body or soup

    # remove scripts and styles
    for tag in main.select('script, style, noscript'):
        tag.decompose()

    html_fragment = str(main)
    md_text = html_to_markdown(html_fragment)

    # Apply GitBook transformations
    md_text = detect_package_manager_tabs(md_text)
    md_text = detect_numbered_steps(md_text)
    md_text = detect_faq_expandables(md_text)

    return md_text

def main():
    sitemap_xml = fetch(SITEMAP_URL)
    urls = parse_sitemap(sitemap_xml)
    print(f"Found {len(urls)} URLs in sitemap")
    for url in urls:
        try:
            md_text = process_page(url)
            fname = sanitize_filename(url)
            out_path = os.path.join(OUT_DIR, fname)
            with open(out_path, "w", encoding="utf-8") as f:
                f.write(md_text)
            print("Wrote:", out_path)
        except Exception as e:
            print("Error processing", url, e)

if __name__ == "__main__":
    main()
````

Notes and limitations

* The script is heuristic-based. It won't be perfect for every page:
  * It tries to prune header/footer by common class/element names but may miss or remove content incorrectly in some sites — always review outputs.
  * The stepper detection looks for long numbered lists with multiline items; short inline enumerations are left intact.
  * Package manager tabs detection searches for small lists mentioning npm/yarn/pnpm followed by a code fence; it may need adjustment for different formatting.
  *   FAQ detection looks for an "FAQ" heading or Q:/A: patterns and converts to&#x20;

      blocks.
* After running, review each Markdown file for correctness and small edits before importing into GitBook.
* If you want, I can adjust the script to be stricter/looser for your site after you try it once and show me sample output.

If you prefer, paste the sitemap XML or a few example page HTMLs here and I will convert them into ready-to-import GitBook Markdown following the rules. Which option do you want to proceed with?
